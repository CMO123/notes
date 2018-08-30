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
## 2.文件系统代码分析
### 2.1 初始化流程
1. register_filesystem，将aufs文件系统登记到系统
2. kern_mount为文件系统申请必备的数据结构
3. 在aufs系统中创建目录

### 2.2 register_filesystem
将文件系统加入文件系统链表file_systems
### 2.3 kern_mount
为文件系统分配了**超级块对象、根inode、根dentry**和**vfsmount对象**

1. **kern_mount(struct file_system_type *type) -->**
2. **vfs_kern_mount(type, 0, type->name, NULL)  **
[ 1.根据文件系统名字**创建vfsmount结构**；2. 根据文件系统自定义的**type->get_sb()**创建一个**超级块对象**，还要创建一个dentry结构作为**root dentry**，一个inode作为**根inode**；3. 设置vfsmount结构的父指针为自身或源文件系统] -->
3. **.get_sb=aufs_get_sb --> get_sb_single(fs_type, flags, data, aufs_fill_super)**  
[ 1. 获得一个超级块对象， 检查是否有根dentry，若无，则用fill_super为超级块对象填充根dentry和根inode；2. simple_set_mnt函数将超级块对象赋值给vfsmount结构所指的超级块，且vfsmount所指mnt_root为超级块的根dentry。至此，**vfsmount可以获得文件系统的超级块对象和根dentry**]  -->
4. **fill_super() --> simple_fill_super（）**
为超级块申请必备的成员
5. **simple_fill_super()具体填充**
（1）为超级块对象赋值，包括文件系统块大小，超级块对象操作函数
（2）创建一个inode结构作为根inode，设置mode为目录，块尺寸和文件大小，操作函数
（3）d_alloc_root创建根dentry

### 2.4 aufs_create_dir流程
aufs_create_dir(name, parent) --> aufs_create_file(name, S_IFDIR..., parent) [执行创建文件]  --> aufs_create_by_name() [创建文件的dentry和inode] --> lookup_one_len() [获得一个dentry结构，lookup_one_len首先在父目录下查找dentry结构，存在同名dentry返回指针，不存在则创建一个dentry] --> 通过aufs_mkdir或者aufs_create创建inode--> aufs_mknod()通过调用aufs_get_inode创建文件inode并调用d_instantiate将dentry加入inode的dentry链表头  --> aufs_get_inode() [创建一个inode，将inode加入超级块对象的inode链表中，inode还要加入一个全局变量链表inode_in_use]

## 3. 文件系统的挂载过程
mount通过系统调用sys_mount来执行，（从unistd.h中找到sys_mount,注释会说明实现在fs/namespace.h中，然后
```c
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
		char __user *, type, unsigned long, flags, void __user *, data)
```
其中第一个参数mount加上sys_即构成sys_mount。sys_mount又调用do_mount。

VFS中mount过程:namespace.c
### 3.1 do_mount（）
首先获得挂载点目录的dentry结构以及目的文件系统的vfsmount结构，这些信息保存在nameidata中。然后根据mount选项调用不同的函数执行mount。-->
```c
long do_mount(const char *dev_name, const char __user *dir_name,
		const char *type_page, unsigned long flags, void *data_page)
{//首先获得挂载点目录的dentry结构以及目的文件系统的vfsmount结构放入path中
// dev_name : /dev/nvme0n1
// dir_name : /mnt/test
// type_page: f2fs
// flags : flags
// data_page: options
	struct path path;
	unsigned int mnt_flags = 0, sb_flags;
	int retval = 0;

	/* ... and get the mountpoint */
	retval = user_path(dir_name, &path);//dir_name = "/mnt/test" 找到挂载点的目录项path.dentry->d_iname = test
	//kern_path()找到挂载点的目录项,即此时path.dentry->d_iname=test;通过如下调用kern_path->filename_loopup->path_loopupat找到struct path，
	//此时path_init():nd->path.mnt->mnt_root->d_iname="/",根目录超级块是SQUASHFS_MAGIC类型,
	//nd->path.mnt->mnt_sb->s_magic是根目录对应的超级块;
	//ps:current->fs->root.dentry->d_iname='/';  user_path->path_lookupat->path_init/link_path_walk;

	if (flags & MS_REMOUNT)
		retval = do_remount(&path, flags, sb_flags, mnt_flags,
				    data_page);
	else if (flags & MS_BIND)
		retval = do_loopback(&path, dev_name, flags & MS_REC);
	else if (flags & (MS_SHARED | MS_PRIVATE | MS_SLAVE | MS_UNBINDABLE))
		retval = do_change_type(&path, flags);
	else if (flags & MS_MOVE)
		retval = do_move_mount(&path, dev_name);
	else
		retval = do_new_mount(&path, type_page, sb_flags, mnt_flags,
				      dev_name, data_page);
	// 根据不同文件系统实现挂载，如：jffs2_mount,ubifs_mount等主要功能是实现super_block申请及初始化。
	// 注意此时path.dentry是要挂载到目录 的目录项
dput_out:
	path_put(&path);
	return retval;
}
```

