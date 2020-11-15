以下内容分四步
一、Google Cloud Platform/ AWS 虚拟机部署(修改防火墙规则)
二、升级VPS内核开启BBR
三、搭建Shadowsocks server
四、设置Shadowsocks server开机启动

一、Google Cloud Platform虚拟机部署
1.申请试用GCP
谷歌云平台可让您构建和主机应用程序和网站，存储数据，并分析对谷歌的可扩展基础架构的数据。

2.修改防火墙
这一步很多教程是最后再设置，但是我选择提前设置，这样后面部署好SS服务就可以直接使用了。

菜单中依次点击 【网络】 –> 【防火墙规则】 –> 【创建防火墙规则】

3.获取静态IP
这一步很重要，只有有了静态IP，你后面部署的SS服务才能用。
直接访问 请点击

或者在菜单中依次点击 【网络】–> 【外部 IP 地址】 –> 【保留静态 IP】

静态IP

名称自定义即可
PS：静态 IP 只能申请一个！！！
大陆速度最佳的机房是台湾彰化的机房了，asia-east1-c对大陆最友好
还有东京亚洲东区，也就是东京机房了asia-northeast1-a

4.创建计算引擎
直接访问 ： 请点击
或者在菜单中依次点击 【计算引擎】–> 【创建实例】

创建计算引擎

机器类型里面选最便宜的那个微型就够用，启动磁盘选**Ubuntu16.04LTS**，其他系统也可以只要是你会部署的

给vps配置静态

这里内部ip选择你刚刚得到的那个静态IP，这样虚拟机就完成了设置

配置后

Google Cloud 自带的浏览器 SSH 挺好用的，推荐使用！另外如果你是chrome浏览器的话推荐SSH for Google Cloud Platform 这个插件给你，chrome内扩展程序安装地址是：

请点击

点击上图的ssh后就直接弹出来

SSH

至此，第一部分GCD上的准备工作和部署全部完成，

二、升级vps内核开启BBR
由于众所周知的原因，单纯部署完shadowsocks服务之后速度都不会太理想，即使你选择的是台湾、日本这种很好的线路，依然会存在丢包和不稳定的情况。因为这一步会需要重新启动，所以放在部署SS服务之前。
BBR 目的是要尽量跑满带宽, 并且尽量不要有排队的情况, 效果并不比速锐差。
Linux kernel 4.9+ 已支持 tcp_bbr 下面简单讲述基于 KVM 架构 VPS 如何开启。

准备工作
进入ssh后不是root权限，先获取root权限
>sudo –i

更新系统（两行命令分开执行，第二步等待时间较长，会出现####和进度百分百，耐心等）
>apt update
>apt upgrade

查看当前内核版本
>uname –a

然后你会发现发现版本低于 4.9
安装新内核
>apt install linux-image-4.10.0-20

卸载旧内核
>apt autoremove

启用新内核
>update-grub

重启
>Reboot

验证内核版本
>uname –r

看到如下类似如下回显，版本号为4.10.0-20-generic

启用BBR
写入配置
>echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
>echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

配置生效
>sysctl -p

检验
>lsmod | grep bbr

看到回显tcp_bbr 20480 0说明已经成功开启 BBR
不需要重新启动，我们接下来直接开始在虚拟机部署SS

三、搭建Shadowsocks server
首先更新一下 apt-get 软件包
>sudo apt-get update

然后通过 apt-get 安装 python-pip
>sudo apt-get install python-pip

完成之后使用 pip 安装 shadowsocks 服务
>sudo pip install shadowsocks

如果报错可参考https://stackoverflow.com/questions/36394101/pip-install-locale-error-unsupported-locale-setting

然后我们需要创建一个 shadowsocks server 的配置文件，可以直接建在当前用户目录下
>sudo vim /etc/shadowsocks/ss.json

回车之后会进入这个创建的文件，按键盘上的 insert键会进入编辑，然后把下面的内容输入进去。按ESC键会发现左下角的insert消失，shift+：这个组合键左下角出现：输入wq回车就保存退出文件。
端口和密码，设置成你想要的就行了

```
{
"server":"0.0.0.0",
"server_port":8838,
"local_address":"127.0.0.1",
"local_port":1080,
"password":"123456",
"timeout":600,
"method":"aes-256-cfb"
}
```
启动 shadowsocks 服务
>sudo ssserver -c /etc/shadowsocks/ss.json -d start


四、设置Shadowsocks server开机启动
虽然我知道服务器一般是不重启的，但是万一重启了，还得重新运行shadowsocks server还是很麻烦的，就想将 shadowsocks 添加到开机运行中去。
创建脚本 /etc/init.d/shadowsocks
>sudo vim /etc/init.d/shadowsocks

添加以下内容
```
#!/bin/sh
### BEGIN INIT INFO
# Provides: shadowsocks
# Required-Start: $remote_fs $syslog
# Required-Stop: $remote_fs $syslog
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: start shadowsocks 
# Description: start shadowsocks
### END INIT INFO

start(){
        ssserver -c /etc/shadowsocks/ss.json -d start
}

stop(){
        ssserver -c /etc/shadowsocks/ss.json -d stop
}

case "$1" in
start)
        start        
        ;;
stop)
        stop        
        ;;
reload)
        stop
        start        
        ;;
*)
        echo "Usage: $0 {start|reload|stop}"
        exit 1        
        ;;
esac

```

然后增加这个文件的可执行权限
>sudo chmod +x /etc/init.d/shadowsocks

注意：这里命令的权限,我想以root权限运行，如果不想以root权限运行可以用sudo -u {user} {command}。如果不给此脚本文件加上其他用户也可执行权限，在运行service shadowsocks不加参数时会提示unrecognized service.

创建文件 /etc/init/shadowsocks.conf
>sudo vim /etc/init/shadowsocks.conf
```
start on (runlevel [2345])stop on (runlevel [016])pre-start script
/etc/init.d/shadowsocks start
end script

post-stop script
/etc/init.d/shadowsocks stop
end script
```
执行

>sudo update-rc.d shadowsocks defaults

添加到开机启动中
好了，搞定，可以在shell中直接运行

>sudo service shadowsocks {start|reload|stop}

来控制了！



**或者**
vi /etc/rc.local
加入：sudo ssserver -c /etc/shadowsocks/ss.json -d start
(倒数第二行）
