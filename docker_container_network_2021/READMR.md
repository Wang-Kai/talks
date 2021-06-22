## 实验环境

### OS

```console
$ hostnamectl
   Static hostname: 192-168-0-101
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 8d74bb1ea87bdf013735410a8a56e0c8
           Boot ID: 3958fbcb1bf54705bde39d1bc28407f2
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 4.19.0-9.el7.ucloud.x86_64
      Architecture: x86-64
```

### Docker

```console
$ docker version
Client:
 Version:           18.06.3-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        d7080c1
 Built:             Wed Feb 20 02:26:51 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.3-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       d7080c1
  Built:            Wed Feb 20 02:28:17 2019
  OS/Arch:          linux/amd64
  Experimental:     false
```

## 实验一：从 Linux 机器上徒手演示容器构建

### 1. 修改进程根目录

`chroot` 命令可以用来修改进程的根目录

```
$ chroot centos /bin/sh

$ pwd
/root
```

### 2. 隔离 PID namespace

宿主机上执行

```
top
```

容器内可以 kill 宿主机进程

```
$ chroot centos /bin/sh
// 容器内执行
$ mount -t proc proc /proc

// kill 数组机上的 top 进程
$ pkill top
```

设定 pid namespace，隔离容器可见性

```console
$ unshare -p -f --mount-proc=$PWD/centos/proc chroot centos /bin/bash

// 限定 pid namespace 后，仅能看到 namespace 内的进程
$ top
```

### 3. 使用 cgroup 限定资源使用

首先，需要创建一个 memory namespace，在 `/sys/fs/cgroup/memory` 下创建一个目录，会自动填充相关文件。

```console
mkdir /sys/fs/cgroup/memory/go-demo

// 限定内存
echo "100000000" > /sys/fs/cgroup/memory/go-demo/memory.limit_in_bytes
// 禁用 swap
echo "0" > /sys/fs/cgroup/memory/go-demo/memory.swappiness

// 把 bash 对应的 PID 加入到 memory namespace
echo 2014 > /sys/fs/cgroup/memory/go-demo/tasks
```

做了 memory cgroup 限定后，go 程序使用内存超限后，会被 kill 掉。go 代码如下：

```go
import (
	"crypto/rand"
	"fmt"
	"time"
)

func main() {
	var memo = make([]byte, 0)
	var i int

	for {

		c := 1024 * 1024 * 10
		b := make([]byte, c)
		_, err := rand.Read(b)
		if err != nil {
			fmt.Println("error:", err)
			return
		}

		i++
		memo = append(memo, b...)
		fmt.Printf("==> read %dM \n", 10*i)

		time.Sleep(time.Second * 2)
	}
}
```

## 实验二：veth pair 连通主机 & 容器网络

```console
// 创建 veth pair
ip link add veth0 type veth peer name veth1

// 创建 network namespace
ip netns add test

// 把其中一个网卡加入到 namespace
ip link set veth0 netns test

// 分别启动两个网卡
ip link set veth1 up
ip netns exec test ip link set veth0 up

// 为网卡设置 IP 地址
ip addr add 10.1.0.2/24 dev veth1
ip netns exec test ip addr add 10.1.0.1/24 dev veth0

// 连通性测试
ping -I 10.1.0.2 10.1.0.1

// 另一端抓包
ip netns exec test tcpdump -i veth0 -n
```

## 实验三：从主机网络到容器网络的全链路抓包

一个从 Internet 到容器的网路流量路径应该是：

1. 主机 eth0
2. iptable NAT 转发给 docker0
3. docker0 转发给容器
4. 容器内 veth0 接收到最总流量

```console
// 抓包主机网卡
tcpdump -i eth0 port 80

// 抓包 docker0 switch
tcpdump -i docker0

// 抓包 veth pair 主机端

// 抓包 veth pair 容器端

// 查看 nginx 日志
```


