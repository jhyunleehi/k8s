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

#### 1. iptable 개념

iptables는 리눅스상에서 방화벽을 설정하는 도구로서 커널 2.4 이전 버전에서 사용되던 ipchains를 대신하는 방화벽 도구이다. iptables는 커널상에서의 netfilter 패킷필터링 기능을 사용자 공간에서 제어하는 수준으로 사용할 수 있다. 

**패킷필터링이란** 지나가는 패킷의 해더를 보고 그 전체 패킷의 운명을 결정하는 것을 말한다. 일반적으로 패킷은 해더와 데이터를 가진다.
해더에 필터링할 정보인 출발지IP:PORT, 도착지IP:PORT, checksum, 프로토콜 옵션등을 가지며 데이터는 각각의 전송데이터가 들어간다. 

특정 조건을 가지고 있는 패킷에 대해 **허용(ACCEPT)**과 **차단(DROP)**등을 지정할 수 있으며, 특정 조건등을 통해 다양한 방식의 패킷 필터링과 처리 방식을 지원한다.

iptables 정책은 여러 구분으로 나눠지며 중요한 부분은 Chain이다.
Chain은 패킷이 조작될 상태를 지정하며 iptables에 내장된 기본 Chain은 다음과 같다. 
(기본 Chain은 영구적이며 삭제가 불가능하다. 이외에 -N 옵션으로 지정하는 사용자 정의 Chain이 있다.) 

- Chain INPUT : 서버로 들어오는 기본 정책

- Chain FORWARD : 서버에서 forwarding 기본 정책

- Chain OUTPUT : 서버에서 나가는 기본 정책

  

- Linux Server를 목적지로 삼는 모든 패킷은 **INPUT Chain**을 통과하고 Linux Server에서 생성되 외부로 보내지는 모든 패킷은 **OUTPUT Chain**을 통과하게 된다.**FORWARD Chain**의 경우 현재의 Linux Server가 목적지가 아닌 패킷이 통과하는 Chain이다. (FORWARD Chain은 NAT(네트워크 공유) 기능 사용을 위해 사용된다.) 

  - 마스커레이드(Masquerade) :  내부 사설 IP의 PC들이 외부 인터넷이 연결 가능하도록 해주는 기능
  - NAT(Network Address Translation) : 네트워크 주소 변환 서비스
    - SNAT(Source NAT) **:** 내부 사설IP에서 외부로 나갈 때 공인IP로 변환 -> 마스커레이드와 비슷
    - DNAT(Destination NAT) : 외부에서 방화벽(외부IP)으로 요청되는 주소로 내부사설IP로 변환



#### 2. iptable 구조

```
iptables -A INPUT -s [발신지] --sport [발신지 포트] -d [목적지] --dport [목적지 포트] -j [정책] 

1) 기본정책을 ACCEPT로 설정하는 법 :
 # iptables -P INPUT ACCEPT
 # iptables -P OUTPUT ACCEPT
 # iptables -P FORWARD ACCEPT

2) 현재 자신의 방화벽 규칙을 볼 수 있는 명령 :
 # iptables --list 또는 iptables -L

3) 규칙 추가 후에 저장하기 :
 # service iptables save 
  -> /etc/sysconfig/iptables 에 저장됨

4) 현재 iptables 규칙을 초기화 하기 :
 # iptables -F
```





#### 3. 규칙 등록

![img](img/2773BA3354A9D63327)

```
# iptables -A INPUT -s 127.0.0.1 -p icmp -j DROP
# iptables -D INPUT -s [발신지] --sport [발신지 포트] -d [목적지] --dport [목적지 포트] -j [정책]
```



