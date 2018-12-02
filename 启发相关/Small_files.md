> https://scicomp.aalto.fi/triton/usage/smallfiles.html
> https://docs.gluster.org/en/latest/Administrator%20Guide/Performance%20Testing/
第一个链接，额，感觉用处不大

# Gluster performance testing
## Testing tools
- fio - for large file I/O tests.
- smallfile - for pure-workload small-file tests
- iozone - for pure-workload large-file tests
- parallel-libgfapi - for pure-workload libgfapi tests
### smallfile Distributed I/O Benchmark
Smallfile是一个基于python的小文件分布式POSIX工作负载生成器，它可以用于快速度量整个集群中各种元数据密集型工作负载的性能。它不依赖于任何特定的文件系统或实现AFAIK。它运行在Linux、Windows上，并且应该也能在大多数UNIX上工作。它旨在补充使用iozone基准来测量大文件工作负载的性能，并借用iozone和ricwerfs_mark的某些概念。它是由本英格兰从2009年3月开始开发的，现在是开源的(ApacheLicense V2)。

下面是一个典型的简单测试序列，然后在后续测试中使用初始创建测试中的文件。有更多的小文件操作类型比这5(参见doc)，但这些是最常用的。
```c
SMF="./smallfile_cli.py --top /mnt/glusterfs/smf --host-set h1,h2,h3,h4 --threads 8 --file-size 4 --files 10000 --response-times Y "
    $SMF --operation create
    for s in $SERVERS ; do ssh $h 'echo 3 > /proc/sys/vm/drop_caches' ; done
    $SMF --operation read
    $SMF --operation append
    $SMF --operation rename
    $SMF --operation delete
```

# small files

在任何文件系统上，数百万个小文件都是一个巨大的问题。您可能认为/Scratch是一个快速的文件系统，没有这个问题，但实际上这里更糟。lustre(Scatch)就像对象存储一样，并将文件与medatata分开存储。这意味着每个文件访问都需要多个不同的网络请求，而大量的文件使得您的研究(和管理集群)陷于停顿。什么很重要？默认配额是1e6文件。对于一个项目来说，1e4并不是很多。但对一个项目的1e6是一个问题。(与lustre fs有关)


你可能被指示到这里是因为你有很多文件。在这种情况下，欢迎来到大数据世界，即使你的总规模不是那么大！(这不仅仅是尺寸，而是使用普通工具的困难)请阅读这篇文章，看看你能学到什么，并问我们你是否需要帮助。

## contents
### The problem with small files
你知道，Lustre是高性能和快速。但是，访问每个文件的开销相对较高。下面，您可以看到一些示例传输速率，并且可以看到，当文件变小时，总体性能会急剧下降。(这些数字是2016年前的lustre系统，现在更好了，但同样的原则也适用。)当您试图读取文件时，这不仅是一个问题，在管理、移动、迁移等方面也是一个问题。

|file size| net transfer rate, many files of this size|
|:---|:---|
| 10GB| 1100MB/s|
|100MB| 990MB/s|
|1MB| 90MB/s|
|10KB|.9MB/s|
|512B|.04MB/S|
### Why do people make millions of small files?
我们理解人们制作大量文件的原因：它很方便。下面是一些常见的问题(以及其他解决方案)，人们可能试图用大量的文件来解决这些问题。
- 平面文件是通用格式。如果它自己的文件中包含所有内容，那么任何其他程序都可以单独查看任何数据。很方便。这是一个快速的方式开始和使用东西的方式。
- 与其他程序的兼容性。和上面一样。
- 能够使用标准的Unixshell工具。也许您的整个预处理管道是将每个数据放在自己的文件中，并在其上运行不同的标准程序。毕竟这是Unix方式。使用文件系统作为索引。假设您有一个读取/写入数据的程序，该程序由不同的key选择。它需要分别定位每个键的数据。将所有这些都放入自己的文件中是很方便的：这将充当数据库索引的角色，您只需使用所需密钥的名称打开该文件。但是文件系统不是一个很好的索引。
  - 一旦您获得太多的文件，数据库是正确的工作工具。有些数据库作为单个文件运行，因此实际上非常容易。
