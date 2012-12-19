---
layout: post
title: "Raspberry PiでモバイルWiFi NAS"
date: 2012-12-19 22:10
comments: true
categories: RaspberryPi network mobile
---

Raspberry Pi
------------

[Raspberry Pi](http://www.raspberrypi.org/) は、700MHzのARMと256MBのメモリー、SDカードスロットやUSBポートを搭載した手のひらサイズPC。
HDMIもついてるのでモニターとキーボード、マウスをつないでデスクトップな使い方も可能。でもやっぱりモッサリ感はぬぐえない。


目的、というか動機
-------------------

「iPad 16GB買ったけどやっぱり容量きびしいなあ」と言ってると、友人にAir Driveなるモノをすすめられた。

買ったまま転がってたRaspberry PiはUSBでの電源供給で動作するのでモバイルバッテリーでも稼働するはず。
無線で繋がるようにしてやれば、同じことできるよね？

ということでストレージを積んで、モバイルNASにしてみる。


材料
----

* Raspberry Pi
* SDHCカード 4GB
* USBメモリ（FAT）
* USB無線LANアダプタ（Buffalo WLI-UC-GN）

無線LANアダプタはAPモードが使えるか、Linux用のドライバがあるかは事前に調べた方が無難かも。

起動ディスク
------------

[Raspbian](http://www.raspbian.org/)を使う。中身はまんまDebian。

1. [Raspberr Pi Downloads](http://www.raspberrypi.org/downloads)から *2012-10-28-wheezy-raspbian.zip* をダウンロード。
2. zipファイル展開して出てきた.imgファイルを[説明](http://elinux.org/RPi_Easy_SD_Card_Setup)の通りdd。

「空き領域にあわせてパーティションを拡大」な感じの記事があるけど、後述のツールでできるので気にせずスルー。


起動
----

HDMIでディスプレイとキーボードつなぐか、イーサネットつないどいて払い出されるアドレス確認してコンソール使えるようにしておく。USB無線LANモジュール、USBメモリ、SDカードをさしてRaspberry Piを起動。

起動してログインすると、「セッティングがまだ終わってないからraspi-config起動しろ」なメッセージが表示されるので素直に従う。ここでパーティション拡張とかできるので、設定し終えたら再起動しておく。

sshでログインする時は、ユーザー名「pi」パスワード「raspberry」。


ネットワークまわりの設定と無線AP化
-----------------------------------

「無線ルーターにUSBドライブ共有機能がついたもの」を目指すので、有線はDHCPからアドレスもらって、無線はアドレス固定でDHCPサービスする形にする。

### 無線LANアダプタが認識されてるか確認

```
$ lsusb
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 0424:9512 Standard Microsystems Corp.
Bus 001 Device 003: ID 0424:ec00 Standard Microsystems Corp.
Bus 001 Device 004: ID 8644:800e 
Bus 001 Device 005: ID 0411:015d BUFFALO INC. (formerly MelCo., Inc.) WLI-UC-GN Wireless LAN Adapter [Ralink RT3070]

$ iwconfig
lo        no wireless extensions.

wlan0     IEEE 802.11bgn  ESSID:off/any 
          Mode:Managed  Access Point: Not-Associated   Tx-Power=20 dBm  
          Retry  long limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
        
eth0      no wireless extensions.
```


### ネットワーク設定にwlan0を追加

/etc/network/interfaces

```
auto lo

iface lo inet loopback
iface eth0 inet dhcp

auto wlan0
iface wlan0 inet static
	address 192.168.10.1
	netmask 255.255.255.0
	gateway 192.168.10.1

```

wlan0を有効にする。

```
$ sudo ifdown wlan0
$ sudo ifup wlan0
```


### hostapdを設定して無線AP化

[hostapd](http://w1.fi/hostapd/)

```
$ sudo apt-get install hostapd
```

設定ファイル作成。

/etc/hostapd/hostapd.conf

```
interface=wlan0
driver=nl80211
ctrl_interface=/var/run/hostapd
ctrl_interface_group=0
ssid=MyRaspberryPi
country_code=JP
hw_mode=g
channel=2
beacon_int=100
max_num_sta=5
macaddr_acl=0
auth_algs=1
wpa=1
wpa_passphrase=wpasecret
wpa_key_mgmt=WPA-PSK
```


hostapd自動起動設定
/etc/default/hostapdのDAEMON_CONFのコメントを外し設定ファイルのパスを書き込む。

```
# Defaults for hostapd initscript
#
# See /usr/share/doc/hostapd/README.Debian for information about alternative
# methods of managing hostapd.
#
# Uncomment and set DAEMON_CONF to the absolute path of a hostapd configuration
# file and hostapd will be started during system boot. An example configuration
# file can be found at /usr/share/doc/hostapd/examples/hostapd.conf.gz
#
DAEMON_CONF="/etc/hostapd/hostapd.conf"

# Additional daemon options to be appended to hostapd command:-
# 	-d   show more debug messages (-dd for even more)
# 	-K   include key data in debug messages
# 	-t   include timestamps in some debug messages
#
# Note that -B (daemon mode) and -P (pidfile) options are automatically
# configured by the init.d script and must not be added to DAEMON_OPTS.
#
#DAEMON_OPTS=""
```


### DHCPサービスを設定
ここではdnsmasqを使う。dhcp3-serverでもいいのでお好みで。

```
$ sudo apt-get install dnsmasq
```

dnsmasqはWiFi側(wlan0)のみ、対象にする。

/etc/dnsmasq.conf
```
no-dhcp-interface=eth0
dhcp-range=192.168.10.50,192.168.10.150,255.255.255.0,12h
dhcp-option=3,192.168.10.1
dhcp-option=option:router,192.168.10.1
```


### 接続確認

ここまでで無線APとして動作するはず。
dnamasqとhostapdを再起動して接続してみる。
IPアドレスが払い出されてつながってればOK。



ファイル共有設定
----------------

ファイル共有にはSambaを使う。FATなストレージをUSBに接続して中身をまるごと共有する。


### USBドライブの自動マウント

```
$ sudo apt-get install usbmount
```
/media/usb0 とかできてる。

自動マウント時のオプション（権限とか）は、
/etc/usbmount/usbmount.conf
で設定。

vfatな場合の設定（パーミッションやcodepage等）を`FS_MOUNTOPTIONS`に追加しておく。

```
ENABLED=1
...
FS_MOUNTOPTIONS="-fstype=vfat,gid=pi,dmask=0000,fmask=0111,codepage=932,iocharset=utf8"
```

マウントポイントどうしようかと思いつつ、どうせひとつしか差さないので放置。Sambaの設定でも /media/usb0 にマウントされる前提にする。


### Sambaの設定


Sambaでは上述の通り /media/usb を共有。


```
[global]
...
   security = user

#======================= Share Definitions =======================
[usbdrive]
   comment = USB Drive
   browseable = yes
   path = /media/usb
   create mask = 0775
   directory mask = 0775
   read only = no
   guest ok = no
```

動作確認
--------

Raspberry Piに無線LANモジュールとFATフォーマットしたUSBメモリを接続して再起動。

無線APとして、「MyRaspberryPi」が見えたら、WPA2 Personalで設定したパスワード「wpasecret」を入力して接続。
接続するクライアントはDHCP設定。
つながったらdnsmasqでIPアドレスが払い出されるハズ。
Sambaクライアントで192.168.10.1/usbdriveに接続できるか確認。



