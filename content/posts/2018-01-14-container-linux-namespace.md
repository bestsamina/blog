+++
title = "[Build container] Container with Linux Namespace"
tags = ["container", "docker", "Go", "Linux"]
categories = ["container"]
date = 2018-01-14T14:51:05+08:00
+++

目錄：  
* 前言  
* Linux Namespace 類別  
* The system calls of namespaces API  
* UTS namespaces (CLONE_NEWUTS)  
* IPC namespaces (CLONE_NEWIPC)  
* PID namespaces (CLONE_NEWPID)  
* Mount namespaces (CLONE_NEWNS)  
* User namespaces (CLONE_NEWUSER)  
* Network namespaces (CLONE_NEWNET)  

# 前言
Linux Namespace 是 Linux Kernel 的其中一個功能，可以隔離對應的系統資源。
關於深入了解 Linux Namespace ，會再另闢一篇來探討。
而這邊就先照著進行，不過為了想法上能夠知道在做什麼，我是套用 C++ 的 Namespace 概念來幫助我先進行實作，讓我透過實際的程式與操作後，而對應到原來這就是 Linux Namespace 以及要怎麼實現。
PS. 建議有一點 OS 概念跟著做會比較好，要不然，最起碼知道 PID 是什麼。

### 實作環境摘要
- Ubuntu 16.04
- Kernel version: 4.13.0-1002-gcp
- Go version: v1.9.2 linux/amd64

# Linux Namespace 類別
依據 Linux Programmer's Manual 的 [Namespace](http://man7.org/linux/man-pages/man7/namespaces.7.html) 文件中提到 "One use of namespaces is to implement containers."。所以當然要來知道 Linux 提供哪些 namespaces 囉！  
主要分為以下 7 類，這邊除了 Cgroup 會另外探討外，其他的六種會在這邊來練習實做。  

```
       Namespace   Constant          Isolates
       Cgroup      CLONE_NEWCGROUP   Cgroup root directory
       IPC         CLONE_NEWIPC      System V IPC, POSIX message queues
       Network     CLONE_NEWNET      Network devices, stacks, ports, etc.
       Mount       CLONE_NEWNS       Mount points
       PID         CLONE_NEWPID      Process IDs
       User        CLONE_NEWUSER     User and group IDs
       UTS         CLONE_NEWUTS      Hostname and NIS domain name
```

# The system calls of namespaces API 
- clone(): 創建新的 process 。當 flag 的參數包含一個或多個 CLONE_NEW* 的 flags，child process 會被包含在新的 namespace 中。
- setns(): 允許 calling process 加入現有的 namespace。
- unshare(): 將 calling process 移到新的 namespace。

# UTS namespaces (CLONE_NEWUTS)
### 介紹
隔離 hostname 和 NIS domain name 兩個系統 identifiers。這些 identifiers 可使用 sethostname() 和 setdomainname() 來設定，並且使用 uname(), gethostname() 和 getdomainname() 來查詢。

