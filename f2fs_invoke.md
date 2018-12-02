#F2FS的调用流程
本文參考https://blog.csdn.net/eleven_xiy/article/details/71249365 以f2fs文件的访问过程为例，介绍实现机制。

## 1. 文件系统的初始化过程
关键函数：init_f2fs_fs()--> register_filesystem(&f2fs_fs_type);
```c
static struct file_system_type f2fs_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "f2fs",
	.mount		= f2fs_mount,
	.kill_sb	= kill_f2fs_super,
	.fs_flags	= FS_REQUIRES_DEV,
};
```
## 2. linux系统中mount的实现过程
### 2.1 vfs中的mount过程
namespace.c
关键函数：sys_mount->do_mount-->do_new_mount-->vfs_kern_mount-->mount_fs()
- do_mount: 包括挂载点目录项名称的解析
- do_new_mount: 根据文件系统类型f2fs找到f2fs_fs_type
- vfs_kern_mount: 由此调用下去会找到真正文件系统的mount函数，如f2fs_mount等。f2fs_mount申请超级块、超级块对应的根目录项、根inode
- mount_fs: 创建超级块以及超级块对应的根目录项，每个超级块都对应根目录项和根inode = 1
```c
root = type->mount(type, flags, name, data);
```
### 2.2 f2fs的mount
#### f2fs_mount()函数
```c
static struct dentry *f2fs_mount(struct file_system_type *fs_type, int flags,
			const char *dev_name, void *data)
{
	return mount_bdev(fs_type, flags, dev_name, data, f2fs_fill_super);
  //mount_bdev是用于块设备挂载的函数。调用f2fs_fill_super
}
```
#### f2fs_fill_super()
完成超级块的申请及初始化，如f2fs_sb_info等。
 1. 设置block大小
 2. 从设备中读取super_block信息，#0和#1
 3. 根据mount时的挂载选项，设置sbi->mount的信息
 4. sbi->max_file_blocks = inode指针计算出来的最大块数目
