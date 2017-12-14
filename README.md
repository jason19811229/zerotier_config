# zerotier_config

场景说明：
- 两个路由器，一个部署在公司A，一个部署在AWS B
1. 公司内部任一PC，发出到google.com的请求
2. 该请求被直接路由到A
3. A根据DST IP，直接路由到B
4. B首先做SNAT，然后再路由出去
------------------------------------
5. google.com的服务器返回包给B
6. B完成反向SNAT后，把包路由给A
7. A根据DST IP，路由给发出请求的PC

command for manage iptable:
https://www.digitalocean.com/community/tutorials/how-to-list-and-delete-iptables-firewall-rules

- enable masquerading (eth0-WAN, eth1-LAN)

sudo iptables -t nat -A POSTROUTING -s 172.16.93.0/24 -o eth0 -j MASQUEREADE
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
- SNAT
# 表示在postrouting链上，将源地址为172.16.93.0/24网段的数据包的源地址都转换为10.0.0.1
iptables -t nat -A POSTROUTING -s 172.16.93.0/24  -j SNAT --to-source 10.0.0.1

sudo apt-get install iptables-persistent
sudo iptables-save > /etc/iptables/rules.v4


command to dump packages:
# 指定接口
sudo tcpdump -i eth0
# 指定IP地址(host)，可以辅加and , or ，!等逻辑符，以及src，dest等表示方向
# 捕获192.168.1.23发出或者收到的包
sudo tcpdump host 192.168.1.23
# 捕获192.168.1.23与192.168.1.104或者192.168.1.105之间往来的包
sudo tcpdump -i eth0 -A host 192.168.1.90 and \( 192.168.1.104 or 192.168.1.105 \)
# 指定从90发往104的包
sudo tcpdump -i eth0 -A src 192.168.1.90 and dst 192.168.1.104

===========================================
Design：
1. 网络配置
    内网信息： 192.168.96.0/26, PC=192.168.0.109
    RouterA:   LAN - 192.168.96.109/255.255.252.0 
                    zt0  - 10.147.17.13/255.255.255.0
    RouterB:   zt0  - 10.147.17.183
    DNS:        负责把白名单上的域名转化为IP地址

2. 路由和IPTable配置
2.0 内网PC机的配置
目标： - 访问Internet的包都路由到RouterA
配置： - DNS and Gateway 都指向192.168.0.100

2.1 DNS的安装和配置
安装：sudo apt install bind9 dnsutils
配置：https://help.ubuntu.com/lts/serverguide/dns-configuration.html
sudo /etc/init.d/bind9 restart

2.2 RouterA的配置
目标： - 允许包转发
          - 把从eth0收到的包都从zt0路由到RouterB
          - 把从zt0口收到的包都从eth0路由到内网PC机

/etc/sysctl.conf : net.ipv4.ip_forward=1
sudo sysctl -p /etc/sysctl.conf
cat /proc/sys/net/ipv4/ip_forward
# route add -host <destination IP address> gw <gateway IP address>
# sudo route add -net $NET netmask $MASK gw $GATEWAY
# sudo route del -net 0.0.0.0 gw 192.168.178.1 netmask 0.0.0.0 dev eth0
# ip route show 或者 route -n

sudo apt-get install iptables-persistent  #save和load iptables
sudo iptables -t nat -A POSTROUTING -o zt0 -j MASQUERADE



2.2 RouterB的配置
目标：- 把从zt0口收到的包都从eth0路由到Internet，并做SNAT
         - 把从eth0收到的包都从zt0路由到RouterA

$sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
$sudo iptables -A FORWARD -i eth0 -o zt0 -m state --state RELATED,ESTABLISHED -j ACCEPT
$sudo iptables -A FORWARD -i zt0   -o eth0 -j ACCEPT

$ sudo iptables -S
$ sudo iptables -t nat  -L

无需配置路由表。

最终结果：


3. 流程说明：
一个src=192.168.0.109 dst=google.com的包发出后，
1. 因为192.168.0.109的gateway和DNS都配置为192.168.0.100，首先去192.168.0.100找google.com对应的IP地址。
2. 因为192.168.0.100的bind9 DNS配置成为cache mode，它named.conf.optimize里面指定去找DNS=10.147.17.183寻求帮助，10.147.17.183也是一个cache mode的DNS(aws vm)，它最终找aws vm缺省的DNS来进行域名到IP的转换
3. 得到google.com的IP地址后，192.168.0.109因为Gateway=192.168.0.100，它把包发给RouteA
4. RouteA收到这个包后，根据配置的静态路由，发现下一跳是10.147.17.183
    - 做SNAT（src=10.147.17.13， dst=google.com）
    - 把包转发给下一跳前
5. RouteB收到包后，
    - 查路由表，发现得由缺省的网关发出，即从zt0到eth0的转发。
    - 查iptables，发现允许这样的转发
    - 查iptables，在从eth0发出去之前，要做SNAT（src=eth0， dst=google.com）
6. google.com的服务器收到包后，
    - 反馈信息返回给到10.147.17.183所在的机器的eth0接口（因为前面做了SNAT）
      （src=google.com， dst=eth0）
7. RouteB收到反馈包后，恢复目的地址（src=google.com， dst=10.147.17.13）
8. RouteB根据路由表，把包转发给RouteB（IP=10.147.17.13）
9. RouteA根据前面SNAT的记录，修改包为（src=google.com，dst=192.168.0.109）


