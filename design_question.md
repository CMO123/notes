# 阅读资料的一些问题和思考

1. extent方法减少数据块总索引大小？？
2. f2fs inode大小4KB比ext4大，浪费存储空间
3. 普通文件和目录文件内联技术，<font color="red">文件类型的划分？？</font>
4. zone: 一个固态SSD的多个flash 原则上可以进行**并发的操作**。F2FS在同一时刻会维护6个不同的segment用于日志写入。会尽量从不同的zone中选取6个segment。
5. F2FS的文件系统附加属性xattr？？可实现内联技术，占用了923个地址中的50个地址。
6. 为什么不动态设置inode内联的大小？？指针和内联结合？？
7. f2fs_inode_info存放f2fs_inode的私有信息。
8. <font color="red">数据压缩技术</font>
9. 修改底层调用接口，是修改<font color="red">???????</font>
```c
sb_bread(sb, SIMPLEFS_SUPERBLOCK_BLOCK_NUMBER);
....
mark_buffer_dirty(bh);
sync_dirty_buffer(bh);
```
10. 锁的并行性问题
```c
 FIXME: This can be moved to an in-memory structure of the simplefs_inode.
 * Because of the global nature of this lock, we cannot create
 * new children (without locking) in two different dirs at a time.
 * They will get sequentially created. If we move the lock
 * to a directory-specific way (by moving it inside inode), the
 * insertion of two children in two different directories can be
 * done in parallel 
static DEFINE_MUTEX(simplefs_directory_children_update_lock);
```
