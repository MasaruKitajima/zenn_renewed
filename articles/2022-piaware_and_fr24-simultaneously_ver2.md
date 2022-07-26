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
Fri 22 Jul 16:46:49 JST 2022
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

## rtl_sdrのインストール

普通にFlightAwareとFlightRadar24にフィードするだけなら*rtl_sdr*は不要なんです。と、言うのは*rtl_sdr*は*SDR(Sofrware Defined Radio)*、つまり*rtl_sdr*をサーバーにしてネットワーク越しSDRレシーバーでラジオを聴く為のものだからです。

ですが*rtl_sdr*をサーバーにしてにMacのSDRレシーバーで、電波強度や受信ゲインの確認や調整ができるので後で便利だろう…と。

### 環境構築

```bash
masaru@pi:~ $ sudo apt-get update
# 導入（コンパイル・リンク）に必要なソフトウェアをインストールしていきます。
masaru@pi:~ $ sudo apt-get install git
masaru@pi:~ $ sudo apt-get install cmake
masaru@pi:~ $ sudo apt-get install libusb-1.0-0-dev
masaru@pi:~ $ sudo apt-get install build-essential
masaru@pi:~ $ sudo apt-get install pkg-config
masaru@pi:~ $ sudo apt-get install libtool
```

### rtl_sdrのインストール

```bash
masaru@pi:~ $ git clone git://git.osmocom.org/rtl-sdr.git
masaru@pi:~ $ cd rtl-sdr
masaru@pi:~/rtl-sdr $ mkdir build
masaru@pi:~/rtl-sdr $ cd build
masaru@pi:~/rtl-sdr/build $ cmake ../
# 中略
-- Build files have been written to: /home/masaru/rtl-sdr/build
masaru@pi:~/rtl-sdr/build $ make
Scanning dependencies of target rtlsdr
[  3%] Building C object src/CMakeFiles/rtlsdr.dir/librtlsdr.c.o
# 中略
[ 21%] Linking C shared library librtlsdr.so
[ 21%] Built target rtlsdr
# 中略
Scanning dependencies of target rtl_biast
[ 96%] Building C object src/CMakeFiles/rtl_biast.dir/rtl_biast.c.o
[100%] Linking C executable rtl_biast
[100%] Built target rtl_biast
masaru@pi:~/rtl-sdr/build $ sudo make install
masaru@pi:~/rtl-sdr/build $ sudo ldconfig
masaru@pi:~/rtl-sdr/build $ cd ..
masaru@pi:~/rtl-sdr $ sudo cp rtl-sdr.rules /etc/udev/rules.d/
```

#### とりあえずテスト

```bash
# その前に再起動っすね。
masaru@pi:~/rtl-sdr $ sudo reboot
masaru@pi:~ $ rtl_test -t
Found 1 device(s):
  0:  Realtek, RTL2838UHIDIR, SN: 00000001

Using device 0: Generic RTL2832U OEM

Kernel driver is active, or device is claimed by second instance of librtlsdr.
In the first case, please either detach or blacklist the kernel module
(dvb_usb_rtl28xxu), or enable automatic detaching at compile time.

usb_claim_interface error -6
Failed to open rtlsdr device #0.
# ブラックリストファイルを作成
masaru@pi:~ $ sudo vi /etc/modprobe.d/rtl-tcp-blacklist.conf
```

```conf
blacklist dvb_usb_rtl28xxu
blacklist rtl2830
blacklist dvb_usb_v2
blacklist dvb_core
```

もう一度再起動しなくては…そして再テスト

```bash
masaru@pi:~ $ lsusb
Bus 001 Device 008: ID 0bda:2838 Realtek Semiconductor Corp. RTL2838 DVB-T
Bus 001 Device 004: ID 046d:c069 Logitech, Inc. M-U0007 [Corded Mouse M500]
Bus 001 Device 005: ID 1c4f:0002 SiGma Micro Keyboard TRACER Gamma Ivory
Bus 001 Device 007: ID 0424:7800 Microchip Technology, Inc. (formerly SMSC)
Bus 001 Device 003: ID 0424:2514 Microchip Technology, Inc. (formerly SMSC) USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Microchip Technology, Inc. (formerly SMSC) USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Found 1 device(s):
  0:  Realtek, RTL2838UHIDIR, SN: 00000001

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
Supported gain values (29): 0.0 0.9 1.4 2.7 3.7 7.7 8.7 12.5 14.4 15.7 16.6 19.7 20.7 22.9 25.4 28.0 29.7 32.8 33.8 36.4 37.2 38.6 40.2 42.1 43.4 43.9 44.5 48.0 49.6
[R82XX] PLL not locked!
Sampling at 2048000 S/s.
No E4000 tuner found, aborting.
```

認識成功です。続いて*dump1090-fa*のインストールです。

## dump1090-faのインストール

目的の*ADS-B*信号は1090MHzで送信されていますが、受信したデータをデコードしてフィード可能な形式に変換するのが*dump1090*の役割です。

