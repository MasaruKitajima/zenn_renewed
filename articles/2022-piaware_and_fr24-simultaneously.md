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
pi@pi:~ $ vi .bashrc
alias mkdir='mkdir -p'
alias la='ls -la'
alias rf='rm -rf'
pi@pi:~ $ source .bashrc

# ラズパイは初期化した時にrootにパスワードが設定されていない
# suを使う為にパスワードを設定する
pi@pi:~ $ sudo passwd root
New password:
Retype new password:
passwd: password updated successfully

# パッケージ更新・再起動
# これはラズパイを初期化する時のお作法
pi@pi:~ $ sudo apt-get update
pi@pi:~ $ sudo apt-get upgrade
```

**要注意**

色々なサイトに初期設定についての記述がありますが`sudo rpi-update`なる記述は**非推奨**ですので、しないでくださいね。

今は*deprecated*になっていますが[Git Hub](https://github.com/)の[リポジトリ](https://github.com/Hexxeh/rpi-update)に**なぜ推奨していないか**と言う理由が説明されています。

ただ…オッチャンは**チャレンジャー**や**人柱希望者**が大好きなので***At Your Own Risk***で試すのは止めないです。

今オッチャンも*iOS 16.2 beta 2, iPad OS 16.2 beta 2, watchOS 9 beta 2*で難儀してますしね💦でも、さすがにメインマシンで*macOS 13 beta 2*を試す程愚かではありません。

では元に戻って…

```bash