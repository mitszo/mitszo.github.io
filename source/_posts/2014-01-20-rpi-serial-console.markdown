---
layout: post
title: "Raspberry PiにUSB-Serial接続する"
date: 2014-01-20 22:00
comments: true
categories: RaspberryPi tips
---


# Raspberry PiにUSB-Serial接続する

いちいちsshとかも面倒なので、USB-Serial接続できるようにします。
Mac前提。Linuxでもほとんど同じ。Windowsはわかりません。

## 用意するもの

* USB-Serial変換アダプタ
* ピンやコネクタ類

僕が買った変換アダプタは[Sparkfunの3.3Vの確かコレ](http://www.switch-science.com/catalog/343/)


## Raspbianの準備
http://www.raspberrypi.org/downloads
からダウンロード。
最新の2014-01-09ではうまく起動しなかった。以下、手元にあった2013-09-25で試した。
過去のイメージは[このあたり](http://downloads.raspberrypi.org/raspbian/images/)から。

Raspbianでしか試してません。ddする前にSDカード確認。
```
$ mount
/dev/disk1 on / (hfs, local, journaled)
devfs on /dev (devfs, local, nobrowse)
map -hosts on /net (autofs, nosuid, automounted, nobrowse)
map auto_home on /home (autofs, automounted, nobrowse)
/dev/disk2s1 on /Volumes/Untitled (msdos, local, nodev, nosuid, noowners)
```
`/dev/disk2` ですね。
```
$ sudo umount /Volumes/Untitled/
Password:
$ mount
/dev/disk1 on / (hfs, local, journaled)
devfs on /dev (devfs, local, nobrowse)
map -hosts on /net (autofs, nosuid, automounted, nobrowse)
map auto_home on /home (autofs, automounted, nobrowse)
```
アンマウントしておいて、

```
$ unzip 2013-09-25-wheezy-raspbian.zip
Archive:  2013-09-25-wheezy-raspbian.zip
  inflating: 2013-09-25-wheezy-raspbian.img
```
zipファイルを展開して、ddで書き込み。
```
$ pv 2013-09-25-wheezy-raspbian.img | sudo dd of=/dev/disk2 bs=4096
1.86GiB 0:04:15 [ 7.6MiB/s] [                                <=>               ]70% ETA 0:02:08
```
pvコマンド使うと進捗表示されてわかりやすい。
（Macならportsで入る。Debian系ならaptで。）

## USB-Serial変換アダプタの用意

ピンはハンダ付けするなりしておいて、RPiの[GPIO](http://elinux.org/RPi_Low-level_peripherals)とUSB-Serial変換ボードの、

* TxとRx
* RxとTx
* GNDとFND

をつなぎます。
こんな感じ。![ピンをつないだ図](images/IMG_3828.jpg)

できたら必要なドライバをインストールして、MacとUSBでつないでおきます。

## コンソール接続

やりかたはいろいろあるでしょうが、screenが無難な気がする。
電源入れる前にターミナルで実行します。

```
screen /dev/tty.usbserial-A600e1CH 115200,cs8,cstopb
```

## 電源ON!

SDカードをさして、Raspberry Piを電源につなげばscreenでブートの様子が見られます。
![screen](images/screen.png)

loginプロンプトが出たら`pi/raspberry`でログイン。
後はssh接続したときと一緒。