今も`fr24feed`をインストールすると`dump1090-mutability`がインストールされるんですが、[GitHubのレポジトリ](https://github.com/mutability/dump1090)には

> ## Old dump1090-mutability fork
>
> dump1090-mutability is no longer maintained; please don't use it for new installs.
> For new installs and ongoing development, try dump1090-fa, which is available at [https://github.com/flightaware/dump1090](https://github.com/flightaware/dump1090)
>
> The historical master branch is available in the unmaintained branch.

と書かれています。flightawareの*dump1090*を使えと言っておりますが、だったら*piaware*を取得して*dump1090-fa*だけインストールするのが良いのでは？

### piawareの取得とdump1090-fa

まずは最新版を確認する為に[flightawareのインストール](https://ja.flightaware.com/adsb/piaware/install)ページを訪れてみます。

> Download and install the PiAware repository package, which tells your Pi's package manager (apt) how to find FlightAware's software packages in addition to the packages provided by Raspbian.
>
> For Raspberry Pis running on Raspbian Bullseye OS, execute the following commands from the command line:
>
> ```bash
> wget https://ja.flightaware.com/adsb/piaware/files/packages/pool/piaware/p/piaware-support/piaware-repository_7.2_all.deb
sudo dpkg -i piaware-repository_7.2_all.deb
> ```

と言う事なので7.2が最新版ですのでこれを入手します。

```bash
masaru@pi:~ $ wget https://ja.flightaware.com/adsb/piaware/files/packages/pool/piaware/p/piaware-support/piaware-repository_7.2_all.deb
masaru@pi:~ $ sudo dpkg -i piaware-repository_7.2_all.deb
Selecting previously unselected package piaware-repository.
(Reading database ... 45761 files and directories currently installed.)
Preparing to unpack piaware-repository_7.2_all.deb ...
Unpacking piaware-repository (7.2) ...
Setting up piaware-repository (7.2) ...
masaru@pi:~ $ sudo apt-get update
# ここでflightwareのページではまるっとインストールする
# 手順ですので、注意してくださいね。
masaru@pi:~ $ sudo apt-get install dump1090-fa
# 中略
Created symlink /etc/systemd/system/default.target.wants/dump1090-fa.service → /lib/systemd/system/dump1090-fa.service.
# これでdump1090-faがsystemserviceに登録された事が判ります。
# つまり自動起動の設定をする必要が無いんですね。
# 終了したとか成功したとか言ってくれないので、プロンプトを待ちます。
```

プロンプトが表示されたら*dump1090-fa*の状態を確認します。

```bash
masaru@pi:~ $ systemctl status dump1090-fa
● dump1090-fa.service - dump1090 ADS-B receiver (FlightAware customization)
     Loaded: loaded (/lib/systemd/system/dump1090-fa.service; enabled; vendor p>
     Active: active (running) since Sun 2022-07-24 09:53:38 JST; 3min 55s ago
       Docs: https://flightaware.com/adsb/piaware/
   Main PID: 1943 (dump1090-fa)
      Tasks: 3 (limit: 1598)
        CPU: 1min 20.934s
     CGroup: /system.slice/dump1090-fa.service
             └─1943 /usr/bin/dump1090-fa --quiet --device-type rtlsdr --gain 60>

Jul 24 09:53:38 pi systemd[1]: Started dump1090 ADS-B receiver (FlightAware cus>
Jul 24 09:53:38 pi dump1090-fa[1943]: Sun Jul 24 09:53:38 2022 JST  dump1090-fa>
Jul 24 09:53:39 pi dump1090-fa[1943]: rtlsdr: using device #0: Generic RTL2832U>
Jul 24 09:53:39 pi dump1090-fa[1943]: Found Rafael Micro R820T tuner
Jul 24 09:53:39 pi dump1090-fa[1943]: rtlsdr: tuner gain set to about 58.6 dB (>
Jul 24 09:53:39 pi dump1090-fa[1943]: adaptive: using 50% duty cycle
Jul 24 09:53:39 pi dump1090-fa[1943]: adaptive: enabled adaptive gain control w>
Jul 24 09:53:39 pi dump1090-fa[1943]: adaptive: enabled dynamic range control, >
Jul 24 09:53:49 pi dump1090-fa[1943]: adaptive: reached upper gain limit, halti>
```

さて*dump1090-fa*の設定ファイル`/usr/lib/systemd/system/dump1090-fa.service`を覗いて見ると

```conf
ExecStart=/usr/share/dump1090-fa/start-dump1090-fa --write-json %t/dump1090-fa
```

なる行があります。これは*dump1090-fa*がログを`/run/dump1090-fa/`に出力している事を意味します。

パソコンのHDDやSSDならともかくSDカードに頻繁に書き書きされるのは物理的寿命を縮めるので、RAM領域を作成して、そちらにログを出力させてみます。