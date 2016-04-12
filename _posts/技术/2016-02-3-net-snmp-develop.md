---
layout: post
title: 基于Net-SNMP简单网络管理的开发指南
category: 技术
tags: openWrt switch
keywords: 特征值
description: 
---


## 1 Net-SNMP简介
Net-SNMP是一个免费的、开放源码的SNMP实现，以前称为UCD-SNMP。它包括agent和多个管理工具的源代码，支持多种扩展方式。不仅扩展了获取方式，而且对于数据类型也有一定的扩展。

**阅读之前需要了解的知识：**

+ 懂linux操作系统的使用
+ 了解snmp协议基本原理
+ Net-SNMP工具的使用
+ 熟悉make自动化工具的所用和阅读makefile代码的能力


## 2 开发私有mib库例子
用net-snmp扩展MIB库，实现方法可归结为四种：

1. 一是静态库方式，通过修改配置头文件，在相应地方包含新引入的mib模块的.c和.h文件，然后重新编译库文件和扩展代码；这种方式不够灵活，每次修改扩展的MIB后，都需要重新编译snmpd和扩展的代码，再重新安装snmpd到系统中。
2. 二是编译动态共享库，只需把新引入的MIB模块的.c和.h文件编译成动态库，通过设置能够让代理程序载入。
3. 三是扩展一个子代理，让SNMPD以主代理的模式运行，对于SNMPD我们只要让它启动就可以，不需要任何的更改和配置，把子代理编译生成的程序运行起来就可以扩展自定义的MIB库。
4. 用shell脚本来扩展

对于第二种方式，一需要编译成.so动态共享库，二需要原代理程序是否包含dlmod或load命令，三还要看系统是否支持。本文我们以第二种方法在linux上开发和测试，Net-SNMP的安装请自行学习。