### 3.2 do_new_mount（）
执行mount的大部分事情，其中的do_add_mount将源文件系统挂载到目的文件系统。
```c
/*
 * create a new mount for userspace and request it to be added into the
 * namespace's tree
 */
static int do_new_mount(struct path *path, const char *fstype, int sb_flags,
			int mnt_flags, const char *name, void *data)
{//参数(&path, type_page, sb_flags, mnt_flags,dev_name, data_page)
	struct file_system_type *type;
	struct vfsmount *mnt;

	type = get_fs_type(fstype);
	/*
	由此调用下去，会调用到真正的文件系统mount函数,如：jffs2_mount()/ubifs_mount()等
	jffs2_mount/ubifs_mount中申请到的超级块，该超级块对应的根目录项、根inode都赋值给了mnt，详见vfs_kern_mount中实现。
	此时：mnt->mnt_root->d_iname="/",这个目录项是此次mount过程创建的超级块对应的根目录项，和根文件系统的"/"对应根目录项不是同一个，只是名字相同。
	即：此处的mnt->mnt_root与do_mount->user_path()中的path.mnt->mnt_root，不是同一个根目录项(user_path得到的是要挂载的目录所在超级块的根目录，如挂载到/mnt/test,
	则超级块对应根文件系统，是squashfs类型的.而此处是挂载时创建的超级块的根目录项，文件系统类型是挂载时mount -t 参数指定的).但根目录项的名字相同（都是‘/’）.
	*/
	mnt = vfs_kern_mount(type, sb_flags, name, data);

	put_filesystem(type);//只要知道它是把module模块加载到内核中。

	err = do_add_mount(real_mount(mnt), path, mnt_flags);
	if (err)
		mntput(mnt);
	return err;
}
```
### 3.3  vfs_kern_mount()
```c
struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{//参数（type，flags，dev_name, option）
	struct mount *mnt;
	struct dentry *root;

	mnt = alloc_vfsmnt(name);

	if (flags & SB_KERNMOUNT)
		mnt->mnt.mnt_flags = MNT_INTERNAL;
	//该函数创建超级块及该超级块的根目录项和根inode.
	// 注意此处mount_fs中返回的是目录项sb->s_root,可以在后文看到sb->s_root目录项是指向 '/'
	root = mount_fs(type, flags, name, data);
	/*
	把此次mount->squashfs_fill_super过程创建的超级块对应的根目录项dentry，根inode和超级块，都赋值给vfsmount ，vfsmount返回到上级调用函数do_new_mount
	此时：mnt->mnt_root->d_iname="/",这个目录项是此次mount过程创建的超级块对应的根目录项，和根文件系统的"/"对应根目录项不是同一个，只是名字相同。
	即：此处的mnt->mnt_root与do_mount->user_path()中的path.mnt->mnt_root，不是同一个根目录项(user_path是要挂载的目录所在超级块的根目录，如挂载到/mnt/test,
	则超级块对应根文件系统，是squashfs类型的，而此处是挂载时创建的超级块的根目录项，文件系统类型是挂载时mount -t 参数指定的).但根目录项的名字相同（都是‘/’）
	*/
	mnt->mnt.mnt_root = root;
	mnt->mnt.mnt_sb = root->d_sb;
	mnt->mnt_mountpoint = mnt->mnt.mnt_root;
	mnt->mnt_parent = mnt;
	lock_mount_hash();
	list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);
	unlock_mount_hash();
	return &mnt->mnt;
}
EXPORT_SYMBOL_GPL(vfs_kern_mount);
```

