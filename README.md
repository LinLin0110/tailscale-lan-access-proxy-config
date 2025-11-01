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
⚠️配置需要用命令行，有几个关键的互相依赖的设置（需要手动调）  

**提前准备**  
一台提供Socks5Server科学上网的电脑（Mac为例，基本都支持）  
一台提供TailscaleServer/VPN穿透客户端的路由器（OpenWRT为例，其他支持Docker/iptables的路由器均可参考）  
一台支持SSH联机的路由器（保证可执行iptables/ip -6 rule添加、自定义system服务）  

**整体思路**  
1. 宿主电脑（Mac）连接到SSR/V2RAY这类科学服务器  
2. 在宿主电脑（Mac）运行Socks5server，暴露在Mac内网地址（例如192.168.0.33:1086）供路由器转发使用  
3. 路由器配置[Tailscale]并暴露服务的子网（例：192.168.0.0/24）  
4. 路由器运行ipt2socks，将本地访问流量转发到Mac暴露的Socks5server，达到只转发匹配的流量到代理  
5. 可使用iptables rule（例如iptables -t nat）将匹配的局域网访问从路由器转发到代理

## Host（Mac）端设置
1. 在Mac上安装Socks5Server（比如使用`proxychains-ng`或`ssh -D`搭建本地代理）  
2. 配置Socks5Server监听本地地址（例如192.168.0.33），端口1086，并确保其他设备可访问  

## Router（路由器）端设置
1. 安装Tailscale客户端并加入VPN网络  
2. 设置Tailscale子网（192.168.0.0/24）并启用子网路由器
3. 安装ipt2socks工具，根据路由器架构选择适配版本  
4. 配置路由器iptables，转发匹配流量到ipt2socks代理  
5. 设置路由器system服务，让代理在开机时自动启动

## 访问方式
1. 在外使用5G/4G网络，通过Tailscale VPN连接到Router暴露的子网  
2. 代理的流量将通过Router->Mac->Socks5Server->科学互联网  
3. PS5/NAS/SMB等设备可通过网络加速实现串流/访问  

## 注意事项
* Mac的Socks5Server需要固定内网IP且正常运行  
* 路由器需要支持iptables/iproute2/SSH  
* 代理规则需要根据个人需求自定义，如访问哪些域名和端口

## FAQ
1. **如何确保PS5串流顺畅？**  
   首先需要在路由器/局域网中保证PS5走代理的延迟及带宽良好。其次建议通过Socks5Server连接到高速代理，以减少延迟和丢包。  
2. **是否能与现有VPN共存？**  
   可以，在Mac端提供的Socks5Server可以直接与现有VPN代理共存，无需额外配置。  
3. **是否支持IPv6访问？**  
   根据路由器和Tailscale版本不同，部分配置可能需要额外设置IPv6路由；然而在大多数情况下，IPv6线路将自动通过Tailscale覆盖。