首先进入[Net-SNMP官网](http://www.net-snmp.org/)获取demo代码，点击进入[Totorials](http://www.net-snmp.org/wiki/index.php/Tutorials)界面，然后可以找到[Writing a Dynamically Loadable Object](http://www.net-snmp.org/wiki/index.php/TUT:Writing_a_Dynamically_Loadable_Object) 点击进入，可以查找到有如下四个文件。

>[Makefile](http://www.net-snmp.org/tutorial/tutorial-5/toolkit/demoapp/Makefile)：A simple makefile used to build the projects
>
>[NET-SNMP-TUTORIAL-MIB.txt](http://www.net-snmp.org/tutorial/tutorial-5/toolkit/mib_module/NET-SNMP-TUTORIAL-MIB.txt)：The MIB we'll be writing code for in the various pieces of the agent extension tutorial
>
>[nstAgentPluginObject.h](http://www.net-snmp.org/tutorial/tutorial-5/toolkit/dlmod/nstAgentPluginObject.h)：The MIB module's header file
>
>[nstAgentPluginObject.c](http://www.net-snmp.org/tutorial/tutorial-5/toolkit/dlmod/nstAgentPluginObject.c)：The MIB module's C source code

创建相应的文件名，将这四个文件内容拷贝下来

### 2.1 加载mib文件

使用net-snmp-config命令获取mib文件加载的默认目录

    ccj@debian:~$ net-snmp-config --default-mibdirs #获取mib文件默认读取目录
    /home/ccj/.snmp/mibs:/usr/local/share/snmp/mibs

以上两个目录都可以加载mib文件，由于我安装了之后对于以上两个目录里都没有snmp.conf文件，所以直接在源代码里查找到snmp.conf文件直接拷贝修改，已经有的可以直接修改。我的用户目录下也没有/home/ccj/.snmp/mibs这个目录，手动创建/home/ccj/.snmp/mibs目录，然后把NET-SNMP-TUTORIAL-MIB.txt文件拷贝到/home/ccj/.snmp/mibs下而snmp.conf配置文件需要拷贝到/home/ccj/.snmp目录下，修改snmp.conf配置文件添加相应mib模块。

    mibs +NET-SNMP-TUTORIAL-MIB

通过snmptranslate命令确认私有mib模块是否加载成功，


	ccj@debian:~$ snmptranslate -Tp -IR netSnmpExamples
    +--netSnmpExamples(2)
       |
       +--netSnmpExampleScalars(1)
       |  |
       |  +-- -RW- Integer32 netSnmpExampleInteger(1)
       |  +-- -RW- Integer32 netSnmpExampleSleeper(2)
       |  +-- -RW- String    netSnmpExampleString(3)
       |           Textual Convention: SnmpAdminString
       |           Size: 0..255
       |
       +--......
       |  |
       |  +--......
       |  | 

### 2.2 Agent加载.so动态库
我们编写的mib文件需要生成动态库，然后加载到Agent代理服务器上之后mib文件对于Agent其实已经没有用，上面加载mib模块只是对客服端而言。Net-SNMP提供的工具mib2c可以把NET-SNMP-TUTORIAL-MIB.txt文件里的具体变量生成对应的.c和.h文件。

    ccj@debian:~$ mib2c -c mib2c.scalar.conf nstAgentPluginObject
    writing to nstAgentPluginObject.h
    writing to nstAgentPluginObject.c
    running indent on nstAgentPluginObject.h
    running indent on nstAgentPluginObject.c

你会发现生成的代码和提供的例子代码有点不一样,有可能例子代码是官方手动写的或者是老版本工具生成的，当然这个mib2c工具只能生成一个代码框架，你需要在上面修改，为了方便演示笔者就直接调用官方例子生成动态库。

    ccj@debian:~/project/demomib$ ls
    Makefile                   nstAgentPluginObject.c
    NET-SNMP-TUTORIAL-MIB.txt  nstAgentPluginObject.h
    ccj@debian:~/project/demomib$ make nstAgentPluginObject.so 
    gcc -I. `net-snmp-config --cflags` -fPIC -shared -c -o nstAgentPluginObject.o nstAgentPluginObject.c
    gcc -I. `net-snmp-config --cflags` -fPIC -shared -o nstAgentPluginObject.so nstAgentPluginObject.o
    ccj@debian:~/project/demomib$ ls
    Makefile                   nstAgentPluginObject.c  nstAgentPluginObject.o
    NET-SNMP-TUTORIAL-MIB.txt  nstAgentPluginObject.h  nstAgentPluginObject.so

其中Makefile例子是一个模板代码如果直接make的话会有问题，不过可以生成nstAgentPluginObject.so动态库。你可能找不到snmpd.conf文件，这个文件是要自己编写的，当然你在源代码下可以查找到一个snmpd.conf例子配置,就在下面的目录里，直接拷贝到/home/ccj/.snmp目录下。为什么是拷贝到这个目录下而不是其他目录呢？这个后面会讲解。

	net-snmp-5.4.4/python/netsnmp/tests

在snmpd.conf文件最后添加如下dlmod配置项来加载动态库。 

    dlmod nstAgentPluginObject /home/ccj/project/demomib/nstAgentPluginObject.so

这里还有一问题，你必须得先确定正常情况下启动snmpd时，snmpget或者snmpwalk可以访问自带的mib信息。在snmpd.conf中你会发现如下配置，它的意思是可以访问到本代理设备的只有本机和192.168.1.0网段的主机。如果你是其他主机访问代理端请添加相应访问设备的网段到配置文件里，嫌麻烦把192.168.1.0/24改成default代表全部可以访问。

    #       sec.name  source          community
    com2sec local     localhost         public
    com2sec mynetwork 192.168.1.0/24    public

启动Net-SNMP服务器，如果你已经启动了请先关闭。这里会有一个问题，我们是怎么知道启动时snmpd加载的snmpd.conf配置文件在哪里。我们可以通过下面的选项直接指定哪个配置文件作为启动。其实可以不用指定配置文件下面会解释。

	sudo snmpd -c /home/ccj/.snmp/snmpd.conf

但是我们还是需要看一下Net-SNMP使用时一些配置文件应该放在哪些目录下，可以通过下面的命令获取配置路径都有那些。

    ccj@debian:~$ net-snmp-config --snmpconfpath
    /usr/local/etc/snmp:/usr/local/share/snmp:/usr/local/lib/snmp:/home/ccj/.snmp:/var/net-snmp

从上面的结果可以发现有home/ccj/.snmp目录，其实把snmpd.conf和snmp.conf配置文件拷贝到上面任意一个目录应该都是可以的（笔者没有验证）。
接下来可以通过以下命令确认动态库是否加载成功。

    ccj@debian:~$ snmpwalk -v 2c -c public localhost nstAgentPluginObject
    NET-SNMP-TUTORIAL-MIB::nstAgentPluginObject.0 = INTEGER: 3

### 2.3 trap告警配置
在snmp.conf中配置以下条目

	activeMonitoring trapsink 192.168.2.40:162 public
	option authtrapenable   1