Memo: `UTS = UNIX Time-sharing System` (ref: https://lwn.net/Articles/531114/)
### 實作流程
#### Step 1. 撰寫程式
使用  namespace API 的 clone() 來創建一個新的 process 並透過 CLONE_NEWUTS flag 來創建一個 UTS namespace 。創建好 namespace 之後，進入 sh (Go lang 範例程式)或 bash (C 範例程式) 環境。  
  
- Go lang: [https://github.com/sufuf3/mygo-container/blob/master/namespace/uts.go](https://github.com/sufuf3/mygo-container/blob/master/namespace/uts.go) 
- C lang: [https://github.com/sufuf3/myc-container/blob/master/namespace/uts.c](https://github.com/sufuf3/myc-container/blob/master/namespace/uts.c)

#### Step 2. 執行程式
- Go lang: `sudo go run uts.go`
- C lang: `gcc uts.c -o uts.o` -> `sudo ./uts.o`

#### Step 3. 觀察

1\. 開另外一個 terminal (或是使用 tumx 等工具切到新的視窗也是可以)，先來看看在 host 本身的 process ，這邊使用 pstree 來將 process 的關係使用樹狀圖來呈現，指令我們下 `pstree -pl` (-p 為 顯示 pid, -l 是不要將長的那一行截斷)。會取得底下類似的內容

```
$ pstree -pl
─bash(5940)───sudo(9098)───go(9099)─┬─uts(9117)─┬─sh(9121)
                                    │           ├─{uts}(9119)
                                    │           └─{uts}(9120)
```

2\. 然後回到我們程式進到的 sh 環境下 `echo $$`

```
# echo $$
9121
```

會發現我們程式目前的 PID 是 9121。  

3. 來確認程式的 namespace type 和 inode number ，這邊我們使用下面的驗證方法 (詳細可以到 http://man7.org/linux/man-pages/man7/namespaces.7.html )

```
# readlink /proc/$$/ns/uts
uts:[4026532211]
# readlink /proc/9121/ns/uts
uts:[4026532211]
# readlink /proc/9117/ns/uts
uts:[4026531838]
```

這邊我們知道子 process 和父 process 的 inode number 是不同的，也確認是他們是不同的 UTS namespace。(因為在 Linux 3.8 後，這些檔案使用 symbolic links，所以如果兩個 processes 都在同一個 namespace，那麼他們的 inode numbers 都會是一樣的)  

4\. 來透過程式改 hostname 看看是否真的會改到 host 的 hostname。

```
# hostname -b mylittlecontainer
# hostname
mylittlecontainer
```

答案當時是沒法改到囉！所以第一步 UTS namespace 隔離成功。因為 host 的 hostname 設定無法被程式內部影響到。  

# IPC namespaces (CLONE_NEWIPC)
### 介紹
隔離 IPC resources, namely, System V IPC objects 和 POSIX message queues。
Memo: `IPC = interprocess communication` (ref: http://ggav8d.blogspot.tw/2012/10/linux-linux-ipc.html)
> 該找時間全面理解 IPC 了 QQ ，我對 IPC 的觀念並沒有融會貫通，這邊就是照著做的概念了。 幸好，還算知道什麼是 semaphore, message queue, pipe, share memory。QQ

### 實作流程
#### Step 1. 撰寫程式
使用  namespace API 的 clone() 來創建一個新的 process 並透過 CLONE_NEWUTS 和 CLONE_NEWIPC flags 來創建 namespace 。創建好 namespace 之後，進入 sh (Go lang 範例程式)或 bash (C 範例程式) 環境。 (其實是 UTS 的程式碼再改動一點點而已)
- Go lang: [https://github.com/sufuf3/mygo-container/blob/master/namespace/ipc.go](https://github.com/sufuf3/mygo-container/blob/master/namespace/ipc.go) 
- C lang: [https://github.com/sufuf3/myc-container/blob/master/namespace/ipc.c](https://github.com/sufuf3/myc-container/blob/master/namespace/ipc.c)

#### Step 2. 執行程式
- Go lang: `sudo go run ipc.go`
- C lang: `gcc ipc.c -o ipc.o` -> `sudo ./ipc.o`

#### Step 3. 觀察
在第一次的 namespace 中在創建一個新的 namespace 可以看到在新的 namespace 中的 message queue 沒有原本 namespace 的 message queue ，可以知道 IPC 已完全被隔離。

- Memo: 
    1. ipcs 查询 process 通訊的狀態
    2. ipcmk - make various IPC resources 
(ref: http://man7.org/linux/man-pages/man1/ipcmk.1.html)

```
$ sudo go run ipc.go
# readlink /proc/$$/ns/uts /proc/$$/ns/ipc
uts:[4026532211]
ipc:[4026532212]
# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

# ipcmk -Q
Message queue id: 0
# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x4364496c 0          root       644        0            0

# go run ipc.go
# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

# PID namespaces (CLONE_NEWPID)
### 介紹
隔離 process ID number 空間。就是同個 processes 在不同的 PID namespace 可以顯示不一樣的 PID。 PID namespace 可以讓 container 提供暫停/恢復 container 的 process 以及轉移 container 到另外的 host 上。PIDs 在 新的  PID namespace 開始都是使用 1。 (如果有在 docker 裡面使用 ps 會了解到這部分。)
  
### 實作流程
#### Step 1. 撰寫程式
使用  namespace API 的 clone() 來創建一個新的 process 並透過 CLONE_NEWPID flag 來創建 namespace 。創建好 namespace 之後，進入 sh (Go lang 範例程式)或 bash (C 範例程式) 環境。 (一樣是上面再改一點點，後面這部分就僅附上程式碼，不會再做說明。)  
  
- Go lang: [https://github.com/sufuf3/mygo-container/blob/master/namespace/pid.go](https://github.com/sufuf3/mygo-container/blob/master/namespace/pid.go) 
- C lang: [https://github.com/sufuf3/myc-container/blob/master/namespace/pid.c](https://github.com/sufuf3/myc-container/blob/master/namespace/pid.c)

#### Step 2. 執行程式
- Go lang: `sudo go run pid.go`
- C lang: `gcc pid.c -o pid.o` -> `sudo ./pid.o`

#### Step 3. 觀察
1\. 先來看看在 host 上的 pstree

```
─bash(5940)───sudo(18717)───go(18718)─┬─pid(18736)─┬─sh(18740)
                                      │            ├─{pid}(18738)
                                      │            └─{pid}(18739)
                                      ├─{go}(18719)
                                      ├─{go}(18720)
                                      ├─{go}(18721)
                                      ├─{go}(18726)
                                      ├─{go}(18737)
                                      └─{go}(18741)
```

2\. 就我們上面的 UTS namespace 所知，實際上在 host 的 PID 是 18736，不過當我們在程式的 sh 環境裡使用 `echo $$` 我們得到的 PID 為 1

```
$ sudo go run pid.go
# echo $$
1
```

> 想要在程式的 sh 環境中使用 ps 指令的話，就讓繼續往後進行囉！到這邊蠻興奮的，因為好像開始看到一點點 docker 的影子了呢！

# Mount namespaces (CLONE_NEWNS)
### 介紹
在每個 namespace instance 隔離 processes 的 list of mount points。透過 `/proc/[pid]/mounts`, `/proc/[pid]/mountinfo`, `/proc/[pid]/mountstats` 來提供 views。

### 實作流程
#### Step 1. 撰寫程式
一樣只改一點點部分。  
  
- Go lang: [https://github.com/sufuf3/mygo-container/blob/master/namespace/mount.go](https://github.com/sufuf3/mygo-container/blob/master/namespace/mount.go) 
- C lang: [https://github.com/sufuf3/myc-container/blob/master/namespace/mount.c](https://github.com/sufuf3/myc-container/blob/master/namespace/mount.c)

#### Step 2. 執行程式
- Go lang: `sudo go run mount.go`
- C lang: `gcc mount.c -o mount.o` -> `sudo ./mount.o`

#### Step 3. 觀察
1\. 第一次的 `ls /proc` 看到的是 host 的 /proc

```
$ sudo go run mount.go 
# ls /proc
1     1244   14030  18395  2      27   409   6022  92         driver       key-users    mtrr          sys
10    1245   1483   18396  20     276  410   6722  acpi       execdomains  kmsg         net           sysrq-trigger
1067  1250   15     18397  20008  28   411   7     buddyinfo  fb           kpagecgroup  pagetypeinfo  sysvipc
11    13     1580   18458  20026  32   412   76    bus        filesystems  kpagecount   partitions    thread-self
1104  1365   15915  19     2015   327  413   77    cgroups    fs           kpageflags   sched_debug   timer_list
1105  13788  16     19184  2016   328  414   78    cmdline    interrupts   loadavg      schedstat     tty
1111  13789  1645   19762  21     33   452   79    consoles   iomem        locks        scsi          uptime
112   13930  1660   19810  22     34   456   8     cpuinfo    ioports      mdstat       self          version
1130  1394   17     19811  23     4    5773  8259  crypto     irq          meminfo      slabinfo      version_signature
1155  14     18     19830  24     403  5939  85    devices    kallsyms     misc         softirqs      vmallocinfo
12    14025  18088  19833  25     406  5940  895   diskstats  kcore        modules      stat          vmstat
1228  14026  18304  19841  26     408  6     9     dma        keys         mounts       swaps         zoneinfo
```

2\. 為了看到 Mount namespace 的 /proc ，我們將 /proc mount 到 namespace 底下。然後再來觀察 `ls /proc`。這邊會發現只有顯示 namespace 底下的檔案。  

```
# mount -t proc proc /proc
# ls /proc
1          consoles   execdomains  irq          kpagecount  modules       schedstat  sys            version
4          cpuinfo    fb           kallsyms     kpageflags  mounts        scsi       sysrq-trigger  version_signature
acpi       crypto     filesystems  kcore        loadavg     mtrr          self       sysvipc        vmallocinfo
buddyinfo  devices    fs           keys         locks       net           slabinfo   thread-self    vmstat
bus        diskstats  interrupts   key-users    mdstat      pagetypeinfo  softirqs   timer_list     zoneinfo
cgroups    dma        iomem        kmsg         meminfo     partitions    stat       tty
cmdline    driver     ioports      kpagecgroup  misc        sched_debug   swaps      uptime
```

3\. 再來我們使用 `ps -ef` 就可以看到我們 namespace 裡面的 process 拉！

```
# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 10:20 pts/1    00:00:00 sh
root         5     1  0 10:21 pts/1    00:00:00 ps -ef
```

# User namespaces (CLONE_NEWUSER)
### 介紹
隔離 security-related identifiers 和 attributes，尤其是一個 process 的 user IDs 和 group IDs 是可以內外不同的。也就是在 host 上面可以用非 root 的使用者，但到 namespace 裡面該使用者可以在 namespace 成為 root 。

### 實作流程
#### Step 1. 撰寫程式
一樣改一點點的程式。  
  
- Go lang: [https://github.com/sufuf3/mygo-container/blob/master/namespace/user.go](https://github.com/sufuf3/mygo-container/blob/master/namespace/user.go) 
- C lang: [https://github.com/sufuf3/myc-container/blob/master/namespace/user.c](https://github.com/sufuf3/myc-container/blob/master/namespace/user.c)

#### Step 2. 執行程式
- Go lang: `sudo go run user.go`
- C lang: `gcc user.c -o user.o` -> `sudo ./user.o`

#### Step 3. 觀察
1\. 首先在本機查 user ID.

```
$ sudo id
uid=0(root) gid=0(root) groups=0(root)
```

2\. 然後執行程式進到 sh 裡面查看 id

```
$ sudo go run user.go 

$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

可以看到不同的 UID ，所以 User namespace 成功隔離了。  

# Network namespaces (CLONE_NEWNET)
### 介紹
隔離 host 本身的 networking 資源，如：network devices, IPv4 and IPv6 protocol stacks, IP routing tables, firewalls, the /proc/net directory, the /sys/class/net directory, port numbers (sockets) ....

### 實作流程
#### Step 1. 撰寫程式
一樣改一點點的程式。  
  
- Go lang: [https://github.com/sufuf3/mygo-container/blob/master/namespace/net.go](https://github.com/sufuf3/mygo-container/blob/master/namespace/net.go) 
- C lang: [https://github.com/sufuf3/myc-container/blob/master/namespace/net.c](https://github.com/sufuf3/myc-container/blob/master/namespace/net.c)

#### Step 2. 執行程式
- Go lang: `sudo go run net.go`
- C lang: `gcc net.c -o net.o` -> `sudo ./net.o`

#### Step 3. 觀察

```
$ sudo go run net.go

$ ifconfig
Warning: cannot open /proc/net/dev (No such file or directory). Limited output.
```

由上可知，原本 host 上的 ifconfig 有很多 Network 的資訊，但進到 namespace 裡面，什麼都沒看到，所以就是隔離網路了。  

Ref:  
1. [https://yq.aliyun.com/articles/64928](https://yq.aliyun.com/articles/64928)  
2. [https://lwn.net/Articles/531114/](https://lwn.net/Articles/531114/)  
3. [http://blog.lucode.net/linux/intro-Linux-namespace-1.html](http://blog.lucode.net/linux/intro-Linux-namespace-1.html)  

### 相關系列文：
[[Build container] 自己的 Container 自己寫](/series/my-own-container)
