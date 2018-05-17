JVM 学习笔记 (2): 常用命令

# 一、jmap

### jmap -heap <pid>
```
[root@centos-linux-1 ~]# jmap -heap 15151
Attaching to process ID 15151, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.162-b12

using thread-local object allocation.
Parallel GC with 2 thread(s)        # 采用的 GC 算法

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 482344960 (460.0MB)
   NewSize                  = 10485760 (10.0MB)
   MaxNewSize               = 160432128 (153.0MB)
   OldSize                  = 20971520 (20.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 122683392 (117.0MB)
   used     = 41862848 (39.92352294921875MB)
   free     = 80820544 (77.07647705078125MB)
   34.122669187366455% used
From Space:
   capacity = 18874368 (18.0MB)
   used     = 4784128 (4.5625MB)
   free     = 14090240 (13.4375MB)
   25.34722222222222% used
To Space:
   capacity = 18350080 (17.5MB)
   used     = 0 (0.0MB)
   free     = 18350080 (17.5MB)
   0.0% used
PS Old Generation
   capacity = 52953088 (50.5MB)
   used     = 21765648 (20.757339477539062MB)
   free     = 31187440 (29.742660522460938MB)
   41.10364252978032% used

19281 interned Strings occupying 2562272 bytes.
```
