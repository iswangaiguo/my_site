---
title: Linux-Firewalld配置
date: 2022-08-15T08:19:59+00:00
categories:
  - 编程
tags:
  - firewalld
---

Firewalld目前是RHEL和Centos默认的防火墙工具，取代了之前的iptables，不过二者都不是真正的防火墙，是用来定义防火墙策略的防火墙管理工具。iptables服务会把配置好的防火墙策略交由内核层面的netfilter网络过滤器来处理，而firewalld服务则是把配置好的防火墙策略交由内核层面的nftables包过滤框架来处理。

## 相关概念

firewalld定义了几个概念：

* **zone**: 安全域，类似于Window上的域网络，工作网络，家庭网络，Internet网络等，不同的安全作用域其安全级别不同，安全程度不同，家庭zone的安全规则就是最宽松的。
* **service**: 它是zone里面的定义一个规则集，它将iptables中单独根据 入接口，出接口，源目端口，协议等定义单独规则整合为一个规则集合，方便识别和管理。
* **protocol**: 定义协议规则
* **port**，source-port：基于端口定义规则，port我的理解:定义源和目标为指定端口的规则。
* **source**: 定义源地址规则
* **interface**: 定义基于出入接口的规则。
* **direct**: 直接定义原始iptables规则。

## 安全区

相较于传统的防火墙工具，FireWalld引入了区域（Zone）的概念

```bash
[root@VM-0-2-centos ~]# systemctl start firewalld //firewalld默认处于关闭状态

[root@VM-0-2-centos ~]# systemctl status firewalld //查看防火墙状态
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-08-15 09:57:24 CST; 17min ago
       Docs: man:firewalld(1)
   Main PID: 3318 (firewalld)
      Tasks: 2 (limit: 12158)
     Memory: 27.1M
        CPU: 358ms
     CGroup: /system.slice/firewalld.service
             └─3318 /usr/bin/python3 -s /usr/sbin/firewalld --nofork --nopid

Aug 15 09:57:24 VM-0-2-centos systemd[1]: Starting firewalld - dynamic firewall daemon...
Aug 15 09:57:24 VM-0-2-centos systemd[1]: Started firewalld - dynamic firewall daemon.

[root@VM-0-2-centos ~]# firewall-cmd --get-default-zone //firewalld默认安全区
public
```

可以看到FireWalld默认区域为Public，FireWalld定义了以下区域

| 区域 | 默认规则策略 |
|------|------------|
| trusted | 允许所有的数据包 |
| home | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、mdns、ipp-client、amba-client与dhcpv6-client服务相关，则允许流量 |
| internal | 等同于home区域 |
| work | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、ipp-client与dhcpv6-client服务相关，则允许流量 |
| public | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh、dhcpv6-client服务相关，则允许流量 |
| external | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh服务相关，则允许流量 |
| dmz | 拒绝流入的流量，除非与流出的流量相关；而如果流量与ssh服务相关，则允许流量 |
| block | 拒绝流入的流量，除非与流出的流量相关 |
| drop | 拒绝流入的流量，除非与流出的流量相关 |

简单来说，区域就是firewalld预先准备了几套防火墙策略集合（策略模板），用户可以根据生产场景的不同而选择合适的策略集合，从而实现防火墙策略之间的快速切换。

**查看Public定义的规则**

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <forward/>
</zone>
```

## Service

上段中的service标签所对应的服务可以在**/usr/lib/firewalld/services**中查看，比如**ssh**服务，服务对应着相应的协议以及端口，用于防火墙的过滤。

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>SSH</short>
  <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface, enable this option. You need the openssh-server package installed for this option to be useful.</description>
  <port protocol="tcp" port="22"/>
</service>
```

## 常用命令

**systemctl服务相关**

```bash
# systemctl start firewalld.service //启动某个服务
# systemctl stop firewalld.service //关闭某个服务
# systemctl restart firewalld.service //重启某个服务
# systemctl status firewalld.service //显示某个服务的状态
# systemctl enable firewalld.service //开机时随机自启动
# systemctl disable firewalld.service //禁止开机启动
# systemctl is-enabled firewalld.service //查看是否开机启动
# systemctl list-unit-files|grep enabled //查看已经启动的服列表
# systemctl --failed //查看启动失败的服务列表
```

**过滤端口**

```bash
[root@VM-0-7-centos ~]# firewall-cmd --zone=public --add-port=3306/tcp  //允许3306端口所有连接
success
[root@VM-0-7-centos ~]# firewall-cmd --reload //重新加载防火墙策略
success
[root@VM-0-7-centos ~]# firewall-cmd  --zone=public --list-all //查看生效的配置
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client ssh
  ports: 3306/tcp
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

firewalld 配置的防火墙策略默认为运行时（**Runtime**）模式，只能临时生效，是立刻生效的，又称为当前生效模式，而且随着系统的重启会失效。

如果想让配置策略一直存在，就需要使用永久（**Permanent**）模式了，方法就是在 firewall-cmd 命令后面添加 --permanent 参数，这样配置的防火墙策略就可以永久生效了。

但是永久生效模式有一个特点，就是使用它设置的策略只有在系统重启后才会生效，如果想让配置策略永久并立即生效，需要手动执行 **firewall-cmd --reload** 重载命令。

## 过滤IP

```bash
[root@VM-0-7-centos ~]# firewall-cmd --permanent --zone=public --add-rich-rule="rule family='ipv4' source address='172.16.1.10' accept" //允许172.16.1.10连接
[root@VM-0-7-centos ~]# firewall-cmd --reload
success
[root@VM-0-7-centos ~]# firewall-cmd --zone=public --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client ssh
  ports: 3306/tcp
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
        rule family="ipv4" source address="172.16.1.10" accept
