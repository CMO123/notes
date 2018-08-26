# 文件系统
创建目录、创建文件、打开文件、文件读写
## 1. VFS中数据结构
### 1.1 super_block
超级块代表文件系统本身，对应文件系统自身的控制块结构，保存文件块大小、超级块操作函数、且所有inode要链接到超级块的链表头。

每个文件系统都有一个超级块结构，**要链接到一个超级块链表**。文件系统内每个文件在打开时都需要在内存中分配一个inode结构，**这些inode结构都要链接到超级块**。

作用：
super_blocks链表可以找到super_block1和super_block2两个文件系统的超级块，每个超级块链接到其对应的inode，即顺着super_blocks链表可以遍历整个操作系统打开过的inode文件。
```c
struct super_block {
	unsigned long s_blocksize;//文件系统块大小
	unsigned char s_blocksize_bits;
	unsigned long long s_maxbytes;//最大文件大小

	struct file_system_type *s_type;//file_system_type
	struct super_operation *s_op;//super block操作
	unsigned long s_magic;
	struct dentry *s_root;//文件系统根dentry的指针
	struct list_head s_inodes;//文件系统内所有inode
	struct list_head s_dirty;//所有dirty的inode对象
	struct block_device *s_bdev;//文件系统存在的块设备指针
	void *s_fs_info;	//文件系统私有info
}
```
### 1.2 目录项dentry
在VFS中，目录本身也是一个文件。每个文件有一个dentry（可能不止一个），这个dentry链接到上级目录的dentry。dentry树可以遍历整个文件系统的所有目录和文件。

为了加快对dentry的查找，内核使用hash表来缓存dentry，称为dentry cache。dentry的查找一般都现在dentry cache中进行查找。

作用：（1）每个文件的dentry链接到父目录的dentry，形成文件系统的结构树。即子目录的d_child成员链接到父目录的d_subdirs成员。
（2） 所有dentry都指向一个dentry_hashtable. 文件被打开过，就在dentry的hash cache中。
（3）dentry的d_mounted成员标识该目录是否为挂载点

#### 1.2.1 vfsmount
表述不同文件系统dentry之间的链接关系。
每个文件系统都有一个这个结构，当文件系统被挂载时，它的vfsmount结构就被链接到内核的一个全局链表----mount_hashtable数组链表。
mount_hashtable是一个数组，每个成员都是一个hash链表。当发现mnt目录是一个特殊的目录时，从mount_hashtable数组找到hash链表头，再遍历整个hash链表，就能找到mnt所在的文件系统的vfsmount，然后mount目录的dentry就被替换，置换为新文件系统的根目录。
```
---mount_hashtable------>vfsmount1
												-------/
									------>vfsmount2
												-------/
```
```c
struct dentry{
	struct inode *d_inode;//指向一个inode，这个indoe和dentry共同描述一个文件或目录
	struct hlist_node d_hash;	//查找hash list
	struct dentry *d_parent;	//parent directory
	struct qstr d_name;

	union{
		struct list_head d_child;//parent list的child
		struct rcu_head d_rcu;
	}
	struct list_head d_subdirs;	//本dentry的children
	struct super_block *d_sb;	//dentry tree的root
	int d_mounted;	//是否为挂载点
}
```
### 1.3 索引节点inode
inode代表一个文件，保存文件大小、创建时间、块大小、文件的读写函数、文件的读写缓存等信息。一个真实的文件可以由多个dentry，而inode只有一个。i_mode成员代表不同的文件类型如字符设备、块设备、目录。

作用：所有inode结构都要链接到inode_hashtable的某个hash链表。另一个作用是缓存文件的数据内容，i_mapping通过使用radix树来保存文件的数据。
```c
struct inode{
	struct list_head i_list;	//描述inode当前状态的链表
	struct list_head i_sb_list;	//用于链接到超级快中的inode链表
	//当创建一个新的inode时，成员i_list要链接到inode_in_use链表，表示inode在使用状态，i_sb_list链接到超级块的s_inodes链表头。由于一个文件可以对应多个dentry，这些dentry要链接到成员i_dentry
	struct list_head i_dentry;
	unsigned long i_ino;	//inode号
	atomic_t	i_count;	//inode的引用计数
	loff_t i_size;	//文件长度

	unsigned int i_blkbits;		//文件块位数
	struct inode_operation *i_op;		//
	const struct file_operation *i_fop;	//文件读写和异步IO former ->i_op->default_file_ops
	struct address_space *i_mapping;	//缓存文件的内容
	struct block_device *i_bdev;	//指向块设备的指针
}
```

### 1.4 文件
文件对象的作用是描述进程和文件交互的关系。硬盘上并不存在文件结构。同一个文件，在不同的进程中有不同的文件对象。
```c
struct file{
	struct dentry *f_dentry;	//文件的dentry结构
	struct vfsmount *f_vfsmnt;	//文件所属的文件系统的vfsmount对象
	const struct file_operation *f_op;
	loff_t f_pos;	//表进程对文件操作的位置
	struct fown_struct f_owner;
	unsigned int f_uid,f_gid;	//文件的用户ID和用户组ID
	struct file_ra_state f_ra;	//用于文件预读的设置

	struct address_space *f_mapping;	//文件的读写缓存页面

}
```
