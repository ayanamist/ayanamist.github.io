---
layout: post
title: 连vpn后wsl中dns不正确的workaround
---

因为wsl中[#1350](https://github.com/Microsoft/WSL/issues/1350)这个bug的存在，导致即使连上vpn，wsl中的resolv.conf文件里依旧是老的dns地址，导致解析出问题。

官方给的[workaround](https://msdn.microsoft.com/en-us/commandline/wsl/troubleshooting#bash-loses-network-connectivity-once-connected-to-a-vpn)非常糟糕，需要大量的手工操作。

不记得在哪里看到的一个comment，提示可以利用wsl中执行exe的能力，来做一个自动纠正：在`~/.bashrc`中追加以下

```bash
(
VPN_DNS=$(ipconfig.exe /all | fgrep -A 50 AnyConnect | fgrep "DNS Servers" | head -1 | awk -F ': ' '{print $2}' | sed 's/\r//')
[[ -n $VPN_DNS ]] && sudo sed -i "1 a nameserver $VPN_DNS" /etc/resolv.conf
)
```

方法比较粗暴，就是看AnyConnect的网卡上有没有DNS，有的话取第一个，加到resolv.conf里。当然写个python脚本把活可以干的更精细一些，不过这样一小段bash也足够我使用了。
