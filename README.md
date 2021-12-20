# K8S

## 컨테이너 기반 기술

* 격리 기술

![컨테이너 역사 - CHROOT, Docker, KUBERNETES , OPENSHIFT](img/Container-innovation)



### chroot (root directory)

```sh
root@docker1:~# chroot --version
chroot (GNU coreutils) 8.28
Copyright (C) 2017 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by Roland McGrath.

root@docker1:~# mkdir -p ~/newroot/{bin,lib,lib64}
root@docker1:~# tree
.
├── docker_lab.tgz
├── k8s_lab.tgz
└── newroot
    ├── bin
    ├── lib
    └── lib64
    
root@docker1:~# ldd /bin/bash
        linux-vdso.so.1 (0x00007ffea30f1000)
        libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f368ea54000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f368e850000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f368e45f000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f368ef98000)

5 directories, 2 files
root@docker1:~# cp  /lib/x86_64-linux-gnu/libtinfo.so.5 ~/newroot/lib/x86_64-linux-gnu
root@docker1:~# cp  /lib/x86_64-linux-gnu/libdl.so.2  ~/newroot/lib/x86_64-linux-gnu
root@docker1:~# cp  /lib/x86_64-linux-gnu/libc.so.6  ~/newroot/lib/x86_64-linux-gnu
root@docker1:~# cp  /lib64/ld-linux-x86-64.so.2   ~/newroot/lib64    

root@docker1:~# chroot ~/newroot/ /bin/bash
bash-4.4# pwd
/

bash-4.4# echo  $$
13159

```

* 다른 창에서 열어 보면. bash 가 열려 있음을 알수 있음
* 완전히 분리된 것이 아니라 root / 만 분리한 것으로 되고 아직까지는 pid는 공유하고 있다. 

```
root@docker1:~# ps -ef | grep bash
root     13159  2115  0 05:23 pts/3    00:00:00 /bin/bash

```

* ls 명령도 라이브러리와 실행 파일 복사

```sh
root@docker1:~# ldd  /bin/ls
        linux-vdso.so.1 (0x00007ffcc2135000)
        libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007fe243349000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fe242f58000)
        libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007fe242ce6000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fe242ae2000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fe243793000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fe2428c3000)

root@docker1:~# cp  /lib/x86_64-linux-gnu/libselinux.so.1 ~/newroot/lib/x86_64-linux-gnu/
root@docker1:~# cp  /lib/x86_64-linux-gnu/libc.so.6 ~/newroot/lib/x86_64-linux-gnu/
root@docker1:~# cp  /lib/x86_64-linux-gnu/libpcre.so.3 ~/newroot/lib/x86_64-linux-gnu/
root@docker1:~# cp  /lib/x86_64-linux-gnu/libdl.so.2 ~/newroot/lib/x86_64-linux-gnu/
root@docker1:~# cp  /lib64/ld-linux-x86-64.so.2 ~/newroot/lib64/
```

### Namespace (격리)

#### 1. PID 격리

나만 보이는 프로세스 id를 가져 가겠다는 의미이군...

```sh
root@docker1:~# unshare --fork --pid --mount-proc=/proc /bin/sh
# ps
  PID TTY          TIME CMD
    1 pts/4    00:00:00 sh
    2 pts/4    00:00:00 ps
# ps axf
  PID TTY      STAT   TIME COMMAND
    1 pts/4    S      0:00 /bin/sh
    3 pts/4    R+     0:00 ps axf

# pstree
sh───pstree
```



#### 2. MNT  격리

나만 보이는 디렉토리를  mount 한다는 의미 이군. 

* 1차 콘솔

```sh
# mkdir /tmp/mnt_ns
# unshare -m /bin/bash

root@docker1:~# ll /proc/$$/ns/mnt
lrwxrwxrwx 1 root root 0 Dec 20 05:48 /proc/7/ns/mnt -> 'mnt:[4026532262]'

root@docker1:~# mount -n -t tmpfs tmpfs  /tmp/mnt_ns/

root@docker1:~# df
Filesystem                        1K-blocks    Used Available Use% Mounted on
...
tmpfs                               2019600       0   2019600   0% /tmp/mnt_ns

```

* 2차 콘솔

```sh
root@docker1:~# unshare -m /bin/bash
root@docker1:~# ll /proc/$$/ns/mnt
lrwxrwxrwx 1 root root 0 Dec 20 05:52 /proc/13402/ns/mnt -> 'mnt:[4026532263]'
root@docker1:~# df -hT | grep  mn
... 결과 없음

```



