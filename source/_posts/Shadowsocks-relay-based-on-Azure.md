title: 在Azure搭建ShadowSocks国内跳板，并开启UDP转发。
date: 2015-2-5 1:11
tags:
- ShadowSocks
- iptables
- Azure
- NAT
categories: 日志

---
买BandwagonHost的VPS已经很久了，现在主要也是给朋友用。随着SS名声的变大，搬瓦工这个名字也在圈子里响当当了。用多了加上墙的变高，上海精品网等等的出现，速度越来越不理想。想来国内Azure还在免费，拿来搭个跳板机应该会有提高。
找到SS的[WIKI][1]，搭建一个转发只需要两行Iptables规则，但是根据作者说性能不如Haproxy。由于不需要负载平衡，所以直接照抄方案二的代码就完成了转发服务器的搭建。随之问题来了，UDP转发功能废了。无奈中，搜索得知Haproxy无法转发UDP包。不能转发UDP则不能借助SS-tunnel转发DNS来解决DNS投毒，更不能让走代理的流量获取最快的节点。这显然不能接受。然而正好咱不需要负载平衡，而且用户一只手掰得过来，想来内核级别的iptables性能应该不会差到哪去，并且这种无差别的转发一定是可以支持SS的UDP转发功能的。于是打定主意使用这个方案。
![iptables工作流程][2]
半路出家的我对复杂的iptables设置很是头疼，扒了文章了解后，大概得知我们的需求是由SNAT和DNAT合作完成的。简而言之就是我们的客户机上将服务器设置为我们的跳板机，则数据包发送到跳板机上，然后修改源IP和目标IP地址，重新转发给后面真实的SS服务器上，以此完成整个过程。
那么依样画葫芦，摘取一下：

    iptables -t nat -A PREROUTING -p tcp --dport 8388 -j DNAT --to-destination SS_VPS_IP:8388
    iptables -t nat -A POSTROUTING -p tcp -d SS_VPS_IP --dport 8388 -j SNAT --to-source Azure_IP
以上只转发TCP包，应该能够实现原本基础的功能了，可是经测试**无效**。检查了Azure网页中的端口映射,检查了iptables是否打开了转发,FORWARD表默认ACCEPT后都没有排除故障，只好用TCPDUMP抓取数据包。抓包后发现，在Azure的机器上，iptables出色的完成了预计的任务（如图）。可是在SS服务器上，则是一个包都没进得来。挠头之余，很是无奈啊。
![Azure服务器抓包][3]
![BWG上的抓包][4]
后来发现，**Azure在外层有一层NAT**，所给虚拟机的IP虽然是公网IP，但是不是绑定在虚拟机网卡上的IP。Azure的防火墙在收到数据包后进行一次NAT，转发给内部虚拟机**。出去的数据包也经过一次NAT**，之后才进行发送。
虽然在出咱虚拟机网卡的包的目标地址被正确的修改，指向了SS-vps，但是源地址是该虚拟机的外网IP（Azure叫他：公用虚拟 IP (VIP)地址），这个包在经国Azure的外围防火墙的时候被丢弃，因为认为这个包的源IP不是内部的服务器的。
修改规则后，测试成功。并且SS-tunnel正常。修改后代码如下：

    iptables -t nat -A PREROUTING -p tcp --dport 8388 -j DNAT --to-destination SS_VPS_IP:8388
    iptables -t nat -A PREROUTING -p udp --dport 8388 -j DNAT --to-destination SS_VPS_IP:8388
    iptables -t nat -A POSTROUTING -p tcp -d SS_VPS_IP --dport 8388 -j SNAT --to-source Azure内部 IP 地址
    iptables -t nat -A POSTROUTING -p udp -d SS_VPS_IP --dport 8388 -j SNAT --to-source Azure内部 IP 地址

这个事情真的是会的不难，难的不会，加上有个暗坑（Azure的防火墙），让本来不复杂的事情变得困难了一些。不过好处是，在给了我动力和机会实践学习了Iptables的一些概念和命令。对表，链，规则有了基本的认识学习。收益颇多，也做个笔记。


  [1]: https://github.com/shadowsocks/shadowsocks/wiki/Setup-a-Shadowsocks-relay
  [2]: /images/3/iptables.gif
  [3]: /images/3/Azure.png
  [4]: /images/3/BWG.png
