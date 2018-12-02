# 遇到的问题
1. femu使用blackssd时，创建vssd1.conf要在运行命令的当前文件
2. 搜不到可以搜源码，看问题是在何处产生的
3. 之前报的mount: mount  on /mnt/f2fs failed: Operation not supported问题，在使用femu的black-ssd后，成功mount。注意：要将之前的lightnvm的配置都改回来，用正常的nvme驱动即可。
4. 在/dev/mypblk中挂载f2fs,可以mkfs但是mount时会出现mount: mount on /dev/mypblk failed: operation not supported。由于xattr等属性没有配置