### 3.4 mount_fs()
创建超级块及超级块对应的根目录项. 每个超级块都对应根目录项（目录项名即dentry->d_iname="/"）和根节点(inode->i_ino=1);
```c
struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
	struct dentry *root;
	struct super_block *sb;
	char *secdata = NULL;
	int error = -ENOMEM;
	/*
	从VFS中的mount调用真实文件系统的jffs2_fs_type->mount=jffs2_mount
	创建super_block: sget(fs_type, get_sb_mtd_compare,get_sb_mtd_set, flags, mtd);
	此时name为mount命令中诸如/dev/mtdblock7的设备名;
	注意：
	1）此时type->mount的返回值root是目录项dentry：sb->s_root,而超级块sb是在jffs2_mount->mount_mtd_aux->sget中创建的，sb->s_root目录项在
	jffs2_mount->jffs2_fill_super中初始化
	2）而对于ubifs文件系统：sb是在ubifs_mount->sget中创建，sb->s_root是在ubifs_mount->ubifs_fill_super中初始化。
	*/

	root = type->mount(type, flags, name, data);

	sb = root->d_sb;

	up_write(&sb->s_umount);
	free_secdata(secdata);
	return root;
}
```
### 3.5 文件系统中的mount
fs/f2fs/super.c
static struct dentry *f2fs_mount(struct file_system_type *fs_type, int flags,
			const char *dev_name, void *data)
{
	return mount_bdev(fs_type, flags, dev_name, data, f2fs_fill_super);//mount_bdev是用于块设备挂载的函数。调用f2fs_fill_super
}
#### 3.5.1 mount_bdev()
在fs/super.c中
```c
struct dentry *mount_bdev(struct file_system_type *fs_type,
	int flags, const char *dev_name, void *data,
	int (*fill_super)(struct super_block *, void *, int))
{	/*
	 * once the super is inserted into the list by sget, s_umount
	 * will protect the lockfs code from trying to start a snapshot
	 * while we are mounting
	 */

	//申请super_block
	s = sget(fs_type, test_bdev_super, set_bdev_super, flags | SB_NOSEC,
		 bdev);

	if (s->s_root) {
		...
	} else {
		s->s_mode = mode;
		snprintf(s->s_id, sizeof(s->s_id), "%pg", bdev);
		sb_set_blocksize(s, block_size(bdev));
		error = fill_super(s, data, flags & SB_SILENT ? 1 : 0);
		//jffs2_fill_super中初始化super_block;同理：ubifs_fill_super中初始化super_block
		s->s_flags |= SB_ACTIVE;
		bdev->bd_super = s;
	}
	return dget(s->s_root);
	//返回值root是目录项dentry：sb->s_root,而超级块sb是在jffs2_mount->mount_mtd_aux->sget中创建的，
	//sb->s_root目录项在jffs2_mount->jffs2_fill_super中初始化
}
EXPORT_SYMBOL(mount_bdev);
```
#### 3.5.2 f2fs_fill_super()
read_raw_super_block()从设备读取super_block信息。

### 3.6 mount总结
至此，完成了mount过程的介绍，简要回顾一下，mount过程系统都做了什么？
1. 创建了超级块super_bock，这个超级块类型为mount -t指定的类型。在诸如：jffs2_fill_super->jffs2_do_fill_super中完成.

2. 为新创建的超级块创建根目录项(dentry->d_iname=“/”)和相应的inode(i_ino=1),即：每次mount对应创建一个超级块，每个超级块都有一个名为"/"的根目录项，和一个inode->i_ino=1的inode，注意如果是通过工具制作好的文件系统，烧录到指定flash分区上，则inode->i_ino要从flash上读取，不一定是1，
在诸如：mount->jffs2_fill_super->jffs2_do_fill_super函数中完成.

3. 将2中新创建的根目录项和根inode赋值给vfsmount变量(vfs_kern_mount函数中完成)，并在do_new_mount->do_add_mount系列函数中对vfsmount进行管理。

