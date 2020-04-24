# openwrt-iptv

## OpenWrt

### LAN

- 编辑网络接口`lan`

```shell script
vi /etc/config/netwrok
config interface 'lan'
	option ifname 'eth0'
	option proto 'static'
	option ipaddr '192.168.100.1'
	option netmask '255.255.255.0'
```

- 开启接口`lan`的dhcp

```shell script
vi /etc/config/dhcp
config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option force '1'
```

### Guest

- 新建网络接口`guest`

```shell script
vi /etc/config/network
config interface 'guest'
	option ifname 'eth3'
	option proto 'static'
	option ipaddr '192.168.101.1'
	option netmask '255.255.255.0'
```

- 开启接口`guest`的`dhcp`服务

```shell script
vi /etc/config/dhcp
config dhcp 'guest'
	option interface 'guest'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option force '1'
```

- 编辑防火墙

```shell script
vi /etc/config/firewall
config zone
	option name 'guest'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option network 'guest'

config forwarding
	option src 'guest'
	option dest 'wan'

config rule
	option name 'Allow-Guest-DNS'
	option src 'guest'
	option dest_port '53'
	option proto 'udp'
	option target 'ACCEPT'

config rule
	option name 'Allow-Guest-DHCP'
	option src 'guest'
	option dest_port '67-68'
	option target 'ACCEPT'

config rule
	option name 'Allow-Guest-IPTV'
	option src 'guest'
	option dest_port '4023'
	option target 'ACCEPT'
```

## IPTV

### 拨号

- 新建网络接口`iptv`

```shell script
vi /etc/config/netwrok
config interface 'iptv'
	option ifname 'eth2'
	option proto 'dhcp'
	option hostname '******'
	option vendorid '******'
	option macaddr '00:00:00:00:00:00'
	option metric '2'
```

- 编辑防火墙

```shell script
config zone
	option name 'iptv'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option masq '1'
	option mtu_fix '1'
	option network 'iptv'

config rule
	option name 'Allow-IPTV-IGMP'
	option src 'iptv'
	option proto 'igmp'
	option target 'ACCEPT'

config rule
	option name 'Allow-IPTV-UDP'
	option src 'iptv'
	option proto 'udp'
	option dest_ip '224.0.0.0/4'
	option target 'ACCEPT'
```

### IPTV to IPTV-LAN

使用组播`igmpproxy`，通过`LAN`连接机顶盒，`WiFi`卡顿。

- 新建网络接口`iptv_lan`

```shell script
vi /etc/config/network
config interface 'iptv_lan'
	option ifname 'eth4'
	option proto 'static'
	option type 'bridge'
	option igmp_snooping '1'
	option ipaddr '192.168.102.1'
	option netmask '255.255.255.0'
```

- 开启接口`iptv_lan`的`dhcp`服务

```shell script
vi /etc/config/dhcp
config dhcp 'iptv_lan'
	option interface 'iptv_lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option force '1'
```

- 编辑防火墙

```shell script
vi /etc/config/firewall
config zone
	option name 'iptv_lan'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option network 'iptv_lan'

config forwarding
	option src 'iptv_lan'
	option dest 'iptv'

config rule
	option name 'Allow-IPTV-LAN-DNS'
	option src 'iptv_lan'
	option dest_port '53'
	option proto 'udp'
	option target 'ACCEPT'

config rule
	option name 'Allow-IPTV-LAN-DHCP'
	option src 'iptv_lan'
	option dest_port '67-68'
	option target 'ACCEPT'
```

- 安装并配置`igmpproxy`

```shell script
opkg update
opkg install igmpproxy

vi /etc/config/igmpproxy
config igmpproxy
	option quickleave 1

config phyint
	option network iptv
	option zone iptv
	option direction upstream
	list altnet 0.0.0.0/0

config phyint
	option network iptv_lan
	option zone iptv_lan
	option direction downstream

vi /etc/sysctl.conf
net.ipv4.conf.all.force_igmp_version=2

service igmpproxy enable
service igmpproxy start
```

### IPTV to WiFi

使用组播转单播`udpxy`，通过`WiFi`连接播放设备。

- 编辑防火墙

```shell script
vi /etc/config/firewall
config forwarding
	option src 'lan'
	option dest 'iptv'

config forwarding
	option src 'guest'
	option dest 'iptv'
```

- 安装并配置`udpxy`

```shell script
opkg update
opkg install udpxy luci-app-udpxy
  
vi /etc/config/udpxy
config udpxy 'lan'
	option disabled '0'
	option respawn '1'
	option verbose '0'
	option status '1'
	option bind '0.0.0.0'
	option port '4022'
	option source 'eth2'
	# option max_clients '3'
	option log_file '/var/log/udpxy'
	# option buffer_size '4096'
	# option buffer_messages '-1'
	# option buffer_time '-1'
	# option nice_increment '0'
	# option mcsub_renew '0'

config udpxy 'guest'
	option disabled '0'
	option respawn '1'
	option verbose '0'
	option status '1'
	option bind '192.168.101.1'
	option port '4023'
	option source 'iptv'
	option log_file '/var/log/udpxy_guest'

service udpxy enable
service udpxy start
```

### 提取组播地址

- 从str到行尾

```regexp
str.*
```

- 从str到行首

```regexp
^.*?str
```