#### 3. UTS 격리

unix time sharing

* 1번 콘솔

```sh
# uname -a
Linux docker1 4.15.0-140-generic #144-Ubuntu SMP Fri Mar 19 14:12:35 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
# unshare -u /bin/bash
root@docker1:~#
root@docker1:~# ll /proc/$$/ns/uts
lrwxrwxrwx 1 root root 0 Dec 20 06:06 /proc/39/ns/uts -> 'uts:[4026532262]'
root@docker1:~# hostname happy.life.com
root@docker1:~# uname -n
happy.life.com
root@docker1:~# exit
exit
# uname -a
Linux docker1 4.15.0-140-generic #144-Ubuntu SMP Fri Mar 19 14:12:35 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

```

* 2번 콘솔

```sh
root@docker1:~# unshare -u /bin/bash
root@docker1:~# ll /proc/$$/ns/uts
lrwxrwxrwx 1 root root 0 Dec 20 06:09 /proc/13485/ns/uts -> 'uts:[4026532262]'
root@docker1:~# uname -n
docker1
```



#### 4. IPC 격리

공유 메모리 설정도 분리된다.



* 1번 콘솔

```sh
# unshare -i /bin/bash
root@docker1:~# ll /proc/$$/ns/ipc
lrwxrwxrwx 1 root root 0 Dec 20 06:13 /proc/55/ns/ipc -> 'ipc:[4026532262]'
root@docker1:~# ipcmk -M 100
Shared memory id: 0
root@docker1:~# ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0xda99bc90 0          root       644        100        0


```



* 2번 콘솔

```sh

root@docker1:~# unshare -i /bin/bash
root@docker1:~# ll /proc/$$/ns/ipc
lrwxrwxrwx 1 root root 0 Dec 20 06:14 /proc/13545/ns/ipc -> 'ipc:[4026532263]'
root@docker1:~# ipcmk -M 100
Shared memory id: 0
root@docker1:~# ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x96896f1f 0          root       644        100        0

root@docker1:~# exit
exit
root@docker1:~# ll -l /proc/$$/ns/ipc
lrwxrwxrwx 1 root root 0 Dec 20 05:34 /proc/4460/ns/ipc -> 'ipc:[4026531839]'
root@docker1:~# ipcmk -M 100
Shared memory id: 1343500
root@docker1:~# ipcs -m

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x00000000 131072     worker1    600        16384      1          dest
...
0x00000000 1310731    worker1    600        524288     2          dest
0xab0b735d 1343500    root       644        100        0

```



#### 5.  USER  격리

```sh
worker1@docker1:~$ unshare --map-root-user --user whoami
root
worker1@docker1:~$ whoami
worker1

```



* 2번 콘솔

```sh

root@docker1:~# su - worker1
worker1@docker1:~$ whoami
worker1
worker1@docker1:~$ unshare --m <tab>
--map-root-user  --mount          --mount-proc
worker1@docker1:~$ unshare --map-root-user --user /bin/bash
root@docker1:~# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@docker1:~# pwd
/home/worker1
```



#### 6. NET 격리

* network namespace로 분리해서 Network 리소스를 격리한다는 것
* v-eht1 ~ v-peer1 peer 관계 설정

![img](img/linux-network-namespace.png)

```sh
# ip netns add guestnet
# ip netns exec guestnet ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# ip netns exec guestnet ip link set lo up
# ip netns exec guestnet ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:5a:a9:d9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.137.101/24 brd 192.168.137.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5a:a9d9/64 scope link
       valid_lft forever preferred_lft forever


# ip link add host type veth peer name guest
# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:5a:a9:d9 brd ff:ff:ff:ff:ff:ff
3: guest@host: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2a:3b:2b:1e:71:e1 brd ff:ff:ff:ff:ff:ff
4: host@guest: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2e:6b:2c:c8:93:15 brd ff:ff:ff:ff:ff:ff
# ethtool -S host
NIC statistics:
     peer_ifindex: 3
# ip link set guest netns guestnet
# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:5a:a9:d9 brd ff:ff:ff:ff:ff:ff
4: host@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2e:6b:2c:c8:93:15 brd ff:ff:ff:ff:ff:ff link-netnsid 0
# ip netns exec guestnet ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: guest@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 2a:3b:2b:1e:71:e1 brd ff:ff:ff:ff:ff:ff link-netnsid 0
#
# ip link set host up
# ip address add 1.1.1.1/24 dev host
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:5a:a9:d9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.137.101/24 brd 192.168.137.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5a:a9d9/64 scope link
       valid_lft forever preferred_lft forever
4: host@if3: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether 2e:6b:2c:c8:93:15 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 1.1.1.1/24 scope global host
       valid_lft forever preferred_lft forever
# ip netns exec guestnet ip link set guest up
# ip netns exec guestnet ip address add 1.1.1.2/24 dev guest
# ip netns exec guestnet ip add show dev guest
3: guest@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 2a:3b:2b:1e:71:e1 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 1.1.1.2/24 scope global guest
       valid_lft forever preferred_lft forever
    inet6 fe80::283b:2bff:fe1e:71e1/64 scope link
       valid_lft forever preferred_lft forever
#
# ping 1.1.1.2
PING 1.1.1.2 (1.1.1.2) 56(84) bytes of data.
64 bytes from 1.1.1.2: icmp_seq=1 ttl=64 time=0.031 ms
^C
--- 1.1.1.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.031/0.031/0.031/0.000 ms
#
# ip netns exec guestnet ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=64 time=0.017 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=64 time=0.054 ms
^C
--- 1.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1023ms
rtt min/avg/max/mdev = 0.017/0.035/0.054/0.019 ms

```

