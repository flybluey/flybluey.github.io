---
layout:     post
title:      Oracle RAC修VIP和SCAN-IP的操作过程
subtitle:   元鼎科技
date:       2019-01-22
author:     冯栋华
header-img: img/post-bg-mma-0.jpg
catalog: true
tags:
    - oracle
    - 修改VIP
    - 修改SCAN-IP
    - 案例
---


#背景#
在客户现场经常需要做数据迁移，但是对于一些C/S软件架构的业务系统来说，数据迁移完成之后，经常需要修改终端机器的连接地址，会造成较大的工作量以及迁移风险，所以需要在数据迁移完成之后，将IP地址修改为原先的IP地址信息。

#操作步骤#

##修改VIP过程
###提前关闭所有实例
srvctl stop database -d XXX
###更新ocr网络信息配置，root用户执行
oifcfg getif
oifcfg delif -global eth0/127.0.0.1
oifcfg setif -global eth0/192.168.20.0:public
srvctl config nodeapps -a

[root@stuaapp01 ~]# oifcfg getif
eth1  192.168.10.0  global  cluster_interconnect
eth0  127.0.0.1  global  public
[root@stuaapp01 ~]# oifcfg delif -global eth0/127.0.0.1
[root@stuaapp01 ~]# oifcfg setif -global eth0/192.168.20.0:public
[root@stuaapp01 ~]# 
[root@stuaapp01 ~]# oifcfg getif
eth1  192.168.10.0  global  cluster_interconnect
eth0  192.168.20.0  global  public

###修改主机IP地址
此处是操作系统修改IP地址和hosts文件，按照实际IP地址修改。
此处不做详细描述
###重启RAC集群服务
crsctl stop crs
crsctl start crs

oifcfg getif
oifcfg delif -global eth0/192.168.20.0
oifcfg setif -global eth0/127.0.0.1:public
srvctl config nodeapps -a

##修改VIP
###收集当前VIP配置信息
/u01/app/11.2.0/grid/bin/srvctl config nodeapps
VIP的状态
/u01/app/11.2.0/grid/bin/crs_stat -t
###停止所有的资源
/u01/app/11.2.0/grid/bin/srvctl stop instance -d oggsrc -n stuaapp01
/u01/app/11.2.0/grid/bin/srvctl stop vip -n stuaapp01 -f
###验证VIP现在离线,接口不再绑定到公共网络接口
/u01/app/11.2.0/grid/bin/crs_stat -t (or $ crsctl stat res t for 11gR2)
###修改/etc/hosts文件中的VIP
 crsctl stat res ora.stuaapp01.vip -p
###root用户，修改VIP的资源
/u01/app/11.2.0/grid/bin/srvctl modify nodeapps -n stuaapp01 -A stuaapp01-vip/255.255.255.0/eth0
###验证变化
/u01/app/11.2.0/grid/bin/srvctl config nodeapps -a
###启动资源
$ srvctl start vip -n stuaapp01
$ srvctl start listener -n stuaapp01
$ srvctl start instance -d oggsrc -n stuaapp01 



##修改SCAN-IP
###检查当前的scan ip配置信息，用root用户运行
srvctl config scan
###停止SCAN listeners 和 SCAN，用grid用户运行
srvctl stop scan_listener
srvctl stop scan
###在/etc/hosts文件修改scan ip配置信息，两个节点都需要改
按具体要求修改，此处不做详细说明
###刷新配置信息，用root用户运行
srvctl modify scan -n rac-scan
###验证更改信息，用root用户运行
srvctl config scan
###重启SCAN & SCAN listener，用grid用户运行
srvctl start scan
srvctl start scan_listener
###如果SCAN VIPs的数量发生变化，则需要更新如下信息（非必需操作）
srvctl modify scan_listener -u


#后记
通过上面的操作步骤，就能顺利完成RAC环境的IP地址修改，包括VIP和SCAN-IP。
