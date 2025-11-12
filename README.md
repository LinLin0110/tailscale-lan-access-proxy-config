**保留Tailscale公网穿透/局域网访问——并兼容科学代理，个人配置参考**  
最终效果：iPhone在外使用5G网络，连接Tailscale，既能访问192.168.0.0/24的局域网设备，也能访问Youtube/Google，并且PS5串流也可用
```
       [iPhone 5G] 
            |
     [Tailscale 回家]
            |
     [Unifi Router ← ipt2socks] ←→ [Socks5 - Mac]
                         |
            [局域网设备：PS5/NAS/SMB]
```

**此方案特点**  
✅几乎兼容任何VPN/代理  
✅不挑路由器（理论上支持ssh的路由器都能装）  
✅高稳定，几乎不干涉基础网络/硬路由层面影  
⚠️配置需要用命令行，步骤略多  
⚠️高风控网站比如ChatGPT 可能被卡（显示地区不可用，疑似因为透明代理导致TLS握手指纹残留）  
**但是高质量节点访问ChatGPT是没问题的**，ipt2socks运行在Tproxy模式或可解决（但是unifiOS不支持）

**提前准备**  
一台提供Socks5Server科学上网的电脑（Mac为例，基本都支持）  
一台提供TailscaleServer/ipt2socks的软路由（UnifiOS 为例，能使用ssh基本就可以）

**步骤A：软路由端，先开启Tailscale Server搭建**  
> 如果是Unifi，需要先开启开启ssh和调试，Unifi 控制台 → 控制台设置 → 打开 SSH 和 高级调试功能

进入ssh安装Tailscale Server，[详细看官方参考](https://tailscale.com/kb/1131/unifi/) 
* 输入安装命令  
`curl -fsSL https://tailscale.com/install.sh | sh`
* 命令行会输出一个配置网页，例如，进去配置
* 配置tailscale server，输入以下命令设置  
#启动tailscale，并开启exit-node  
`sudo tailscale up --advertise-exit-node`

#可选项，关ipv6防泄漏-防止出现ipv6依然为CN的情况 
`sudo TS_NO_IPV6=1 tailscaled`

**步骤B：用户端配置（iPhone为例）**  
AppStore下载Tailscale，先测试连通性  
最终，所有步骤E完成后，设置如下（等下回来看）
```
EXIT NODE 
    On;节点选择软路由（提供了tailscale Server的主机）

Subnet routes
    Off
    #连接Socks5Server后就有局域网，此功能无需开启
```

```
DNS
    Off
    自定义;0.0.0.0,8.8.8.8（DNS由代理服务器解析,8.8.8.8保证连接稳定性）
```

**步骤C：tailscale网页控制配置**  
[tailscale网页控制台](https://login.tailscale.com/admin/machines)，进去登陆
```
tailscale服务器-选择软路由
    Edit route settings
        Exit Node On
        #不开启就无法连接Socks5服务
```

**步骤D：准备Socks5Server提供主机，（以Mac为例）**  
科学代理类型：
* 如果你使用Surge/Loon等软件，自带Socks5Server  

如果你的VPN是私有App，需要其他App开启Socks5服务器
* 推荐microsocks（特性支持完整），需要命令行安装（其他方式也可以）
#安装
`brew install microsocks`  
#启动
`microsocks -i 0.0.0.0 -p 1080`

**步骤E：软路由，安装ipt2socks（流量转发工具，iOS的出口流量，转发到Socks5）**
安装ipt2socks，[详细看官方参考](https://github.com/zfl9/ipt2socks/)
* 选择自己的CPU版本 https://github.com/zfl9/ipt2socks/releases/tag/v1.1.4
* 复制到，软路由的 data文件夹（ssh到软路由的SFTP上传即可）
* 电脑上创建.sh脚本，内容如下
    * 记得把[192.168.1.x]替换为你的Socks5 Server ip；
    *  我的配置特殊（UnifiOS），导致ipt2socks只支持REDIRECT，不支持TPROXY（内核精简导致的）；动手能力强的可优化
* 保存为`ipt2socks-Auto.sh`
* 并复制到 `/data`
```
#!/bin/sh
PATH=/usr/sbin:/usr/bin:/sbin:/bin:/data

# 定义端口变量
LISTEN_PORT=60080

# 启动/配置 ipt2socks ，并保持后台可用  
# 重要：记得把[192.168.1.x]替换为你的Socks5 Server ip
nohup /data/ipt2socks \
  --server-addr 192.168.1.x \
  --server-port 1080 \
  --listen-port $LISTEN_PORT \
  --listen-addr4 0.0.0.0 \
  --redirect \
  --tcp-only \
  > /dev/null 2>&1 &
#配置解析

# 清理旧规则
iptables -t nat -F IPT2S 2>/dev/null
iptables -t nat -X IPT2S 2>/dev/null
iptables -t nat -N IPT2S

# 添加 NAT 转发规则
iptables -t nat -A IPT2S -d 100.64.0.0/10 -j RETURN
iptables -t nat -A IPT2S -d 192.168.1.0/24 -j RETURN
iptables -t nat -A IPT2S -p tcp -j REDIRECT --to-port $LISTEN_PORT
iptables -t nat -A PREROUTING -i tailscale0 -j IPT2S  

```
运行脚本，一键启动ipt2socks 和 修改iptables  
`chmod +x /data/ipt2socks-Auto.sh`  
`sh /data/ipt2socks-Auto.sh`

**步骤F：测试和体验，客户端（iPhone为例）连接测试，在5G网络下功能是否正常**  
关闭Wi-Fi，5G网络下，iPhone开启tailscaleVPN（确保以上配置正确，特别是iOS App中DNS要正确设置，以及开EXIT NODE选软路由，可回到步骤C看）
* SMB连接Mac硬盘测试
* Googe/Youtube测试/ip测试
* PS5串流测试

**最后确保ipt2socks服务每次都开机重启，通过脚本ipt2socks-Auto.sh开机自启动实现（已做到开机重启脚本，就忽略）**  
因为UnifiOS的on_boot.d的目录不支持自启动，这里使用非常规方法（crontab保活）  
`(crontab -l 2>/dev/null; echo "* * * * * pgrep -x ipt2socks >/dev/null || /data/ipt2socks-Auto.sh >/dev/null 2>&1") | crontab -`
然后执行  `crontab -l`  检查一下是否存在添加的规则

**Done🎉**


> 🧩 参考资料：
> 1. Tailscale Unifi 安装指南  
>    https://tailscale.com/kb/1131/unifi/
> 2. ipt2socks 项目主页（支持 TPROXY/REDIRECT）  
>    https://github.com/zfl9/ipt2socks
> 3. Microsocks：轻型 Socks5 服务器  
   https://github.com/rofl0r/microsocks