### cgroup (control group)

* cgroup은 단일 프로세스 또는 task에 대해서 cpu, memory, network, storage 자원 할당 및 제어 하는 커널 기능
* VFS의 디렉토리 표시되는 계층 구조로 group 일부 기능은  부모 속성을 상속
* 일반 파일 시스템은 단일 트리 구조 이지만 ,  cgroup 계층 구조는  /sys/fs/cgroup 이외  임의의 계층 구조를 추가 가능

#### /proc/cgroup

```sh
# cat  /proc/cgroups
#subsys_name    hierarchy       num_cgroups     enabled
cpuset  7       1       1
cpu     8       89      1
cpuacct 8       89      1
blkio   2       81      1
memory  3       130     1
devices 5       81      1
freezer 11      1       1
net_cls 4       1       1
perf_event      9       1       1
net_prio        4       1       1
hugetlb 6       1       1
pids    10      97      1
rdma    12      1       1

# mount -t cgroup  -o  cpu,cpuacct cpu,cpuacct /cgroup/cpu
# ls -l /cgroup/cpu
total 0
-rw-r--r--  1 root root 0 Dec 20 04:15 cgroup.clone_children
-rw-r--r--  1 root root 0 Dec 20 07:16 cgroup.procs
-r--r--r--  1 root root 0 Dec 20 04:15 cgroup.sane_behavior
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.stat
-rw-r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage_all
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage_percpu
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage_percpu_sys
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage_percpu_user
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage_sys
-r--r--r--  1 root root 0 Dec 20 04:15 cpuacct.usage_user
-rw-r--r--  1 root root 0 Dec 20 04:15 cpu.cfs_period_us
-rw-r--r--  1 root root 0 Dec 20 04:15 cpu.cfs_quota_us
-rw-r--r--  1 root root 0 Dec 20 04:15 cpu.shares
-r--r--r--  1 root root 0 Dec 20 04:15 cpu.stat
-rw-r--r--  1 root root 0 Dec 20 04:15 notify_on_release
-rw-r--r--  1 root root 0 Dec 20 04:15 release_agent
drwxr-xr-x 80 root root 0 Dec 20 04:10 system.slice
-rw-r--r--  1 root root 0 Dec 20 04:15 tasks
drwxr-xr-x  5 root root 0 Dec 20 04:10 user.slice

root@docker1:# mkdir -p /cgroup/cpu/limit_70_percent
root@docker1:# echo 700 > /cgroup/cpu/limit_70_percent/cpu.shares
root@docker1:# mkdir -p /cgroup/cpu/limit_30_percent
root@docker1:# echo 300 > /cgroup/cpu/limit_30_percent/cpu.shares

root@docker1:/cgroup/cpu/limit_70_percent# echo  $(pgrep a.out) > tasks
root@docker1:/cgroup/cpu/limit_30_percent# echo  $(pgrep b.out) > tasks
```



* cpu 2라서 a.out, b.out이 추가적으로  cpu 1번에 a.out, a.out 두개가 할당 되면  7:3 으로 할당 되지 않는다.
* 정확하게 하려면 CPU를 1개로 설정한 다음 테스트 하면  7:3 으로 cpu 사용 시간이 할당된다.  

