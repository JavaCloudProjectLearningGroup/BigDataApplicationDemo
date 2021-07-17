# 03 hadoop-HDFS集群搭建-伪分布式模式

## 搭建思路

- 基础设施
- 部署配置
- 初始化运行
- 命令行使用

## 基础设施

操作系统、环境、网络、必须软件

1. 设置IP及主机名
2. 关闭防火墙&selinux
3. 设置hosts映射
4. 时间同步
5. 安装jdk
6. 设置SSH免秘钥

## Hadoop 安装

1. 基础设施：

   设置网络

   设置IP
   vi etc/sysconfig/network-scripts/ifcfg-eth0

   设置主机名
   vi etc/sysconfig/network

   设置本机的IP到主机名的映射
   vi etc/hosts

   关闭防火墙
   service iptables stop
   chkconfig iptables off

   关闭selinux
   vi etc/selinux/config
   SELINUX=disabled

   做时间同步
   yum install ntp -y
   vi etc/ntp.conf
   server ntp1.aliyun.com（和aliyun同步）

   安装JDK

   rpm -i jdk-8u181-linux-x64.rpm