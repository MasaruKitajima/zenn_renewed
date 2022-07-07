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
```

### うぎゃぁ…

```bash
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 vlc-bin : Depends: libvlc-bin (= 3.0.16-1+rpi1+rpt2) but 3.0.17.4-0+deb11u1+rpi1 is to be installed
 vlc-plugin-base : Depends: vlc-data (= 3.0.16-1+rpi1+rpt2) but 3.0.17.4-0+deb11u1+rpi1 is to be installed
 vlc-plugin-skins2 : Depends: vlc-plugin-qt (= 3.0.17.4-0+deb11u1+rpi1) but 3.0.16-1+rpi1+rpt2 is to be installed
E: Broken packages
```

なってこった！エラーが出てるがね。`VLC`なんて使わないんだけど、解決しないと先に進めないよね…

```bash
masaru@feeder:~ $ sudo apt-get dist-upgrade
104 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
Need to get 244 MB of archives.
After this operation, 11.5 MB of additional disk space will be used.
```

いや、どーする？って言われてもやらんきゃどーしよーもないんでしょ？***Y***です！

その後に、念のために「リリース情報が変わったヤツはアップデートしてもいいで」と言う意味で

```bash
masaru@feeder:~ $ sudo apt-get update --allow-releaseinfo-change
```

を実行します。そして再起動します。

```bash
masaru@feeder:~ $ sudo reboot
```

しばらく待ってネットワーク越しに見られるようになったら次に進めます。


**要注意**

色々なサイトに初期設定についての記述がありますが`sudo rpi-update`なる記述は**非推奨**ですので、しないでくださいね。

今は*deprecated*になっていますが[Git Hub](https://github.com/)の[リポジトリ](https://github.com/Hexxeh/rpi-update)に**なぜ推奨していないか**と言う理由が説明されています。

ただ…オッチャンは**チャレンジャー**や**人柱希望者**が大好きなので***At Your Own Risk***で試すのは止めないです。

今オッチャンも*iOS 16.2 beta 2, iPad OS 16.2 beta 2, watchOS 9 beta 2*で難儀してますしね💦でも、さすがにメインマシンで*macOS 13 beta 2*を試す程愚かではありません。

では元に戻って…

## FlightAwareにフィードする

今も`fr24feed`をインストールすると`dump1090-mutability`がインストールされるんですが、[GitHubのレポジトリ](https://github.com/mutability/dump1090)には

> ## Old dump1090-mutability fork
>
> dump1090-mutability is no longer maintained; please don't use it for new installs.
> For new installs and ongoing development, try dump1090-fa, which is available at [https://github.com/flightaware/dump1090](https://github.com/flightaware/dump1090)
>
> The historical master branch is available in the unmaintained branch.

と言う事でオワコンなのですよ。なので素直にリンク先の`dump-1090-fa`をビルドします。

 ### 前準備

`dump1090-fa`を自力でビルドするために必要になるだろうパッケージを先にインストールします。（単に`README`に書いてあったと言う理由で）

```shell
masaru@feeder:~ $  sudo apt-get install build-essential fakeroot debhelper librtlsdr-dev pkg-config libncurses5-dev libbladerf-dev libhackrf-dev liblimesuite-dev
```

無事に終わったー！んじゃ、ビルドしてきましょー。

```shell
masaru@feeder:~ $ git clone https://github.com/flightaware/dump1090 dump1090-fa
masaru@feeder:~ $ cd dump1090-fa
# 今回はMakeではなくでパッケージにします。
# このオプションだとrtl-sdr専用になります。ｌ
masaru@feeder:~/dump1090-fa $  dpkg-buildpackage -b --no-sign --build-profiles=custom,rtlsdr
```

ビルドが植わると上の階層にパッケージが生成されています。

- dump1090-fa_7.2_armhf.buildinfo
- dump1090-fa_7.2_armhf.changes
- dump1090-fa_7.2_armhf.deb
- dump1090-fa-dbgsym_7.2_armhf.deb

で、使うのは`dump1090-fa_7.2_armhf.deb`ですので

```
masaru@feeder:~ $ sudo dpkg -i dump1090-fa_7.2_armhf.deb
```

でインストールしてから自動起動の設定をしてからサービスを開始します。

```shell
masaru@feeder:~ $ sudo systemctl enable dump1090-fa
masaru@feeder:~ $ sudo systemctl start dump1090-fa
masaru@feeder:~ $ systemctl status dump1090-fa
● dump1090-fa.service - dump1090 ADS-B receiver (FlightAware customization)
     Loaded: loaded (/lib/systemd/system/dump1090-fa.service; enabled; vendor p>
     Active: active (running) since Tue 2022-07-05 15:27:14 JST; 49s ago
       Docs: https://flightaware.com/adsb/piaware/
   Main PID: 8681 (dump1090-fa)
      Tasks: 3 (limit: 1598)
        CPU: 17.633s
     CGroup: /system.slice/dump1090-fa.service
             └─8681 /usr/bin/dump1090-fa --quiet --device-type rtlsdr --gain 60>

Jul 05 15:27:14 feeder dump1090-fa[8681]: Tue Jul  5 15:27:14 2022 JST  dump109>
Jul 05 15:27:14 feeder dump1090-fa[8681]: rtlsdr: using device #0: Generic RTL2>
Jul 05 15:27:14 feeder dump1090-fa[8681]: Detached kernel driver
Jul 05 15:27:14 feeder dump1090-fa[8681]: Found Rafael Micro R820T tuner
Jul 05 15:27:15 feeder dump1090-fa[8681]: rtlsdr: tuner gain set to about 58.6 >
Jul 05 15:27:15 feeder dump1090-fa[8681]: adaptive: using 50% duty cycle
Jul 05 15:27:15 feeder dump1090-fa[8681]: adaptive: enabled adaptive gain contr>
Jul 05 15:27:15 feeder dump1090-fa[8681]: adaptive: enabled dynamic range contr>
Jul 05 15:27:15 feeder dump1090-fa[8681]: Allocating 4 zero-copy buffers
Jul 05 15:27:25 feeder dump1090-fa[8681]: adaptive: reached upper gain limit, h>
```

よし、動いてる。

ここで**長寿を願って**設定を変更します。`dumo1090-fa`のログの出力は`/var/run/dump1090-fa`に出力されます。

これではkoredehaSDカード自体への書き込みが頻繁に発生しますので、SDカードの寿命を縮めてしまいます。

と、言う事でRAMディスクである`/tmp`以下にログを出力してもらいます。

```shell
masaru@pi:~ $ sudo vi /lib/systemd/system/dump1090-fa.service
```

```conf
[Service]
User=dump1090
RuntimeDirectory=dump1090-fa
RuntimeDirectoryMode=0755
# 以下2行を追加
ExecStartPre=/bin/bash -c '/bin/mkdir -p /tmp/dump1090-fa 2> /dev/null'
ExecStartPre=/bin/chmod 777 /tmp/dump1090-fa
# ここまで
# 書き換える
# ExecStart=/usr/share/dump1090-fa/start-dump1090-fa --write-json %t/dump1090-fa
ExecStart=/usr/share/dump1090-fa/start-dump1090-fa --write-json /tmp/dump1090-fa
SyslogIdentifier=dump1090-fa
Type=simple
Restart=on-failure
RestartSec=30
RestartPreventExitStatus=64
Nice=-5
```

これでJSONファイルの出力先が変わるのですが、それを読み込んでいる`lighttpd`の設定も変える必要があります。

```shell
masaru@pi:~ $ sudo vi /etc/lighttpd/conf-available/89-skyaware.conf
:%s/\/run\/dump1090-fa\//\/tmp\/dump1090-fa\/
:wq
```


### PiAwareのインストール

[公式サイト](https://ja.flightaware.com/adsb/piaware/install)に書いてある通りに進めてみます。

```shell
masaru@feeder:~ $ wget https://ja.flightaware.com/adsb/piaware/files/packages/pool/piaware/p/piaware-support/piaware-repository_7.2_all.deb
Resolving ja.flightaware.com (ja.flightaware.com)... 206.253.81.28
Connecting to ja.flightaware.com (ja.flightaware.com)|206.253.81.28|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6564 (6.4K) [application/x-debian-package]
Saving to: ‘piaware-repository_7.2_all.deb’

piaware-repository_ 100%[===================>]   6.41K  --.-KB/s    in 0s

2022-07-05 17:21:38 (38.3 MB/s) - ‘piaware-repository_7.2_all.deb’ saved [6564/6564]
masaru@feeder:~ $ sudo dpkg -i piaware-repository_7.2_all.deb
Selecting previously unselected package piaware-repository.
(Reading database ... 104958 files and directories currently installed.)
Preparing to unpack piaware-repository_7.2_all.deb ...
Unpacking piaware-repository (7.2) ...
Setting up piaware-repository (7.2) ...
masaru@feeder:~ $ sudo apt-get update
masaru@feeder:~ $ sudo apt-get install piaware
The following additional packages will be installed:
  itcl3 tcl tcl-tls tcl8.6 tcllib tclx8.4
Suggested packages:
  itcl3-doc dump978-fa tcl-tclreadline tcllib-critcl tclx8.4-doc
The following NEW packages will be installed:
  itcl3 piaware tcl tcl-tls tcl8.6 tcllib tclx8.4
0 upgraded, 7 newly installed, 0 to remove and 2 not upgraded.
Need to get 8,460 kB of archives.
After this operation, 40.4 MB of additional disk space will be used.
Do you want to continue? [Y/n] Y #Yって答えんでどーするねんｗ
masaru@feeder:~ $ sudo piaware-config allow-auto-updates yes
Set allow-auto-updates to yes in /etc/piaware.conf:7
masaru@feeder:~ $ sudo piaware-config allow-manual-updates yes
Set allow-manual-updates to yes in /etc/piaware.conf:8
# dump978=978MHzは主に小型機がエアロマリタイム用にアメリカで使っていました
# 近年はセスナやヘリなどの事故の多発を受けて日本でも試験導入をしています。
# 入れなくても通常のADS-Bのフィードには影響はありませんので飛ばしても構わないです。
masaru@feeder:~ $ sudo apt-get install dump978-fa
After this operation, 48.4 MB of additional disk space will be used.
Do you want to continue? [Y/n] Y
masaru@feeder:~ $ sudo reboot
```

再起動後、PiAwareを起動します。

```shell
# 自動起動の設定
masaru@feeder:~ $ sudo systemctl enable piaware
# 起動！
masaru@feeder:~ $ sudo systemctl start piaware
```

PiAwareが正常に起動するまで4〜5分位かかります。だから、その後に確認をしましょう。

##  dump1090-faの監視

### ローカルホストの準備

```shell
[masarukitajima@localhost /Users/masarukitajima:] $ brew install grafana
[masarukitajima@localhost /Users/masarukitajima:] $ brew services restart grafana
```

[ローカルのGrafana](http://localhost:3000/)にログインします。初期ID・PWはどちらも*admin*です。

**ログインしたらすぐにパスワードの変更を求められます**

とりあえずローカルホストは一旦終了。

次にGrafanaから*dumo1090*用のダッシュボードを取得します。[dump1090exporter](https://grafana.com/grafana/dashboards/768)に行って*Download JSON*をクリックして保存しておいてください。

### ラズパイ側1 - dump1090-exporte

ラズパイ側では[dump1090-exporter](https://github.com/claws/dump1090-exporter)を使います。

```shell
masaru@pi:~ $ sudo apt install python3-pip python3-setuptools python3-dev
masaru@pi:~ $ sudo pip3 install dump1090exporter
```

上記のGitHubのページでサンプルとして掲示されている実行スクリプトは

```shell
dump1090exporter \
  --resource-path=http://192.168.1.201:8080/data \
  --port=9105 \
  --latitude=-34.9285 \
  --longitude=138.6007 \
  --log-level info
 ```

 です。この内変更すべきなのは*resource-path, latitude ,longitude*だけです。

 `resource-path`は`dump1090-fa`が出力するJSONファイルのpathです。今の環境では`--resource-path=/tmp/dump1090-fa`となります。

 - --latitude
 - --longitude

はご自分の場所にします。

さて、これをシステム起動時に自動起動させたい訳ですけれど、わざわざsystemd用ファイルを作るほど大仰なものでもないので*(面倒だから…と言うのは内緒です)*、`/etc/rc.local`の最後の`exit 0`の前に

```config
/bin/sleep 20
/usr/local/bin/dump1090exporter --latitude=-43.06464 --longitude=141.31741 --resource-path "/tmp/dump1090-fa" &
 ```

 の2行を追加します。

 `/bin/slep 20`は、システム起動してから20秒経てば落ち着くかな？と思って入れています。この時間は皆さんの環境で変わりますので、調整してみてください。

 ```shell
 masaru@pi:~ $ sudo vi /etc/rc.local
 ```

```shell
masaru@pi:~ $ systemctl status dump1090-fa
● dump1090-fa.service - dump1090 ADS-B receiver (FlightAware customization)
     Loaded: loaded (/lib/systemd/system/dump1090-fa.service; enabled; vendor p>
     Active: active (running) since Thu 2022-07-07 14:04:40 JST; 1h 43min ago
```

で、dump1090-faが無事に起動しているのが確認できたら、一度手動で実行してみます。

```shell
masaru@pi:~ $ /usr/local/bin/dump1090exporter --latitude=-43.06464 --longitude=141.31741 --resource-path "/tmp/dump1090-fa" &
[1] 3545
masaru@pi:~ $ 2022-07-07 15:50:21.176 [INFO] [dump1090exporter.exporter] Monitoring dump1090 resources at: /tmp/dump1090-fa
2022-07-07 15:50:21.176 [INFO] [dump1090exporter.exporter] Refresh rates: aircraft=0:00:10, statstics=0:01:00
2022-07-07 15:50:21.177 [INFO] [dump1090exporter.exporter] Origin: Position(latitude=-43.06464, longitude=141.31741)
2022-07-07 15:50:21.182 [INFO] [dump1090exporter.exporter] serving dump1090 prometheus metrics on: http://0.0.0.0:9105/metrics
2022-07-07 15:50:21.184 [INFO] [dump1090exporter.exporter] Origin successfully extracted from receiver data: Position(latitude=43.06, longitude=141.32)
```

私の環境でdump1090-faが動いているホスト=ラズパイのホストですからIPアドレスは*192.168.1.100*です。ブラウザで[http://192.168.1.100:9105/metrics](http://192.168.1.100:9105/metrics)へアクセスして情報出力を確認します。

```
# HELP dump1090_messages_total Number of Mode-S messages processed since start up
# TYPE dump1090_messages_total gauge
dump1090_messages_total{time_period="latest"} 15690
# HELP dump1090_recent_aircraft_max_range Maximum range of recently observed aircraft
# TYPE dump1090_recent_aircraft_max_range gauge
dump1090_recent_aircraft_max_range{time_period="latest"} 0.0
# HELP dump1090_recent_aircraft_max_range_by_direction Maximum range of recently observed aircraft by direction relative to receiver
# TYPE dump1090_recent_aircraft_max_range_by_direction gauge
# 後略
```

何か値が入っていれば110数行出力されれば問題無しです。

### ラズパイ側2 - Prometheus

[dump1090-exporter](https://github.com/claws/dump1090-exporter)のREADMEには

> `Prometheus`の構成がどうのこうの

と書いてあるので、とりあえず[Prometheus Download Center](https://prometheus.io/download/)からお使いのCPU用のファイルを`wget`で取得します。

```shell
masaru@pi:~ $ uname -a
Linux pi 5.15.32-v7+ #1538 SMP Thu Mar 31 19:38:48 BST 2022 armv7l GNU/Linux
```


と言う事なので*armv7*用を取得します。