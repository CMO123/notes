# 记录不太懂的小东西
1. kname = (char *)result->iname;//将kname所指向的地址接在result变量之后
2. const char __user *filename中，__user表示用户空间
3. 通过检查挂载点目录的dentry结构的d_mounted成员是否为0判断挂载点目录是否已经被挂载。
