# 用fio测试文件系统
## 1. 测试不同channel的影响，1,4,8,16,32
### 1.1 在pblk设备上
开始设置jobs为1，线程少，负载少，增加job到32
1. 虚拟机模拟中，channel配置lnum_ch=1,lnum_lun=1
2. 启动mount-pblk-for-f2fs.sh挂载nvme，pblk，f2fs等模块
3. 修改pblk的挂载个数start.sh
4. 修改fio
5. 运行fio

结果：
f2fs并没有比ext4性能差，反而好很多

猜想：
1. 在pblk设备上
2. 设备很大，没有触发GC

### 1.2 在femu的ssd上测
注意：要修改vssd.conf和vssd.raw两个设备
