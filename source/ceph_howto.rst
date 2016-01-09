===================
使用saltstack部署ceph环境
===================

:Date: 2015-06-25
:Version: 1.0
:Authors: 董大伟  dawei@yunshan.net.cn 
:Changelog: init
 
.. topic:: 关于SaltStack

   **SaltStack** 是一个服务器基础架构集中化管理平台，具备配置管理、远程执行、监控等功能。

   **SaltStack** 基于 *Python* 语言实现，结合轻量级消息队列（``ZeroMQ``）与  *Python* ( 第三方模块
   （Pyzmq、PyCrypto、Pyjinjia2、python-msgpack 和 PyYAML 等）构建。 通过部署
   **SaltStack** 环境，我们可以在成千上万台服务器上做到批量执行命令，
   根据不同业务特性进行配置集中化管理、分发文件、采集服务器数据、操作系统基础及软件包管理等。

   和大多数类似工具一样，**SaltStack** 需要在一台机器（主控）上安装服务器端软件
   （SaltStack 称之为 ``salt master`` ），在多台机器（受控）上安装客户端软件
   （SaltStack 称之为 ``salt minion`` ）。在主控机器上给下属（受控）发命令，在受控机器
   上接受和执行上级（主控）的命令。

   **CEPH** 部署的方式通常有2种（官网推荐的 ``Ceph-Deploy`` 和 手动命令行 部署方式），由于 
   ``Ceph-Deploy`` 配置较复杂，不适合 *LiveCloud* 快速部署应用；手动配置，命令繁多，
   容易出错。结合  **SaltStack** 具有集中部署的优点，特将二者整合，形成了 ``livecloud`` 的Ceph部署工具。

----
部署前提
----
硬件及系统准备

* 每台  *Ceph* 节点主机至少有2块单独的 SATA 或 SAS 磁盘； 
* 每台  *Ceph* 节点推荐使用 SSD 做  *Ceph* 的 *Journal* 盘（具体请参考相关文档） 
* 网络配置完毕（推荐要求有存储平面，网络为10G网络） 

安装准备

* 在Ceph节点上安装 ``CentOS 7.0（3.10.0-123.el7.x86_64）`` ，安装时选择  ``Infrastructure Server`` 。 
* 使用  ``lc_xenctl install kvmhost`` 安装kvm之后（如何初始化kvm主机，请参考相关文档） 
* 准备一台主机或虚拟机作为 **Salt-Master** （安装**Salt-Master** 软件，提供repo，和ntp服务器等） 

-------
部署的基本步骤
-------

#. 检查 KVM 基本安装，确保 ``lc_xenctl`` 配置已完成，检查 libvirt 是否安装

   .. sourcecode:: console

      rpm -qa | grep libvirt

#. 在 ``salt-master`` 节点上安装所需要的软件，下载所需的安装包，执行

   .. sourcecode:: console
   
      tar zxf deployment-toolkit.tar.gz
      cd deployment-toolkit
      sh install.sh   # 安装salt-master

#. 配置 ``salt-master``，将相应的配置写入到  ``salt-master`` 中，编辑如下两个文件

   .. sourcecode:: console

      vim master.yml
      vim ceph.yml
      # 确认无误后，执行:
      python setup.py

   .. warning:: 

      如果之前已经安装过salt-minion，请执行:

      .. sourcecode:: console
   
         rm -fr /var/cache/salt/minion/* && systemctl restart salt-minion

      或者执行:

      .. sourcecode:: console
   
         salt '*' saltutil.clear_cache

#. 由 ``salt-master`` 自动完成 ``salt-minion`` 节点的安装

   .. sourcecode:: console

      salt-ssh '*' -r 'echo "<saltmasterIP> <saltmaster_hostname>" >> /etc/hosts'
      salt-ssh '*' state.sls ceph.minion

#. 认证minion节点

   .. sourcecode:: console
   
      # 认证key
      salt-key -L
      salt-key -A -y

#. 由 ``salt-master`` 状态同步

   .. sourcecode:: console
   
      salt '*' state.highstate                # 状态同步
      salt '*' state.sls ceph.ntp             # 安装并配置ntp
      salt '*' state.sls ceph.ceph            # 安装ceph
      salt '*' state.sls ceph.kvm             # 安装kvm

#. 由 ``salt-master`` 配置 *ceph* 的mon，osd和pool信息

   .. sourcecode:: console
   
      salt '*' saltutil.refresh_pillar        # 更新pillar
      salt '*' saltutil.sync_all              # 更新模块
      salt '*' ceph.journal                   # 配置journal盘
      salt '*' ceph.mon                       # 配置mon
      salt '*' ceph.osd                       # 配置osd
      salt '*' ceph.pool                      # 配置pool

#. 修改pyagent文件和关联kvm pool信息

   .. sourcecode:: console

      salt '*' kvm.pool                       # 配置kvm-pool
      salt '*' state.sls ceph.pyagexec        # 配置pyagexec

----
部署举例
----

以下以一个 ``salt-master`` 和3个 *ceph* 节点为例，详细介绍saltstack部署ceph环境。

.. csv-table:: 部署环境
   :header: Role,IP,Storage_IP,Public_IP,OS
   :widths: 5, 5, 5, 5, 5
   
   bss, 172.16.1.23,   ,10.33.39.23, CentOS6.5-x86_64 minimal 
   oss, 172.16.1.24,   ,10.33.39.24, CentOS6.5-x86_64 minimal 
   centos104, 172.16.1.104, 172.20.1.104,   ,CentOS7.0-x86_64 basic 
   centos106, 172.16.1.106, 172.20.1.104,  , CentOS7.0-x86_64 basic 
   centos112, 172.16.1.112, 172.20.1.104,   ,CentOS7.0-x86_64 basic 
   salt-master, 172.16.39.11,   ,10.33.39.11, CentOS7.0-x86_64 basic 

salt-master安装
=============

上传安装文件到 ``salt-master``虚拟机或服务器上，并解压缩tar包:

.. sourcecode:: console

   [root@localhost opt]# tar -zxvf deployment-toolkit.tgz
   [root@localhost opt]# cd /opt/deployment-toolkit/ceph-deploy

修改master.yml文件:

.. sourcecode:: console

   [root@localhost opt]# vim master.yml
   ---
   base_dir: /opt/deployment-toolkit/ceph-deploy
   nodes:
     - name: centos104
       ip: 172.16.1.104
       user: root
       passwd: yunshan3302
     - name: centos106
       ip: 172.16.1.106
       user: root
       passwd: yunshan3302
     - name: centos112
       ip: 172.16.1.112
       user: root
       passwd: yunshan3302
   ...

.. warning::

   修改master.yml文件时请确保base_dir的位置正确

修改ceph.yml文件:

.. sourcecode:: console

   [root@localhost opt]# vim ceph.yml
   ---
   ceph:
     global:
       cluster_network: 172.20.0.0/16
       public_network: 172.20.0.0/16
       fsid: 294bc494-81ba-4c3c-ac5d-af7b3442a2a5
     mon:
       interface: eth23
     pools:
     - name: capacity
       pg_num: 128
       pgp_num: 128
     - name: performance
       pg_num: 128
       pgp_num: 128
   nodes:
     master:
       hostname: centos39_11
       ip: 172.16.39.11
     ntp:
       ntpservers:
       - 172.16.39.11
       localnetworks:
       - 172.16.0.0
     centos104:
       roles:
       - ceph-osd
       - ceph-mon
       devs:
       - sdb
       - sdc
   
     centos106:
       roles:
       - ceph-osd
       - ceph-mon
       devs:
       - sdb
       - sdc
   
     centos112:
       roles:
       - ceph-osd
       - ceph-mon
       devs:
       - sdb
   - sdc
   ...

.. topic:: ceph.yml说明

   + cluster_network：为存储平面IP地址段 
   + interface： 为存储平面网卡，建议统一配置 
   + ntp：如没有专用NTP服务器，请指定为salt-master IP 

上面为没有SSD Journal盘的配置，如果有专用SSD盘，请node部分参考如下的配置:

.. sourcecode:: console

   nodes:
     master:
       hostname: centos39_11
       ip: 172.16.39.11
     ntp:
       ntpservers:
       - 172.16.39.11
       localnetworks:
       - 172.16.0.0
     centos104:
       roles:
       - ceph-osd
       - ceph-mon
       journal:
         sdb:
           partition:
             per_size: 8G
             count: 2
       devs:
         sdc:
           journal: sdb1
         sdd:
           journal: sdb2
   
     centos106:
       roles:
       - ceph-osd
       - ceph-mon
       journal:
         sdb:
           partition:
             per_size: 8G
             count: 2
       devs:
         sdc:
           journal: sdb1
         sdd:
           journal: sdb2
   
     centos112:
       roles:
       - ceph-osd
       - ceph-mon
       journal:
         sdb:
           partition:
             per_size: 8G
             count: 2
       devs:
         sdc:
           journal: sdb1
         sdd:
   journal: sdb2
   ...

修改好yml文件后执行 如下命令安装 ``salt-master``:

.. sourcecode:: console

   sh install.sh

一般情况下，执行完该脚本后会启动如下的三个服务 monkey，ntpd 和 salt-master 
由于该salt-master对外提供yum repo，请关闭防火墙。

执行python setup.py生成供saltstack使用的pillar数据

.. sourcecode:: console

   python setup.py
   Generate in /opt/deployment-toolkit/ceph-deploy/pillar/ceph.sls
   
安装salt-minion
=============

完成 ``salt-master`` 安装完毕后，依次执行如下命令:

.. sourcecode:: console

   salt-ssh '*' -r 'echo "172.16.39.11 salt-master" >> /etc/hosts'

该命令会将 ``salt-master`` 信息写入到每个minion所在的节点上，方便 ``salt-ssh`` 
安装minion组件，输出如下的结果:

.. sourcecode:: console

   centos104:
       ----------
       retcode:
           0
       stderr:
       stdout:
           root@172.16.1.104's password:
   centos112:
       ----------
       retcode:
           0
       stderr:
       stdout:
           root@172.16.1.112's password:
   centos106:
       ----------
       retcode:
           0
       stderr:
       stdout:
           root@172.16.1.106's password:

安装 ``salt-minion`` 节点:

.. sourcecode:: console

   salt-ssh '*' state.sls ceph.minion

执行成功后，可以登录到 ``salt-minion`` 节点上输入:

.. sourcecode:: console

   systemctl status salt-minion

查看是否安装成功。

认证minion节点
==========

.. note::

   如果之前使用saltstack部署过ceph，请执行:

   .. sourcecode:: console
   
      rm -fr /var/cache/salt/master/* && systemctl restart salt-master

默认 ``salt-minion`` 节点在 ``salt-master`` 上处于未认证状态 Unaccepted Keys，
需要将 ``salt-minion`` 节点加入到 ``salt-master`` 上，纳入  ``salt-master`` 的管理，
salt-key 的基本用法如下：:

.. sourcecode:: console

   # 显示所有minion的认证信息
   salt-key -L
   
   # 接受172.16.1.104的认证信息
   salt-key -a 172.16.1.104
   
   # 接受172.16.1.104的认证信息，不需要手动验证
   salt-key -a 172.16.1.104 -y
   
   # 接受172.16.1.104的认证信息，即使该minion是Rejected Keys状态
   salt-key -a 172.16.1.104 --include-all
   
   # 接受所有 Unaccepted Keys 状态的minion的认证信息
   salt-key -A
   
   # 拒绝认证192.168.0.100
   salt-key -d 192.168.0.100
   
   # 拒绝所有 Unaccepted Keys 状态的minion
   salt-key -D

.. note:: 

   如果使用 salt-key -L 看不到任何信息，请到每个 minion 节点上手动重启 minion 服务。或者执行

   .. sourcecode:: console

      salt-ssh '*' -r 'systemctl restart salt-minion'

配置ceph节点
========

正确配置minion节点之后，在master上执行:

.. sourcecode:: console

   salt '*' state.highstate

该命令将master上的pillar的数据和状态同步到minion节点上。

如果使用 ``salt-master`` 作为NTP服务器的话执行如下操作，否则略过:

.. sourcecode:: console

   salt '*' state.sls ceph.ntp

安装或升级ceph:

.. sourcecode:: console

   salt '*' state.sls ceph.ceph

安装或升级libvirt:
   
.. sourcecode:: console
   
   salt '*' state.sls ceph.kvm

如果存在单独的 ``SSD Journal`` 盘，请执行如下操作，否则略过:

.. sourcecode:: console

   salt '*' ceph.journal

配置MON节点信息:

.. sourcecode:: console
   
   salt '*' ceph.mon

配置OSD节点信息:

.. sourcecode:: console
   
   salt '*' ceph.osd

创建ceph pool:

.. sourcecode:: console
   
   salt '*' ceph.pool

关联KVM里面的pool和 ``ceph pool``:

.. sourcecode:: console
   
   salt '*' kvm.pool
为pyagexec生成相应的配置文件:

.. sourcecode:: console

   salt '*' state.sls ceph.pyagexec

ceph环境验证
========

在某个节点上执行如下命令，可以看到当前的ceph状态:

.. sourcecode:: console

   ceph osd tree
   ID WEIGHT   TYPE NAME              UP/DOWN REWEIGHT PRIMARY-AFFINITY
   -1 10.57999 root default
   -2  2.00000         host centos104
    8  1.00000             osd.8           up  1.00000          1.00000
    3  1.00000             osd.3           up  1.00000          1.00000
   -3  2.00000         host centos106
    4  1.00000             osd.4           up  1.00000          1.00000
    7  1.00000             osd.7           up  1.00000          1.00000
   -4  2.00000         host centos112
    5  1.00000             osd.5           up  1.00000          1.00000
    6  1.00000             osd.6           up  1.00000          1.00000

ceph模板的导入
=========

ceph环境正确部署后，将模板文件qcow2格式的放在任意一台ceph节点上

.. sourcecode:: bash

   sh import_to_ceph.sh template-centos6.5.qcow2 _01_CentOS_02_6.5_04_64Bit_05_En capacity

即可以完成模板的导入，import_to_ceph.sh如下：

.. sourcecode:: bash

   #!/bin/bash
   
   ARGS=3
   E_BADARGS=65
   
   if [ $# -ne $ARGS ]  # Correct number of arguments passed to script?
   then
       echo "Usage: `basename $0` <qcow2_name> <vm_name> <pool_name>"
       exit $E_BADARGS
   fi
   
   QCOW2_NAME=$1
   VM_NAME=$2
   POOL_NAME=$3
   
   qemu-img convert -O raw $QCOW2_NAME $VM_NAME -p
   rbd -p $POOL_NAME --image-format 2 import --stripe-unit 65536 --stripe-count 16 $VM_NAME
   virsh pool-refresh $POOL_NAME
   
   echo Done
 
查看模板信息:

.. sourcecode:: console

   rbd -p capacity ls
