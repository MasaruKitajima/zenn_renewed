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

```she;;
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