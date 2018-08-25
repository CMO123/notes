# 文件系统
## 1. VFS中数据结构
### 1.1 super_block
超级块代表文件系统本身，对应文件系统自身的控制块结构，保存文件块大小、超级块操作函数、且所有inode要链接到超级块的链表头。
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
inode代表一个文件，保存文件大小、创建时间、块大小、文件的读写函数、文件的读写缓存等信息。一个真实的文件可以由多个dentry，而inode只有一个。
```c
struct inode{

	
}
```
