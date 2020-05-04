# openwrt-iptv

## OpenWrt

### LAN

- 编辑网络接口`/etc/config/netwrok`

```shell script
config interface 'lan'
	option ifname 'eth0'
	option proto 'static'
	option ipaddr '192.168.100.1'
	option netmask '255.255.255.0'
```

- 开启接口dhcp`/etc/config/dhcp`

```shell script
config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option force '1'
```

- 编辑防火墙`/etc/config/firewall`

```shell script
config zone
	option name 'lan'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'
	option network 'lan'

config forwarding
	option src 'lan'
	option dest 'wan'
```

### WAN

- 新建网络接口`/etc/config/netwrok`

```shell script
config interface 'wan'
	option ifname 'eth1'
	option proto 'pppoe'
	option username 'username'
	option password 'password'
	option ipv6 '0'
	option metric '10'
```

- 编辑防火墙`/etc/config/firewall`

```shell script
config zone
	option name 'wan'
	option input 'REJECT'
	option output 'ACCEPT'
	option forward 'REJECT'
	option masq '1'
	option mtu_fix '1'
	option network 'wan'
```

### IPTV

- 新建网络接口`/etc/config/netwrok`

```shell script
config interface 'iptv'
	option ifname 'eth2'
	option proto 'dhcp'
	option hostname 'hostname'
	option vendorid 'vendorid'
	option macaddr '00:00:00:00:00:00'
	option metric '20'
```

- 编辑防火墙`/etc/config/firewall`

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

### Guest

- 新建网络接口`/etc/config/netwrok`

```shell script
config interface 'guest'
	option ifname 'eth3'
	option proto 'static'
	option ipaddr '192.168.101.1'
	option netmask '255.255.255.0'
```

- 开启接口dhcp`/etc/config/dhcp`

```shell script
config dhcp 'guest'
	option interface 'guest'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option force '1'
```

- 编辑防火墙`/etc/config/firewall`

```shell script
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
```

## IPTV

### udpxy to LAN/Guest

使用组播转单播`udpxy`，通过`WiFi`连接播放设备。

- 编辑防火墙`/etc/config/firewall`

```shell script
config forwarding
	option src 'lan'
	option dest 'iptv'

config forwarding
	option src 'guest'
	option dest 'iptv'

config rule
	option name 'Allow-LAN-Udpxy'
	option src 'lan'
	option dest_port '4022'
	option target 'ACCEPT'

config rule
	option name 'Allow-Guest-Udpxy'
	option src 'guest'
	option dest_port '4023'
	option target 'ACCEPT'
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
	option log_file '/var/log/udpxy_lan'
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
	option source 'eth2'
	option log_file '/var/log/udpxy_guest'

service udpxy enable
service udpxy start
```

### igmpproxy to LAN

使用组播`igmpproxy`，通过`LAN`连接机顶盒，使用`WiFi`卡顿。

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
	option network lan
	option zone lan
	option direction downstream

vi /etc/sysctl.conf
net.ipv4.conf.all.force_igmp_version=2

service igmpproxy enable
service igmpproxy start
```

- 安装`MWAN3`

```shell script
opkg update
opkg install mwan3 luci-app-mwan3

vi /etc/config/mwan3
config rule 'iptv_igmp'
	option dest_ip '239.254.200.0/23'
	option proto 'all'
	option sticky '0'
	option use_policy 'iptv_only'

config rule 'iptv_rtsp'
	option dest_ip '192.168.24.0/22'
	option proto 'all'
	option sticky '0'
	option use_policy 'iptv_only'

config rule 'wan_rule'
	option proto 'all'
	option sticky '0'
	option use_policy 'wan_only'
```