```
1) 소스 ip가 192.168.0.111 인 접속의 모든 접속 포트를 막아라.
    # iptables -A INPUT -s 192.168.0.111 -j DROP

2) INPUT 사슬에 출발지 주소가 127.0.0.1(-s 127.0.0.1) 인 icmp 프로토콜(-p icmp) 패킷을 거부(-j DROP)하는 정책을 추가(-A)하라
    # iptables -A INPUT -p icmp -s 127.0.0.1 -j DROP

3) INPUT 사슬에 목적지 포트가 23번(--dport23)인 tcp 프로토콜(-p tcp) 패킷을 거부하는(-j DROP)규칙을 추가(-A) 하라.
    # iptables -A INPUT -p tcp --dport 23 -j DROP

4) INPUT 사슬에 목적지 포트 번호가 80번(--dport 80)인 tcp 프로토콜(-p tcp)패킷을 받아들이는(-j ACCEPT) 규칙을 추가(-A) 하라
    # iptables -A INPUT -p tcp --dport 80 -j ACCEPT

5) INPUT 사슬에 목적지 포트번호가 1023번 보다 작은 모든 포트(--dport :1023)인 tcp프로토콜(-p tcp)패킷을 거부하는(-j DROP)규칙을 추가(-A)하라
    # iptables -A INPUT -p tcp --dport :1023 -j DROP

6) ftp포트를 열어라
    # iptables -I INPUT -p tcp --dport 21 -j ACCEPT

7) imap 서비스를 방화벽에서 열어라
    # iptables -I INPUT -s 192.168.0.0/255.255.255.0 -p udp --dport 143 -j ACCEPT

8) 웹서버 방화벽 열어라
    # iptables -I INPUT -p tcp --dport 80 -j ACCEPT

9) 웹서버 포트 80 -> 8880으로 교체하라( 웹서비스 포트 변경시 /etc/services 에서도 변경 해줘야 함)
   # iptables -R INPUT 2 -p tcp --dport 8880 -j ACCEPT

10) domain-access_log 파일에 있는 모든 ip의 모든 접속 포트를 막아라(DOS공격 방어시 사용)
   # cat domain-access_log |awk '{print $1}'|sort |uniq |awk '{print "iptables -A INPUT -s "$1" -j DROP"}'|/bin/bash

```







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





## Docker 

### docker install

#### 1. netplan 이용한 IP 설정

* 우리는 Network Manager를 이용하지 않고  Netplan을 이용하여 고정  IP 설정한다. 
* `# netplan apply`

```sh
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.137.101/24]
      gateway4: 192.168.137.1
      nameservers:
        addresses: [168.126.63.1,168.126.63.2]
```



#### 2. install docker

```sh
## docker installation
1) # sudo apt-get update

2) # sudo apt-get install -y  apt-transport-https ca-certificates curl software-properties-common
https를 사용해서 레포지토리를 사용할 수 있도록 필요한 패키지를 설치한다. 

3) # curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
Docker 공식 리포지토리에서 패키지를 다운로드 받았을 때 위변조 확인을 위한 GPG 키를 추가한다.

4) # apt-key fingerprint     
Docker.com 의 GPG 키가 등록됐는지 확인한다. 

5) # add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
Docker 공식 저장소를 리포지토리로 등록한다.

6) # grep docker /etc/apt/sources.list
deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
저장소 등록정보에 기록됐는지 확인한다. 

7) # apt-get update    
리포지토리 정보를 갱신

8) # apt-get install -y docker-ce
docker container engine 을 설치한다.

```



```sh
root@docker1:~# docker version
Client: Docker Engine - Community
 Version:           20.10.12
 API version:       1.41
 Go version:        go1.16.12
 Git commit:        e91ed57
 Built:             Mon Dec 13 11:45:27 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.12
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.16.12
  Git commit:       459d0df
  Built:            Mon Dec 13 11:43:36 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.4.12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          1.0.2
  GitCommit:        v1.0.2-0-g52b36a2
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

```



```sh
root@docker1:~# uname -r
4.15.0-163-generic
root@docker1:~# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Docker Buildx (Docker Inc., v0.7.1-docker)
  scan: Docker Scan (Docker Inc., v0.12.0)

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 20.10.12
 Storage Driver: overlay2    <<=== 
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 io.containerd.runtime.v1.linux runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc version: v1.0.2-0-g52b36a2
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: default
 Kernel Version: 4.15.0-163-generic
 Operating System: Ubuntu 18.04.5 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.852GiB
 Name: docker1
 ID: DDWF:MZJE:PGJA:HQIT:TVY4:JKNX:A7E4:FBEB:S3AM:4DKW:6EXW:4AK7
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/   <=== docker hub를 의미 
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

WARNING: No swap limit support

```

#### 3. docker system

```sh
root@docker1:/proc# docker system

Usage:  docker system COMMAND

Manage Docker

Commands:
  df          Show docker disk usage
  events      Get real time events from the server
  info        Display system-wide information
  prune       Remove unused data
```



```sh
root@docker1:/proc# docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          0         0         0B        0B
Containers      0         0         0B        0B
Local Volumes   0         0         0B        0B
Build Cache     0         0         0B        0B
```

#### 4. docker image

