# 別解

## rtl-sdrのインストール

### 下準備

まずはインストールするに当たり必要な各種パッケージをインストールします。

```bash
masaru@pi:~ $ sudo apt-get install cmake
masaru@pi:~ $ apt-cache search libusb
masaru@pi:~ $ sudo apt-get install libusb-1.0-0-dev
masaru@pi:~ sudo apt-get install build-essential
masaru@pi:~ $ sudo apt-get install libtool
```

と、これでいいかな？

### rtl-sdr

```bash
masaru@pi:~ $ git clone git://git.osmocom.org/rtl-sdr.git
masaru@pi:~ $ cd rtl-sdr
masaru@pi:~/rtl-sdr $ mkdir build
masaru@pi:~/rtl-sdr $ cd build
masaru@pi:~/rtl-sdr/build $ cmake ../
-- Build files have been written to: /home/masaru/rtl-sdr/build
masaru@pi:~/rtl-sdr/build $ make
masaru@pi:~/rtl-sdr/build $ sudo make install
masaru@pi:~/rtl-sdr/build $ sudo cp ../rtl-sdr.rules /etc/udev/rules.d/
```

これで`rtl-sdr`で採用されている*RTL28xx*チップに対応するテレビ受信用標準ドライバをblacklist登録してカーネルに読み込まれないようにします。

その為にファイルを一つ作成します。

```bash
masaru@pi:~/rtl-sdr $ sudo vi /etc/modprobe.d/rtl_blacklist.conf
```

内容は

```conf
blacklist dvb_usb_rtl28xxu
blacklist rtl2832
```

とします。

ここでラズパイを再起動します。

#### 動作確認

まず、ADS-B用のRTL-SDRドングルがUSBに接続され、認識されているかを確認します。

```bash
masaru@pi:~ $ lsusb
Bus 001 Device 004: ID 0bda:2838 Realtek Semiconductor Corp. RTL2838 DVB-T
Bus 001 Device 005: ID 0424:7800 Microchip Technology, Inc. (formerly SMSC)
Bus 001 Device 003: ID 0424:2514 Microchip Technology, Inc. (formerly SMSC) USB 2.0 Hub
Bus 001 Device 002: ID 0424:2514 Microchip Technology, Inc. (formerly SMSC) USB 2.0 Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

んじゃテストぞな。

```bash
masaru@pi:~ $ rtl_test -t
Found 1 device(s):
  0:  Realtek, RTL2838UHIDIR, SN: 00000001

Using device 0: Generic RTL2832U OEM
Found Rafael Micro R820T tuner
Supported gain values (29): 0.0 0.9 1.4 2.7 3.7 7.7 8.7 12.5 14.4 15.7 16.6 19.7 20.7 22.9 25.4 28.0 29.7 32.8 33.8 36.4 37.2 38.6 40.2 42.1 43.4 43.9 44.5 48.0 49.6
[R82XX] PLL not locked!
Sampling at 2048000 S/s.
No E4000 tuner found, aborting.
```

**予告（違う）**

TODO: M1 macにSDR(Software Defined Radio)でrtl_tcp対応のものをインストールすべし！

## dump1090-faのインストール

インストールしたいのは`dump1090-fa`ですが、これは`piaware`の中に含まれています。ですのでまず`piaware`をもらってきて、その中から`dump1090-fa`だけをインストールします。

```bash
masaru@pi:~ $ wget https://ja.flightaware.com/adsb/piaware/files/packages/pool/piaware/p/piaware-support/piaware-repository_7.2_all.deb
masaru@pi:~ $ sudo dpkg -i piaware-repository_7.2_all.deb
masaru@pi:~ $ sudo apt-get update
# 間違えて`sudo apt-get install piaware`でpiawareを丸ごとインストールしないように注意ね。
masaru@pi:~ $ sudo apt-get install dump1090-fa
以下のパッケージが自動でインストールされましたが、もう必要とされていません:
  libfuse2
これを削除するには 'sudo apt autoremove' を利用してください。
# いらんもんは削除して空き容量を増やすべし
masaru@pi:~ $ sudo apt autoremove
```

### dump1090-faの設定

```bash
```
