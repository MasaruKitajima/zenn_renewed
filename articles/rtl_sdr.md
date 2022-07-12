# 別解

## rtl-sdrのインストール

### 下準備

まずはインストールするに当たり必要な各種パッケージをインストールします。

```bash
masaru@pi:~ $ sudo apt-get install cmake
masaru@pi:~ $ apt-cache search libusb
masaru@pi:~ $ sudo apt-get install libusb-1.0-0-dev
masaru@pi:~ $ sudo apt-get install autoconf
masaru@pi:~ $ sudo apt-get install libtool
```

と、これでいいかな？

### rtl-sdr

```bash
masaru@pi:~ $ git clone git://git.osmocom.org/rtl-sdr.git
masaru@pi:~ $ cd rtl-sdr
masaru@pi:~/rtl-sdr $ aclocal
masaru@pi:~/rtl-sdr $ autoreconf --force --install
masaru@pi:~/rtl-sdr $ ./configure
masaru@pi:~/rtl-sdr $ make
masaru@pi:~/rtl-sdr $ sudo make install
masaru@pi:~/rtl-sdr $ sudo make install-udev-rules
sudo ldconfig
```

これで`rtl-sdr`で採用されている*RTL28xx*チップに対応するテレビ受信用標準ドライバをblacklist登録してカーネルに読み込まれないようにします。