```sh
root@docker1:/proc# docker image

Usage:  docker image COMMAND

Manage images

Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
```



### Lifecycle od Docker Container

```sh
root@docker1:/proc# docker container

Usage:  docker container COMMAND

Manage containers

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  inspect     Display detailed information on one or more containers
  kill        Kill one or more running containers
  logs        Fetch the logs of a container
  ls          List containers
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  prune       Remove all stopped containers
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  run         Run a command in a new container
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  wait        Block until one or more containers stop, then print their exit codes
```





![img](img/docker_state.png)



```sh
docker pause <container-id/name>
docker unpause <container-id/name>
docker start <container-id/name>
docker stop <container-id/name>
docker restart <container-id/name>
docker kill <container-id/name>
docker rm <container-id/name>
```

* https://index.docker.io/v1/ 공식 github 에 올라와 있는 이미지로 생성 

```sh
root@docker1:/proc# docker create -it --name=jhyunlee ubuntu
Unable to find image 'ubuntu:latest' locally   <<=== local에 이미지 없으면 다운로드 받는다. 
latest: Pulling from library/ubuntu
7b1a6ab2e44d: Pull complete
Digest: sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322
Status: Downloaded newer image for ubuntu:latest
1395cd35553301a34e42991f9bacee286d709f20c1443144fd110f0faba5eafa
root@docker1:/proc# docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED         STATUS    PORTS     NAMES
1395cd355533   ubuntu    "bash"    8 seconds ago   Created             jhyunlee
```



```sh
docker pause <container-id/name>
docker unpause <container-id/name>
docker start <container-id/name>
docker stop <container-id/name>
docker restart <container-id/name>
docker kill <container-id/name>
docker rm <container-id/name>
```



*  docker create centos

```sh
root@docker1:/proc# docker container create -it --name testos centos
Unable to find image 'centos:latest' locally
latest: Pulling from library/centos
a1d0c7532777: Pull complete
Digest: sha256:a27fd8080b517143cbbbab9dfb7c8571c40d67d534bbdee55bd6c473f432b177
Status: Downloaded newer image for centos:latest
69cddbdf5b9c4bfef74f39737b5c241b85769a08e300dc747e67e56146237ec5
root@docker1:/proc# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS    PORTS     NAMES
69cddbdf5b9c   centos    "/bin/bash"   6 seconds ago    Created             testos
1395cd355533   ubuntu    "bash"        21 minutes ago   Created             jhyunlee
```



#### docker operation

- docker images (-a)
- docker create ...
- docker ps (-a)
- docker run ..
- docker logs 'container_name'
- docker bash에서 빠져 나올때는 CTL+PQ 로 빠져 나와야 한다.  



#### 1. docker run

```sh
root@docker1:/proc# docker container create -it --name=yumi00 centos
0d12c010838780603361f5e394f64ed6a8f954aaf9d13508c3b5999bea8d431d
root@docker1:/proc# docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS    PORTS     NAMES
0d12c0108387   centos    "/bin/bash"   9 seconds ago    Created             yumi00
69cddbdf5b9c   centos    "/bin/bash"   12 minutes ago   Created             testos
1395cd355533   ubuntu    "bash"        34 minutes ago   Created             jhyunlee

root@docker1:/proc# docker container start -ai yumi00
[root@0d12c0108387 /]# pwd
/
CTL-PQ  백그라운드로 나오기

root@docker1:/proc# docker run -it --name yumi02 centos
root@docker1:/proc# docker run -it --name yumi03 centos /bin/date

<docker yumi03은 Exited 된다>
Every 0.5s: docker ps -a
CONTAINER ID   IMAGE     COMMAND       CREATED          STATUS                      PORTS     NAMES
52acf12d4901   centos    "/bin/date"   56 seconds ago   Exited (0) 55 seconds ago             yumi03
53810af77fd7   centos    "/bin/bash"   3 minutes ago    Up 3 minutes                          yumi02
0d12c0108387   centos    "/bin/bash"   2 hours ago      Up 2 hours                            yumi00
69cddbdf5b9c   centos    "/bin/bash"   2 hours ago      Created                               testos
1395cd355533   ubuntu    "bash"        2 hours ago      Created                               jhyunlee

```

* 격리 환경 확인 : PID 분리

```sh
[root@0d12c0108387 /]# ps -e -o ppid,pid,sz,rss,comm
 PPID   PID    SZ   RSS COMMAND
    0     1  3012  3292 bash  <<=== pid 가 1번이다. 
    1    22 10635  2012 ps
[root@0d12c0108387 /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

```