4. 以后如果在mount挂载到的目录下创建文件或目录，都是基于mount挂载过程中创建的超级块进行操作。


## 4. 文件系统读写
### 4.1 文件的访问
访问文件时，struct file、 super_block、inode、dentry、address_space结构重要。
打开文件：
sys_open -> do_filp_open -> path_openat -> do_last -> lookup_dcache ->d_alloc ->__d_alloc
### 4.2 文件的打开
在fs/open.c中
```c
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{	if (force_o_largefile())
		flags |= O_LARGEFILE;
	return do_sys_open(AT_FDCWD, filename, flags, mode);
}
```
do_sys_open():
```c
struct file *f = do_filp_open(dfd, struct filename tmp, &op);
```
fs/namei.c中do_filp_open():得到struct file filp的指针。
```c
filp = path_openat(&nd, op, flags | LOOKUP_RCU);
```
在open.c的do_dentry_open()中赋值
```c
f->f_op = fops_get(inode->i_fop);
```
### 4.3 文件的读操作
1. VFS读部分：
在fs/read_write.c中
```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	struct fd f = fdget_pos(fd);
	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_read(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}
	return ret;
}
```
vfs_read(struct file, char *, size_t, loff_t):
```c
ret = rw_verify_area(READ, file, pos, count);
ret = __vfs_read(file, buf, count, pos);
```
__vfs_read():
```c
//如果文件定义了read函数，调用文件自身的读函数。
	if (file->f_op->read)
		return file->f_op->read(file, buf, count, pos);
	else if (file->f_op->read_iter)
		return new_sync_read(file, buf, count, pos);
```
2. f2fs读操作
f->f_op = inode->i_fop;而inode的->i_fop。 操作赋值在f2fs/inode.c的f2fs_iget()函数中
```c
make_now:
	if (ino == F2FS_NODE_INO(sbi)) {
		inode->i_mapping->a_ops = &f2fs_node_aops;
		mapping_set_gfp_mask(inode->i_mapping, GFP_F2FS_ZERO);
	} else if (ino == F2FS_META_INO(sbi)) {
		inode->i_mapping->a_ops = &f2fs_meta_aops;
		mapping_set_gfp_mask(inode->i_mapping, GFP_F2FS_ZERO);
	} else if (S_ISREG(inode->i_mode)) {
		inode->i_op = &f2fs_file_inode_operations;
		inode->i_fop = &f2fs_file_operations;
		inode->i_mapping->a_ops = &f2fs_dblock_aops;
	} else if (S_ISDIR(inode->i_mode)) {
		inode->i_op = &f2fs_dir_inode_operations;
		inode->i_fop = &f2fs_dir_operations;
		inode->i_mapping->a_ops = &f2fs_dblock_aops;
		mapping_set_gfp_mask(inode->i_mapping, GFP_F2FS_HIGH_ZERO);
	} else if (S_ISLNK(inode->i_mode)) {
		if (f2fs_encrypted_inode(inode))
			inode->i_op = &f2fs_encrypted_symlink_inode_operations;
		else
			inode->i_op = &f2fs_symlink_inode_operations;
		inode_nohighmem(inode);
		inode->i_mapping->a_ops = &f2fs_dblock_aops;
	} else if (S_ISCHR(inode->i_mode) || S_ISBLK(inode->i_mode) ||
			S_ISFIFO(inode->i_mode) || S_ISSOCK(inode->i_mode)) {
		inode->i_op = &f2fs_special_inode_operations;
		init_special_inode(inode, inode->i_mode, inode->i_rdev);
}
```
普通文件的读写操作函数如下(file.c):
```c
const struct file_operations f2fs_file_operations = {
	.llseek		= f2fs_llseek,
	.read_iter	= generic_file_read_iter,
	.write_iter	= f2fs_file_write_iter,
	.open		= f2fs_file_open,
	.release	= f2fs_release_file,
	.mmap		= f2fs_file_mmap,
	.flush		= f2fs_file_flush,
	.fsync		= f2fs_sync_file,
	.fallocate	= f2fs_fallocate,
	.unlocked_ioctl	= f2fs_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= f2fs_compat_ioctl,
#endif
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
};
```
