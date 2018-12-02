# OCSSD环境重新搭建，好伤
> 2018-11-21
## 1. 客户机搭建

step1：qemu-kvm安装系统
```c
qemu-img create -f qcow2 centos7-revise.qcow2 200G
qemu-system-x86_64 -m 32G centos7-revise.qcow2 -cdrom ./Centos7-CD.iso --enable-kvm
启动：
/root/qhw/femu/build-femu/x86_64-softmmu/qemu-system-x86_64 \
-name "pmy-FEMU-SSD" \
-m 32G -smp 32 --enable-kvm \
-hda image-revise/centos7-revise.qcow2 \
-net nic,macaddr=52:54:00:12:34:78 -net tap,ifname=tap1,script=./qemu-ifup.sh,downscript=no \
-qmp unix:./qmp-sock,server,nowait \
-k en-us \
-sdl
```
step2：参照lightnvm搭建文档
```
安装必要软件包
yum update
yum install git -y
yum groupinstall "Development Tools"
sudo yum install -y gcc bc openssl-devel ncurses-devel pciutils libudev-devel elfutils-libelf-devel

内核版本升级：从github下载最新然后版本回退
git clone https://github.com/torvalds/linux.git
下载很慢，发现，window本地下载速度竟然很快，本地下载后转到服务器
git log --pretty=oneline | grep "Linux 4.1*"
git reset --hard xxx(4.17)
29dcea88779c856c7dc92040a0c01233263101d4 Linux 4.17
0adb32858b0bddf4ada5f364a84ed60b196dbcda Linux 4.16
由于4.16的版本编译会报error，所以回退到4.17
后面由于与修改代码不兼容，会退到4.16-rc6

编译内核
模块化nvme、ocssd、f2fs
$ grub2-editenv list
$ grub2-set-default 0

安装lnvm和nvme-cli
$ git clone https://github.com/OpenChannelSSD/lnvm.git  
$ make
$ make install
$ lnvm devices
$ git clone https://github.com/linux-nvme/nvme-cli.git
$ cd nvme-cli
$ make
$ make install

nvme设备挂载
  创建一个空的NVMe设备
  dd if=/dev/zero of=blknvme bs=1M count=8196
  用qemu-nvme/femu启动虚拟机,qemu-system-x86_64是之前安装的qemu-nvme编译的bin中的，不是kvm的。

/root/qhw/femu/build-femu/x86_64-softmmu/qemu-system-x86_64 \
-name "pmy-FEMU-SSD" \
-m 32G -smp 32 --enable-kvm \
-hda image-revise/centos7-revise.qcow2 \
-device virtio-scsi-pci,id=scsi0 \
-drive file=image-revise/vssd1.raw,if=none,aio=threads,format=raw,id=id0 \
-device nvme,drive=id0,serial=serial0,id=nvme0,namespaces=1,lver=1,lmetasize=16,ll2pmode=0,nlbaf=5,lba_index=3,mdts=10,lnum_ch=8,lnum_lun=1,lnum_pln=1,lsec_size=4096,lsecs_per_pg=1,lpgs_per_blk=512,ldebug=1,femu_mode=0 \
-net nic,macaddr=52:54:00:12:34:78 -net tap,ifname=tap1,script=./qemu-ifup.sh,downscript=no \
-qmp unix:./qmp-sock,server,nowait \
-k en-us \
-sdl
```

step 3: lnvm，nvme-core，nvme，pblk的模块化

- 准备好lightnvm_modules代码，其中拷贝nvme，lightnvm代码
```
报错
nvme挂载时：
nvme: Unknown symbol nvme_uninit_ctrl (err 0)
修改.h后重新编译一次
```
- 根据笔记模块化lightnvm(完全一样)
- 制作makefile
- blacklist掉系统的nvme模块，vi /etc/modprobe.d/blacklist.conf 添加：blacklist nouveau

```
问题，加载nvme.ko后崩掉
blacklist掉系统的nvme后重启，不崩
```
- 初始化pblk
```
nvme lnvm list   #可看到我们模拟的设备
# 初始化设备
nvme lnvm create -d nvme0n1 --lun-begin=0 --lun-end=3 -n mydevice -t pblk
mkfs.ext4 /dev/mydevice
```
问题：mkfs时出现
```
pblk: metadata I/O failed. Line0...
主机端报：femu_oc_rw: I/O does not respect device write constrains.Sectors send: (0). Min:1 sectors required
```
重新调整nvme配置的参数，调整lsecs_per_pg = 4即可解决

## 2. 修改代码
step1: 拷贝一份上面做好的nvme和pblk
报错：
```
/root/lightnvm_modules/pblk-revise/core.c:127:31: error: ‘struct nvm_geo’ has no member named ‘nr_luns’
     int lunid = (ch * dev->geo.nr_luns) + lun;
```
呵呵呵呵呵呵呵，版本不同，里面的函数不同，呵呵呵呵呵呵，直接把本地的linux内核拷贝过了
不过还是想试一下4.16-rc6

好了，继续！
1. 将代码这边的nvme和lightnvm代码复制过去，编译，insmod，成功
2. 将liblightnvm代码拷贝过去

```
修改CMakeLists.txt
add_library(${LNAME} STATIC ${HEADER_FILES} ${SOURCE_FILES})
改为
add_library(${LNAME} SHARED ${HEADER_FILES} ${SOURCE_FILES})

make configure
make
sudo make install

ps:CUnit
yum install centos-release-scl-rh
yum install CUnit
yum install CUnit-devel
```
3. 编译f2fs-tools
```
# autoreconf --install
# ./configure
# make
```
4. 修改include/linux/lightnvm.h
```
extern int __nvm_configure_create(struct nvm_ioctl_create *);
```
5. mkfs.f2fs报错
```
不能format /dev/nvme0n1
主机端：
femu_oc_rw: I/O does not respect device write constrains.Sectors send: (1). Min:4 sectors required
```
6. mount
报错
```
F2FS MOUNT ....
SELinux: (dev nvme0n1, type f2fs) has no security xattr handler
F2FS Umount....
禁用selinux即可
vi /etc/selinux/config
将SELINUX=enforcing改为SELINUX=disabled，重启生效
```
> end 