```sh
# mount 
overlay on /var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/CTOCGPDCAQ5RTUQQGW3N6W6QXN:/var/lib/docker/overlay2/l/IOKVNDAHHUP6K2D57QGSTSZFYP,upperdir=/var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446/diff,workdir=/var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446/work)
```

* overlay2  파일 시스템

```sh
root@docker1:~# docker container inspect yumi00 | grep -C2 Upper
                "LowerDir": "/var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446-init/diff:/var/lib/docker/overlay2/0c6c5c7261012598557885a41025c1a2d83062aad14facb410d4974bd4fa3a71/diff",
                "MergedDir": "/var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446/merged",
                "UpperDir": "/var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446/diff",
                "WorkDir": "/var/lib/docker/overlay2/5d86594652bedf74d15cae43df3767dbd70648038487e75db857bbd97e05b446/work"
            },
```



* 재미 있는 것이 기존 콘솔에 ...bash 기동해 놓고 다른 창에서 docker attch 하니 명령 화면 내용이 완전히 공유 되어 버리네...



#### 2. docker exec 

```sh
root@docker1:/proc# docker container exec  -it yumi04 /bin/bash
[root@147b373ec411 /]#
[root@147b373ec411 /]# ps -e -o pid,sz,rss,comm
  PID    SZ   RSS COMMAND
    1  7917  3600 ping   <<=== 기존에 run으로 돌고 있는 process
    6  3012  3288 bash   <<== exec 로 접속한 bash 
   22 10635  2200 ps
[root@147b373ec411 /]# exit
exit
```

* 여기서 중요한 것은 추가로 접속한 bash에서 빠져 나와도 docker 컨테이너는 죽지는 않는다.
* 왜냐 하면 기존에 ping process가 UP 되어 있기 때문이다.
* 왠만하면 `docker exec  -it <name> /bin/bash`  로 접속하면 편리하다.
* `docker attach  <name>` 으로 접속하는 것 보다는 좀더 덜 스트레스 받는 다. 빠져 나올때 container 상태가 exit되지 않기 때문에...
* 





#### 3. docker rm

* `docker rm -f  <name>`
* `docker ps -a -q`
* `docker rm -f $(docker ps -a -q)`

```sh
root@docker1:~# docker container rm  yumi04
Error response from daemon: You cannot remove a running container 147b373ec411d3bf8bfe3ece0dbbbc9548b18d9be199ada8d1b327b1964be367. Stop the container before attempting removal or force remove

root@docker1:~# docker container rm  -f yumi04
yumi04
```



```sh
root@docker1:~# docker rm -f $(docker ps -a -q)
53810af77fd7
0d12c0108387
```



#### 4. docker cp

```sh
root@docker1:~# docker run  -itd --name test00 centos
83f5123677b9de018fd8c25c13f60ee9499fe10b93c2a245886660e26145f525
root@docker1:~# ls -l /etc/hosts
-rw-r--r-- 1 root root 204 Jun 23 14:22 /etc/hosts
root@docker1:~# docker container cp /etc/hosts test00:/containerfile
root@docker1:~# docker container exec  test00 /bin/ls -l  containerfile
-rw-r--r-- 1 root root 204 Jun 23 14:22 containerfile
root@docker1:~# docker container exec -it test00 /bin/bash
[root@83f5123677b9 /]# ls
bin  containerfile  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
[root@83f5123677b9 /]# ls -l containerfile
-rw-r--r-- 1 root root 204 Jun 23 14:22 containerfile
root@docker1:~# docker container cp test00:/containerfile  containerfile
```



#### 5. docker diff

* layerfile system은 새롭게 생성된 것도 기록하고 변경된 것도 기록하고 삭제된 것도 기록한다.
* 그런데 이렇게 변경된 내용은 영속적으로 저장되지 않는다는 것을 알아야 한다. 

```
root@docker1:~# docker container run  -it --name  test00 centos
CTL+PQ
root@docker1:~# docker container run  -it --name  test01 centos
root@docker1:~# docker container exec -it test01 /bin/bash
[root@52f3fda0193d /]# useradd containeruser
[root@52f3fda0193d /]# read escape sequence
root@docker1:~# docker container diff test01
C /home
A /home/containeruser
A /home/containeruser/.bash_logout
A /home/containeruser/.bash_profile
A /home/containeruser/.bashrc
C /var
C /var/log
C /var/log/lastlog
C /var/spool
C /var/spool/mail
A /var/spool/mail/containeruser
C /etc
C /etc/gshadow-
C /etc/shadow
C /etc/passwd-
A /etc/subgid-
C /etc/passwd
C /etc/group
C /etc/subuid
C /etc/group-
C /etc/gshadow
C /etc/subgid
C /etc/shadow-
A /etc/subuid-

```