- **<font color="red">并发性</font>**：您使用文件系统作为并发层。如果提交一堆作业，每个作业都会将数据写入自己的文件。因此，您不必担心附加到同一个文件/数据库同步/锁定/等等的问题。这其实是一个很普遍的原因。
  - 这是个大原因。文件系统是连接不同作业(例如数组作业)输出的最可靠方法，很难找到更好的策略。继续这样做是合理的，并在第二阶段合并作业输出以减少文件数量
- 安全性/安全性：文件系统将不同的文件彼此隔离开来，所以如果您修改一个文件，那么破坏其他文件的可能性就更小了。这和上面的理由是一致的。
- 在日常工作中，您一次只访问几个文件，因此您永远不会意识到存在问题。然而，当我们试图管理数据(迁移、移动等)时，就会出现一个问题。
- 要意识到forking过程有类似的开销。small reads也是不理想的，但不那么坏(？)

### **<font color="red">Strategies</font>**
- 意识到你将不得不改变你的工作流程。你不能用grep，排序，WC等做所有的事情。更多恭喜你，你有大数据。
- 为你的计划考虑正确的策略：一个严肃的计划应该为此提供选择。
  - 例如，我看到了一些机器学习框架，这些框架提供了一个选项，可以将所有输入数据压缩到一个为读取优化的文件中。这正是为这种情况而设计的。您可以单独读取所有文件，但会慢一些。因此，在这种情况下，人们应该首先阅读文档，并看到有一个解决方案。其中一个将获取所有原始文件并制作处理后的输入文件。然后，将原始的培训数据打包到一个压缩的档案中进行长期存储.如果您需要查看单个输入文件，则始终可以逐个解压缩。
- Split - combine - analyze
  - 继续像你一直在做的那样：每个(数组？)作业生成不同的输出文件。然后，运行后，将输出合并到一个文件/数据库中。清理/存档中间文件。使用这个合并的db/file来分析长期的数据。这可能是调整工作流程的最简单方法。
- HDF5: 特别是对于数值数据，这是一个很好的格式组合您的结果。就像文件中的文件系统一样，您仍然可以根据不同的键命名数据，以便进行单独的访问。
- 解压到本地磁盘，完成后打包为scratch。
  - 主要文章：计算节点本地驱动
  - 这个策略可以与下面的许多其他策略结合起来
  - 这个策略特别好，当您的数据是写一次读多次的时候。您将所有原始数据打包到一个方便的存档中，并在需要时将其解压到本地磁盘。当你完成时你会删除它。
- 使用适合您的领域的适当数据库(Sqite)：存储大量的小数据，在那里任何东西都可以快速找到，而且您可以高效地进行计算，这正是数据库所做的。很难有一个通用数据库为您工作，但现在有各种各样的特殊用途数据库。其中之一是否适合储存您的计算结果进行分析？
  - 请注意，如果您确实是在执行高性能的随机IO，那么将数据库放在Scratch上并不是一个好主意，您需要考虑更多。考虑将其与本地磁盘相结合：您可以将预先创建的数据库文件复制到本地磁盘，并执行所需的所有随机访问。完成后删除。如果需要，可以直接进行修改/更改。
- KV store: 基本上，这是一个更通用的数据库。它存储特定密钥的任意数据。
- Read all data to memory.
- compress them down when done.
- make sure you have proper backups for large files, mutating files introduces risks!
- 如果您有其他程序只能在单独的文件上操作
  - 这是一个很困难的情况，那么请调查一下结合上面的策略可以做些什么。至少您可以在完成时打包，并且可能在访问时复制到本地磁盘是一个好主意。
- MPI-I/O:  if you are writing your own MPI programs, this can parallelize output
