---
title: "ラズパイでPiAwareとFR24にfeedするベストプラクティス（だと思う）"
emoji: "✈️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [raspberrypi]
published: false
---

# ADS-Bフィードにおける問題があったので解決したった！

簡単に言うと[FlightAware](https://ja.flightaware.com/)と[Flight Radar 24](https://www.flightradar24.com/)の双方にデータを*feed*しようとすると、両社のソフトウェアが相互干渉してしまい必要なプロセスが全て起動しているのに、肝心の`dump1090`コンポーネントがデータを供給しなくなってしまったんです。

つまり***feed*しようにもデータ送信以前の問題**だった訳です。

で、ネットの海を彷徨っていると、まぁ情報が玉石混交で**これでもか！**と言う量のサイトがありました。で*Try and Error*を繰り返して何とかなったので、これは**自分メモ**としても大事だけど、ひょっとしたら悩める善意の*feeder*の皆さんのお役に立てるかも？と思い記録する事にしました。

## SDカードのフォーマット

**後で書く**


## 自分で環境構築

###　下準備

```bash
# SSHでラズパイへログイン
$ ssh pi
Linux pi 5.15.32-v7+ #1538 SMP Thu Mar 31 19:38:48 BST 2022 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jun 24 15:22:54 2022 from 192.168.1.101

# 初期状態だと、ラズパイさんのvimがvi互換モードになっているので
# Insertモードでカーソルキーが使えないのは不便なので治しておきます。
# ついでに他の設定も楽になりたいからしておきます。
## 下の表を参照してね。

masaru@feeder:~ $ vi .vimrc
```

```conf
set nocompatible
set backspace=indent,eol,start
set autoindent
set smartindent
set tabstop=2
set shiftwidth=2
```

|項目|意味|
|:--|:--|
|set nocompatible|vi互換モード停止|
|set backspace=indent,eol,start|バックスペースの挙動|
|set autoindent|自動インデント|
|set smartindent|C言語の構文に則った自動インデント|
|set tabstop=2|タブ幅をスペース2つに|
|set shiftwidth=2|自動インデント幅を2つに|

```bash
# 後々楽したいので、aliasを設定する
masaru@feeder:~ $ vi .bashrc
  alias mkdir='mkdir -p'
  alias la='ls -la'
  alias rf='rm -rf'
masaru@feeder:~ $ source .bashrc

# ラズパイは初期化した時にrootにパスワードが設定されていない
# suを使う為にパスワードを設定する
masaru@feeder:~ $ sudo passwd root
New password:
Retype new password:
passwd: password updated successfully

# パッケージ更新・再起動
# これはラズパイを初期化する時のお作法
masaru@feeder:~ $ sudo apt-get update
masaru@feeder:~ $ sudo apt-get upgrade
masaru@feeder:~ $ sudo reboot
```

しばらく待ってネットワーク越しに見られるようになったら次に進めます。


**要注意**

色々なサイトに初期設定についての記述がありますが`sudo rpi-update`なる記述は**非推奨**ですので、しないでくださいね。

今は*deprecated*になっていますが[Git Hub](https://github.com/)の[リポジトリ](https://github.com/Hexxeh/rpi-update)に**なぜ推奨していないか**と言う理由が説明されています。

ただ…オッチャンは**チャレンジャー**や**人柱希望者**が大好きなので***At Your Own Risk***で試すのは止めないです。

今オッチャンも*iOS 16.2 beta 2, iPad OS 16.2 beta 2, watchOS 9 beta 2*で難儀してますしね💦でも、さすがにメインマシンで*macOS 13 beta 2*を試す程愚かではありません。

では元に戻って…

### IPアドレスを固定する

多くの方がRouterで固定IPをラズパイのMACアドレスに振ってるから大丈夫！と仰ってますが、確実性を求めるならラズパイでも設定すべきです。

```powershell
masaru@feeder:~ $ sudo vi /etc/dhcpcd.conf
```

最下段あたりに

```terraform
interface wlan0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1

interface eth0
static ip_address=192.168.1.101/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
```

記述したら

```shell
masaru@pi:~ $ sudo reboot
```

でシステムを再起動します。再起動したら念のために`ifconfig`で確認しておきます。


```bash
masaru@pi:~ $ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.101  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 2400:2413:9682:3600:ed4f:29f8:c32c:ddca  prefixlen 64  scopeid 0x0<global>
        inet6 fe80::7ff6:a548:204a:95c9  prefixlen 64  scopeid 0x20<link>
        ether b8:27:eb:a6:ae:8a  txqueuelen 1000  (Ethernet)
        RX packets 88193  bytes 17277854 (16.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1840  bytes 210587 (205.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 10  bytes 1546 (1.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 10  bytes 1546 (1.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.169.1.100  netmask 255.255.255.0  broadcast 192.169.1.255
        inet6 2400:2413:9682:3600:ccfd:48d4:d7fb:c545  prefixlen 64  scopeid 0x0<global>
        inet6 fe80::386b:31d5:fa92:4bfb  prefixlen 64  scopeid 0x20<link>
        ether b8:27:eb:f3:fb:df  txqueuelen 1000  (Ethernet)
        RX packets 726  bytes 121892 (119.0 KiB)
        RX errors 0  dropped 1  overruns 0  frame 0
        TX packets 301  bytes 46362 (45.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

よしよし。

### 時間は合ってる？

念のために日時設定が合っているかを確認します。

```powershell
masaru@pi:~ $ date
Mon 19 Sep 15:35:32 JST 2022
```

一応NTPとの同期を確認

```bash
masaru@pi:~ $ timedatectl status
               Local time: Fri 2022-07-22 16:47:47 JST
           Universal time: Fri 2022-07-22 07:47:47 UTC
                 RTC time: n/a
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
masaru@pi:~ $ timedatectl show -a
Timezone=Asia/Tokyo
LocalRTC=no
CanNTP=yes
NTP=yes
NTPSynchronized=yes
TimeUSec=Fri 2022-07-22 16:48:29 JST
RTCTimeUSec=
```

よしよし。

#### RTC

余談ですが、*RTC*は*Real Time Clock*の頭字語で別名ハードウェアクロックと言います。

通常のPCには元々基盤に組み込まれています。だから再起動の度に時間合わせの必要がありません。更にオフラインでNTPサーバーを使えなくても時刻が保持されるのは*RTC*のお陰です。

ですが、ラズパイには初期状態では存在していません。ネットから切り離した環境で使う方は*RTC*モジュールを取りつけなければ、時間の管理ができませんし、再起動の度に日時設定をしなくてはなりません。

### 私にRTCは不要

今回の目的は`ads-b`のデータフィードですから、インターネットに常時接続されているのが前提です。

なので*RTC*は不要です。

## PiAwareのインストールと設定

### PiAwareから

[公式](https://ja.flightaware.com/adsb/piaware/install)に従って最新のPiAwareをインストールします。

```bash
masaru@pi:~ $ wget https://ja.flightaware.com/adsb/piaware/files/packages/pool/piaware/p/piaware-support/piaware-repository_7.2_all.deb
masaru@pi:~ $ sudo dpkg -i piaware-repository_7.2_all.deb
# 念のためにaptをアップデートしてからPiAwareをインストール
masaru@pi:~ $ sudo apt-get update
masaru@pi:~ $ sudo apt-get install piaware
# PiAwareの自動アップデートと手動アップデートを有効化する
masaru@pi:~ $ sudo piaware-config allow-auto-updates yes
masaru@pi:~ $ sudo piaware-config allow-manual-updates yes
```

### 次にdump1090を

```bash
masaru@pi:~ $ sudo apt-get install dump1090-fa
# dump978=978MHzは主に小型機がエアロマリタイム用にアメリカで使っていました
# 近年はセスナやヘリなどの事故の多発を受けて日本でも試験導入をしています。
# 入れなくても通常のADS-Bのフィードには影響はありませんので飛ばしても構わないです。
masaru@pi:~ $ sudo apt-get install dump978-fa
masaru@pi:~ $ sudo reboot
```

### 起動したら状態を見てみる

```bash
masaru@pi:~ $ systemctl status piaware
● piaware.service - FlightAware ADS-B uploader
     Loaded: loaded (/lib/systemd/system/piaware.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-09-19 16:00:58 JST; 1min 26s ago
       Docs: https://flightaware.com/adsb/piaware/
   Main PID: 1684 (piaware)
      Tasks: 3 (limit: 1598)
        CPU: 2.893s
     CGroup: /system.slice/piaware.service
             ├─1684 /usr/bin/piaware -p /run/piaware/piaware.pid -plainlog -statusfile /run/piaware/status.json
             └─1697 /usr/lib/piaware/helpers/faup1090 --net-bo-ipaddr localhost --net-bo-port 30005 --stdout

Sep 19 16:01:02 pi sudo[1695]:  piaware : PWD=/ ; USER=root ; COMMAND=/bin/netstat --program --tcp --wide --all --numeric
Sep 19 16:01:02 pi sudo[1695]: pam_unix(sudo:session): session opened for user root(uid=0) by (uid=999)
Sep 19 16:01:02 pi sudo[1695]: pam_unix(sudo:session): session closed for user root
Sep 19 16:01:02 pi piaware[1684]: ADS-B data program 'dump1090-fa' is listening on port 30005, so far so good
Sep 19 16:01:02 pi piaware[1684]: Starting faup1090: /usr/lib/piaware/helpers/faup1090 --net-bo-ipaddr localhost --net-bo-port 30005 --stdout
Sep 19 16:01:02 pi piaware[1684]: Started faup1090 (pid 1697) to connect to dump1090-fa
Sep 19 16:01:02 pi piaware[1684]: UAT support disabled by local configuration setting: uat-receiver-type
Sep 19 15:55:30 pi piaware[679]: logged in to FlightAware as user guest
Sep 19 15:55:30 pi piaware[679]: my feeder ID is 037aa382-7983-4f39-8674-309f5e348abd
Sep 19 16:01:33 pi piaware[1684]: 0 msgs recv'd from dump1090-fa; 0 msgs sent to FlightAware
```

細かい設定は後にするとして…これが初回の[FlightAware](https://ja.flightaware.com/)へのfeedでしたら[このページ](https://ja.flightaware.com/adsb/piaware/install)の**6 FlightAware.comでPiAwareクライアントを申し込む**以降の手順に従ってください。

私は以前にfeedしていてfeeder IDを持っていますので、そのIDに書き換えます。

```bash
masaru@pi:~ $ sudo piaware-config feeder-id 5baxxxxx-1yya-4zz4-abcd-1a2bcd3d4e5f6d
# 書き換えたらPiAwareを再起動します。
masaru@pi:~ $ sudo systemctl restart piaware
# 少し待ってから確認します。
masaru@pi:~ $ systemctl status piaware
● piaware.service - FlightAware ADS-B uploader
     Loaded: loaded (/lib/systemd/system/piaware.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2022-09-19 16:00:58 JST; 11min ago
       Docs: https://flightaware.com/adsb/piaware/
   Main PID: 1684 (piaware)
      Tasks: 3 (limit: 1598)
        CPU: 4.746s
     CGroup: /system.slice/piaware.service
             ├─1684 /usr/bin/piaware -p /run/piaware/piaware.pid -plainlog -statusfile /run/piaware/status.json
             └─1697 /usr/lib/piaware/helpers/faup1090 --net-bo-ipaddr localhost --net-bo-port 30005 --stdout

Sep 19 16:01:02 pi piaware[1684]: Starting faup1090: /usr/lib/piaware/helpers/faup1090 --net-bo-ipaddr localhost --net-bo-port 30005 --stdout
Sep 19 16:01:02 pi piaware[1684]: Started faup1090 (pid 1697) to connect to dump1090-fa
Sep 19 16:01:02 pi piaware[1684]: UAT support disabled by local configuration setting: uat-receiver-type
Sep 19 16:01:03 pi piaware[1684]: logged in to FlightAware as user hogefuga
Sep 19 16:01:03 pi piaware[1684]: my feeder ID is 5baxxxxx-1yya-4zz4-abcd-1a2bcd3d4e5f6d
Sep 19 16:01:33 pi piaware[1684]: 0 msgs recv'd from dump1090-fa; 0 msgs sent to FlightAware
Sep 19 16:06:33 pi piaware[1684]: 0 msgs recv'd from dump1090-fa (0 in last 5m); 0 msgs sent to FlightAware
Sep 19 16:08:35 pi piaware[1684]: piaware received a message from dump1090-fa!
Sep 19 16:10:08 pi piaware[1684]: piaware has successfully sent several msgs to FlightAware!
Sep 19 16:11:33 pi piaware[1684]: 40 msgs recv'd from dump1090-fa (40 in last 5m); 40 msgs sent to FlightAware
```

と、言う感じで`user`ログインも`feeder-id`も正しい事を確認します。

## 確認

後日ね！