#### 6. docker inspect

```
root@docker1:~# docker container inspect test00
[
    {
        "Id": "83f5123677b9de018fd8c25c13f60ee9499fe10b93c2a245886660e26145f525",
        "Created": "2021-12-21T05:18:34.081724475Z",
        "Path": "/bin/bash",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 31864,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2021-12-21T05:18:34.525486958Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:5d0da3dc976460b72c77d94c8a1ad045d4847e53fadb6",
        "ResolvConfPath": "/var/lib/docker/containers/83f5123677b8.../resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/83f5123677b9de..../hostname",
        "HostsPath": "/var/lib/docker/containers/83f5123677b9de018...../hosts",
        "LogPath": "/var/lib/docker/containers/83f5123../83f51....-json.log",
        "Name": "/test00",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "docker-default",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "host",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "KernelMemory": 0,
            "KernelMemoryTCP": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefinit/diff:/var/lib/docker/overlay2/0c6c5c7261012598557885a41025c1a2d83062aad14facb410d4974bd4fa3a71/diff",
                "MergedDir": "/var/lib/docker/overlay2/768d8d...../merged",
                "UpperDir": "/var/lib/docker/overlay2/768d8d2...../diff",
                "WorkDir": "/var/lib/docker/overlay2/768d8d29...../work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "83f5123677b9",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": true,
            "OpenStdin": true,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/bash"
            ],
            "Image": "centos",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "org.label-schema.build-date": "20210915",
                "org.label-schema.license": "GPLv2",
                "org.label-schema.name": "CentOS Base Image",
                "org.label-schema.schema-version": "1.0",
                "org.label-schema.vendor": "CentOS"
            }
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "264c432f8474255b46b154ca274f3ada7f49627e13865418a7f711ba20af2f00",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {},
            "SandboxKey": "/var/run/docker/netns/264c432f8474",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "6c5360887b4b42c1fae9d4855992b15f7ef4bbe1dcfbd67fffc05153cffc8f6a",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:02",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "5025f3fa9b38bfa51ef1f7d20dd2dcb3d430add6c34f1fc5a8862042a2856cbd",
                    "EndpointID": "6c5360887b4b42c1fae9d4855992b15f7ef4bbe1dcfbd67fffc05153cffc8f6a",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]

```



#### 7. container 위치

* layer2 파일 시스템을 사용하기 때문에... 이것을 LowerDir와 UppperDir를 잘 분리해서 생각해야 한다. 
* 실제 제공되는 파일 시스템은 MergedDir이 해당된다. 

```
root@docker1:/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/diff# touch AAAAAAfile
root@docker1:/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/diff# docker container exec -it test00 /bin/bash
[root@83f5123677b9 /]# ls
AAAAAAfile  containerfile  etc   lib    lost+found  mnt  proc  run   srv  tmp  var
bin         dev            home  lib64  media       opt  root  sbin  sys  usr

root@docker1:~# docker container inspect test00
    "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857-init/diff:/var/lib/docker/overlay2/0c6c5c7261012598557885a41025c1a2d83062aad14facb410d4974bd4fa3a71/diff",
                "MergedDir": "/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/merged",
                "UpperDir": "/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/diff",
                "WorkDir": "/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/work"
            },
            "Name": "overlay2"

```

* mount  된 파일  시스템은 merged 된 파일 시스템을 나타내게 된다. 

```
root@docker1:/var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/diff# df
Filesystem                        1K-blocks    Used Available Use% Mounted on
udev                                1985004       0   1985004   0% /dev
/dev/mapper/ubuntu--vg-ubuntu--lv  19475088 7606716  10856048  42% /
/dev/sda2                            999320  160100    770408  18% /boot
overlay                            19475088 7606716  10856048  42% /var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/merged
overlay                            19475088 7606716  10856048  42% /var/lib/docker/overlay2/bdae5423ea79b8a3589ba30c6bc3894b533b22f1ca185ecb078d30126de8932d/merged

```



## Docker volume

### bind mount