```sh
root@docker1:~# top
top - 07:41:07 up  3:30,  3 users,  load average: 4.03, 2.21, 1.01
Tasks: 227 total,   7 running, 173 sleeping,   0 stopped,   0 zombie
%Cpu(s):100.0 us,  0.0 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0
KiB Mem :  4039204 total,   185844 free,  1095308 used,  2758052 buff/cache
KiB Swap:  2017276 total,  2017008 free,      268 used.  2644476 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
14157 root      20   0    4384    728    664 R  73.2  0.0   6:39.78 a.out
14158 root      20   0    4384    716    652 R  28.1  0.0   6:00.62 b.out
14180 root      20   0    4384    716    652 R  26.5  0.0   0:03.47 b.out
14179 root      20   0    4384    812    748 R  24.2  0.0   0:03.12 a.out
14176 root      20   0    4384    752    688 R  23.8  0.0   0:43.95 a.out
14177 root      20   0    4384    760    696 R  23.8  0.0   0:44.00 b.out


```



### iptables

- iptables 은 docker에서 외부 네트워크와 통신하는데 중요한 역할
- packet 필터링 기능도 있어서 방화벽 기능도 구현 가능, 주로 패킷 필터링과  NAT 기능을 수행한다. 



![](img/iptables.jpeg)





![](D:\Code\k8s\img\Netfilter-packet-flow.svg)







## File system 

### union File system 

#### AUFS



#### OverlayFS

* 이미지 는 Read Only 이고 절대로 변경이 안된다. 
* 변경 되는 것은 Container의 RW layer 만 변경 된다. 

![OverlayFS as a Docker Storage Driver](img/overlayfs-as-a-docker-storage-driver-6-638.jpg)





#### AUFS, overlayfs

![](img/layerfilesystem.JPG)



```sh
root@docker1:~# mkfs overlayfs
mke2fs 1.44.1 (24-Mar-2018)
The file overlayfs does not exist and no size was specified.
root@docker1:~# mkdir overlayfs
root@docker1:~# cd overlayfs/
root@docker1:~/overlayfs# mkdir container image1 image2 image3 work merge
root@docker1:~/overlayfs# touch image1/{a,b} image2/c
root@docker1:~/overlayfs# mount -t  overlay overlay -o lowerdir=image2:image1,upperdir=container,workerdir=work merge
mount: /root/overlayfs/merge: wrong fs type, bad option, bad superblock on overlay, missing codepage or helper program, or other error.
root@docker1:~/overlayfs# mount -t  overlay overlay -o lowerdir=image2:image1,upperdir=container,workdir=work merge
root@docker1:~/overlayfs#


root@docker1:~/overlayfs# df
Filesystem                        1K-blocks    Used Available Use% Mounted on
overlay                            19475088 7312464  11150300  40% /root/overlayfs/merge

root@docker1:~/overlayfs# tree . -l work
.
├── container
├── image1
│   ├── a
│   └── b
├── image2
│   └── c
├── image3
├── merge
│   ├── a
│   ├── b
│   └── c
└── work
    └── work
work
└── work
<파일 생성 작업>
root@docker1:~/overlayfs/merge# tree .
.
├── a
├── b
├── c
└── jhyunlee
    └── okman


```

* overlay file system

```sh
root@docker1:~/overlayfs# tree
.
├── container
│   └── jhyunlee
│       └── okman
├── image1
│   ├── a
│   └── b
├── image2
│   └── c
├── image3
├── merge
│   ├── a
│   ├── b
│   ├── c
│   └── jhyunlee
│       └── okman
└── work
    └── work

```

* file작업
  *  파일 변경 하더라도 image layer는 변경이 없다.
  * 파일 변경이 발생하면 container 계층에 변경된 내용만 등록 된다.
  * CoW (Copy on Write) 개념으로 동작한다. 
  *  `mount -t  overlay overlay -o lowerdir=image2:image1,upperdir=container,workdir=work merge`

```sh
root@docker1:~/overlayfs# rm  ./merge/a
root@docker1:~/overlayfs# touch ./merge/d
root@docker1:~/overlayfs# tree
.
├── container  <<=== 수정된 내용은 여기에 등록
│   ├── a
│   ├── d
│   └── jhyunlee
│       └── okman
├── image1  <<== image1 계층은 변경 없음
│   ├── a
│   └── b
├── image2 <<== image2 계층은 변경 없음
│   └── c
├── image3 <<== image3 계층은 변경 없음
├── merge  <<==== 변경된 최종은 여기에 merge 되어 관리됨.  
│   ├── b
│   ├── c
│   ├── d
│   └── jhyunlee
│       └── okman
└── work
    └── work

```