```sh
root@docker1:/var/lib/docker/overlay2# mkdir /tmp/hostdir
root@docker1:/var/lib/docker/overlay2# mkdir /tmp/readonly
root@docker1:/var/lib/docker/overlay2# echo writeable > /tmp/hostdir/file1
root@docker1:/var/lib/docker/overlay2# echo readonly > /tmp/readonly/file2
root@docker1:/var/lib/docker/overlay2# docker container run -it -v /tmp/hostdir:/container_dir -v /tmp/readonly:/read_dir:ro --name=test02 centos
[root@ce8f018a15b6 /]# ls -l
... 디렉토리 확인....
[root@ce8f018a15b6 /]# cat /container_dir/file1
writeable
[root@ce8f018a15b6 /]# cat /read_dir/file2
readonly
[root@ce8f018a15b6 /]# echo  hello > /container_dir/file1
[root@ce8f018a15b6 /]# echo  hello > /read_dir/file2
bash: /read_dir/file2: Read-only file system
CTL+PQ


root@docker1:/var/lib/docker/overlay2# docker container inspect test02 | grep  -A 20 "Mount"
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "docker-default",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": [
                "/tmp/hostdir:/container_dir", <<=====
                "/tmp/readonly:/read_dir:ro" <<====
            ],
--
        "Mounts": [  <<========
            {
                "Type": "bind",
                "Source": "/tmp/hostdir",
                "Destination": "/container_dir",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/tmp/readonly",
                "Destination": "/read_dir",
                "Mode": "ro",
                "RW": false,
                "Propagation": "rprivate"
            }
        ],
```

* df에 대한 변경은 없다. 이것을 보면 컨테이너의 Overlay 파일 시스템에 대한 것만 df에서 보이는 것 같고... Container 내부에서 bind  mount 한 것은 보이지 않는 것 같다. 

```
root@docker1:~# df
Filesystem                        1K-blocks    Used Available Use% Mounted on
udev                                1985004       0   1985004   0% /dev
tmpfs                                403920    1552    402368   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv  19475088 7606928  10855836  42% /
tmpfs                               2019588       0   2019588   0% /dev/shm
tmpfs                                  5120       4      5116   1% /run/lock
tmpfs                               2019588       0   2019588   0% /sys/fs/cgroup
/dev/sda2                            999320  160100    770408  18% /boot
tmpfs                                403916      28    403888   1% /run/user/125
tmpfs                                403916       0    403916   0% /run/user/0
overlay                            19475088 7606928  10855836  42% /var/lib/docker/overlay2/768d8d29138c57daa7e99dfefab120b10a90c2af83b14412ca40c6bb3d267857/merged
overlay                            19475088 7606928  10855836  42% /var/lib/docker/overlay2/bdae5423ea79b8a3589ba30c6bc3894b533b22f1ca185ecb078d30126de8932d/merged
overlay                            19475088 7606928  10855836  42% /var/lib/docker/overlay2/94ec1d74ebe44d47fb47e0f303423e0ba499b2278248f9c3a644c6e910dd2d73/merged

```





* docker 모두 삭제

```
# docker container rm  -f $(docker ps -a -q)
```



### docker volume 

#### 1. 자동 생성방법 volume

```
root@docker1:~# docker container  run -it --name  test00 -v /volume centos


[root@2fb829e5aeaa /]# df
Filesystem                        1K-blocks    Used Available Use% Mounted on
overlay                            19475088 7606568  10856196  42% /
/dev/mapper/ubuntu--vg-ubuntu--lv  19475088 7606568  10856196  42% /volume

[root@2fb829e5aeaa /]# cp /etc/passwd /volume/
[root@2fb829e5aeaa /]# ls -l /volume/
total 4
-rw-r--r-- 1 root root 703 Dec 21 06:37 passwd

<mount 정보>
[root@2fb829e5aeaa /]# mount | grep volume
/dev/mapper/ubuntu--vg-ubuntu--lv on /volume type ext4 (rw,relatime,data=ordered)

```

* mount 상태 

* docker volume은 lv로 할당해서 볼륨이 보임... 
* 그런데 용량이 19GB인것이 좀 이상하다.
* 위치는 /var/lib/docker

```
root@docker1:~# docker container inspect test00 | grep  -A 10 "Mounts"
        "Mounts": [
            {
                "Type": "volume",
                "Name": "9515b6854d7d6662cfd786a6aea4fe5c94711720d2bbdd3e5ee551aaf3bacd26",
                "Source": "/var/lib/docker/volumes/9515b6854d7d6662cfd786a6aea4fe5c94711720d2bbdd3e5ee551aaf3bacd26/_data",
                "Destination": "/volume",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
                
<lvm으로 볼륨을 만들어서 할당 한것으로 보임>                
root@docker1:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                         8:0    0   20G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    1G  0 part /boot
└─sda3                      8:3    0   19G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0   19G  0 lvm  /   <<<==== ext4로 mkfs 되어 있음. 
sr0                        11:0    1 58.3M  0 rom

<생성한 파일 확인>
root@docker1:~# ls -l "/var/lib/docker/volumes/9515b6854d7d6662cfd786a6aea4fe5c94711720d2bbdd3e5ee551aaf3bacd26/_data"
total 4
-rw-r--r-- 1 root root 703 Dec 21 06:37 passwd


```



#### 2. named volume 

```
root@docker1:~# docker volume create persistvol
persistvol
root@docker1:~# docker volume ls
DRIVER    VOLUME NAME
local     9515b6854d7d6662cfd786a6aea4fe5c94711720d2bbdd3e5ee551aaf3bacd26
local     persistvol
root@docker1:~# docker volume inspect persistvol
[
    {
        "CreatedAt": "2021-12-21T06:53:20Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/persistvol/_data",
        "Name": "persistvol",
        "Options": {},
        "Scope": "local"
    }
]
```



```
root@docker1:~# docker container  run -it  --name=vol_container -v persisvol:/container centos
[root@834e327c0550 /]# ls container/
[root@834e327c0550 /]# cd container/
[root@834e327c0550 container]# ls
[root@834e327c0550 container]# touch AAAA
[root@834e327c0550 container]# touch BBBB
[root@834e327c0550 container]# touch CCCC

[root@834e327c0550 /]# df
Filesystem                        1K-blocks    Used Available Use% Mounted on
overlay                            19475088 7606748  10856016  42% /
/dev/mapper/ubuntu--vg-ubuntu--lv  19475088 7606748  10856016  42% /container   <<===
```



* 실제 mount 위치 inspect

```
root@docker1:~# docker container inspect vol_container  | grep  -A 10  "Mount"
        "Mounts": [
            {
                "Type": "volume",
                "Name": "persisvol",
                "Source": "/var/lib/docker/volumes/persisvol/_data",
                "Destination": "/container",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }

<생성된 파일은 있구나>            
root@docker1:~# ls -l "/var/lib/docker/volumes/persisvol/_data"
total 0
-rw-r--r-- 1 root root 0 Dec 21 06:55 AAAA
-rw-r--r-- 1 root root 0 Dec 21 06:55 BBBB
-rw-r--r-- 1 root root 0 Dec 21 06:55 CCCC
```



#### 3. 볼륨 컨테이너 이용한 volume 

* 기존 생성된 볼륨을 컨테이너의 볼륨을 mount

```
root@docker1:~# docker container run -it  --volumes-from vol_container --name=data_container centos
[root@3700e9d0425a /]# ls
bin  container  dev  etc  home  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
[root@3700e9d0425a /]# ls -l /container/
total 0
-rw-r--r-- 1 root root 0 Dec 21 06:55 AAAA
-rw-r--r-- 1 root root 0 Dec 21 06:55 BBBB
-rw-r--r-- 1 root root 0 Dec 21 06:55 CCCC

```





## Docker Network

#### veth**@if

* docker0 

```
root@docker1:~# ip a
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
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:26:07:e8:04 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:26ff:fe07:e804/64 scope link
       valid_lft forever preferred_lft forever
23: veth44e7f84@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether ce:f7:fd:83:6a:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ccf7:fdff:fe83:6a87/64 scope link
       valid_lft forever preferred_lft forever
25: vethd74296a@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 2a:fd:9e:f9:4f:01 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::28fd:9eff:fef9:4f01/64 scope link
       valid_lft forever preferred_lft forever
27: veth25f3f20@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether c6:b0:5f:cc:96:22 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::c4b0:5fff:fecc:9622/64 scope link
       valid_lft forever preferred_lft forever
```



```
root@docker1:~# docker exec -it  test00 /bin/bash
[root@2fb829e5aeaa /]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
22: eth0@if23: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

```

#### bridged Network 

```
root@docker1:~# brctl show docker0
bridge name     bridge id               STP enabled     interfaces
docker0         8000.02422607e804       no              veth25f3f20
                                                        veth44e7f84
                                                        vethd74296a

```





```
root@docker1:~# docker run -d --name netbr00 --network=bridge -p 8080:80 nginx

<웹브라우져에서>
http://192.168.137.101:8080/

```

#### iptables

```
root@docker1:~# iptables -t nat -nL
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
```



```
root@docker1:~# ip a
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:5a:a9:d9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.137.101/24 brd 192.168.137.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5a:a9d9/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:26:07:e8:04 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:26ff:fe07:e804/64 scope link
       valid_lft forever preferred_lft forever
29: veth3298eab@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether ea:47:27:b5:4c:55 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::e847:27ff:feb5:4c55/64 scope link
       valid_lft forever preferred_lft forever
```



#### route

```
root@docker1:~# ip route
default via 192.168.137.1 dev enp0s3 proto static
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.137.0/24 dev enp0s3 proto kernel scope link src 192.168.137.101
root@docker1:~# netstat -nr
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         192.168.137.1   0.0.0.0         UG        0 0          0 enp0s3
172.17.0.0      0.0.0.0         255.255.0.0     U         0 0          0 docker0
192.168.137.0   0.0.0.0         255.255.255.0   U         0 0          0 enp0s3

```



## docker images

### docker image

```
root@docker1:~# docker images -a
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    f6987c8d6ed5   5 hours ago    141MB
ubuntu       latest    ba6acccedd29   2 months ago   72.8MB
centos       latest    5d0da3dc9764   3 months ago   231MB
```



```
root@docker1:~# docker image ls -a
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    f6987c8d6ed5   5 hours ago    141MB
ubuntu       latest    ba6acccedd29   2 months ago   72.8MB
centos       latest    5d0da3dc9764   3 months ago   231MB
```



#### inspect

```
root@docker1:~# docker image inspect centos
[
    {
        "Id": "sha256:5d0da3dc976460b72c77d94c8a1ad043720b0416bfc16c52c45d4847e53fadb6",
        "RepoTags": [
            "centos:latest"
        ],
        "RepoDigests": [
            "centos@sha256:a27fd8080b517143cbbbab9dfb7c8571c40d67d534bbdee55bd6c473f432b177"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2021-09-15T18:20:05.184694267Z",
        "Container": "9bf8a9e2ddff4c0d76a587c40239679f29c863a967f23abf7a5babb6c2121bf1",
        "ContainerConfig": {
            "Hostname": "9bf8a9e2ddff",
            "Domainname": "",
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": ["/bin/sh","-c","#(nop) ","CMD [\"/bin/bash\"]"
            ],
            "Image": "sha256:f5b050f177fd426be8fe998a8ecf3fb1858d7e26dff4080b29a327d1bd5ba422",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
            }
        },
        "DockerVersion": "20.10.7",
        "Author": "",
        "Config": {
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
            ],
            "Cmd": [
                "/bin/bash"
            ],
            "Image": "sha256:f5b050f177fd426be8fe998a8ecf3fb1858d7e26dff4080b29a327d1bd5ba422",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "org.label-schema.build-date": "20210915",
                "org.label-schema.license": "GPLv2",
                "org.label-schema.name": "CentOS Base Image",
                "org.label-schema.schema-version": "1.0",
                "org.label-schema.vendor": "CentOS"
            }
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 231268856,
        "VirtualSize": 231268856,
        "GraphDriver": {
            "Data": {
                "MergedDir": "/var/lib/docker/overlay2/0c6c5c7261012598557885a41025c1a2d83062aad14facb410d4974bd4fa3a71/merged",
                "UpperDir": "/var/lib/docker/overlay2/0c6c5c7261012598557885a41025c1a2d83062aad14facb410d4974bd4fa3a71/diff",
                "WorkDir": "/var/lib/docker/overlay2/0c6c5c7261012598557885a41025c1a2d83062aad14facb410d4974bd4fa3a71/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:74ddd0ec08fa43d09f32636ba91a0a3053b02cb4627c35051aff89f853606b59"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]
```





#### docker  pull <image:tag>

```

root@docker1:~# docker image ls -a
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
nginx        latest    f6987c8d6ed5   6 hours ago    141MB
mariadb      latest    e2278f24ac88   5 weeks ago    410MB
ubuntu       latest    ba6acccedd29   2 months ago   72.8MB
centos       7         eeb6ee3f44bd   3 months ago   204MB
centos       latest    5d0da3dc9764   3 months ago   231MB
```

