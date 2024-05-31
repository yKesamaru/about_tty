# TTYについてと日本語表示化

## はじめに
画面が固まってしまった時、よく使うのが仮想コンソールから特定のプロセスを殺すことです。
わたしの場合、そのためだけに仮想コンソールを使っていると言っても良いです。
ですので特に日本語表示の必要性は高くなかったのですが、日本語が文字化けしていることについてはずっと気になっていました。
そこで、今回はあらためて仮想コンソールまわりの知識の整理・まとめを行い、日本語表示できるようにするまでを残しておきたいと思います。

記事の終わりに参考文献リストを載せています。そちらをみていただければ日本語*入力*まで行うことができるでしょう。わたしには必要ないので、今回の記事では日本語表示できるようにするまでを扱います。

![](assets/eye-catch.png)

## 環境
```bash
$ inxi -Sxxx --filter
System:
  Kernel: 6.5.0-35-generic x86_64 bits: 64 compiler: N/A Desktop: Unity
    wm: gnome-shell dm: GDM3 42.0 Distro: Ubuntu 22.04.4 LTS (Jammy Jellyfish)
```
## UbuntuにおけるTTY端末
> UNIXの黎明期に大量に使われていた端末が`Teletype`です。
> コンピュータからの出力はプリンタに長い紙に打ち出されており、人間からの入力には紙テープやタイプライターが使われていました。UNIXでは端末のことを`tty`と呼ぶことがありますが、この`tty`という名称はテレタイプ(`TeleTYpe`)から来ています。
> テレタイプの後に登場したのがダム端末(`dumb terminal`)です。これには文字だけを映せるモニタとキーボードと、それに付随するハードウェアが付いていました。コンピュータのような自分で計算する機能が付いていないから`dumb(馬鹿)`です。ダム端末で最も有名なのは、今はなきDECが出していた`VT100`でしょう。
> ところで、`VT100`を始めとする初期のダム端末はキャラクタ端末(character terminal)とよばれ、、文字しか入出力できませんでした。入力（キーボード）の方は良いとしても、「文字しか出力できない」というのは中々想像しにくいでしょう。（中略）
> そして最も最近に登場したのが、端末エミュレータ(`terminal emulator`)です。もともとハードウェアであった端末を全てソフトウェアにしてしまったのが端末エミュレータで、GNOME terminal, Konsoel, mlterm, kterm, rxvtなどがその例としてあげられます。
> 仮想コンソール
> Linuxでは物理的な端末がそのまま使われるわけではなく、仮想コンソール(`virtual console`)が間に挟まっています。（中略）
> Linuxマシンを起動した時、真っ黒い画面に白い文字が大量に表示されるのを見たことがあるでしょう。これがLinuxの仮想コンソールの最初の１つです。`Ctrl+alt+Fn`を押すと、仮想コンソールを切り替えられます。
> ちなみにX Window Systemの起動中はXが端末を1つ専有します。つまり普通は仮想コンソールを1つ使うことになるでしょう。おそらくtty7が使われているはずです。
> 参考：青木 蜂郎. *ふつうのLinuxプログラミング*. ソフトバンククリエイティブ, 2005, pp. 63-65.

![Teletype Model 33, wikipedia](https://raw.githubusercontent.com/yKesamaru/about_tty/master/assets/2024-05-31-08-21-49.png)

Ubuntuも以前のバージョンでは`tty7`を使っていましたが、現在は`tty1, 2`です。`systemd`の導入からここらへんが変化したようです。

## `Virtual Console`と`Pseudo Terminal`
どちらも日本語にすると「仮想端末」になってしまうので混乱の元[^1]なのですが、`仮想コンソール(/dev/ttyX)`は`仮想端末(gnome-terminalなど)とは異なります。
[^1]: 話している人と聞いている人で、違うものを連想している状態がままある、ということ。
後者は正しくは「`疑似端末`」です。が、ふつうに話すときは「`仮想端末`」と（わたしも）表現しています。単に「`端末`」とか「`コンソール`」と読んだりしますね。
英語だと`Virtual Console`と`Pseudo Terminal`となっていてわかりやすいのですが。
例えば、以下のように違いがあります。
1. `Virtual Consol`
   ```bash
   $ tty
   /dev/tty3
   ```
2. `Pseudo Terminal`
   ```bash
   # tty: 現在のターミナルデバイスファイルパスを表示
   $ tty
   /dev/pts/0
   ```
### 補足：`Pseudo Terminal`の`Master`と`Slave`
あまり深堀はしませんが、`Pseudo Terminal`における`/dev/pts/0`について簡単に触れておきます。
`/dev/pts`ディレクトリの`pts`は`Pseudo Terminal Slave`（疑似端末スレーブ）の略です。
`/dev/pts`ディレクトリは疑似端末を管理するためのディレクトリです。
`/dev/pts/0`は最初の疑似端末(`Pseudo Terminal`)を表しています。次に作成された疑似端末は`/dev/pts/1`のように表現されます。
先程`Slave`（スレーブ）と表現しましたが、当然`Master`（マスター）もあります。[^2]
[^2]: SCSI接続のように、現在は名称が変わってるかもです。
1. `Master`: `/dev/ptmx`。ターミナルエミュレータがアクセスするデバイスファイル。
2. `Slave`: `/dev/pts/N`。ユーザが直接操作する仮想端末。`N`は端末番号。
ユーザーがターミナルで入力する内容は、スレーブデバイスを通じてマスターデバイスに渡され、そこからシェルなどに伝えられます。

### 補足：`chvt`コマンド
`Ctrl+Alt+Fn`キーの他にも仮想コンソールを切り替えることができます。
```bash
$ which chvt
/usr/bin/chvt
```
たとえば以下のようにして仮想コンソールを切り替えます。
```bash
$ sudo chvt 4
```
参考：[How To Switch Between TTYs Without Using Function Keys In Linux](https://ostechnix.com/how-to-switch-between-ttys-without-using-function-keys-in-linux/)


## `Virtual Consol`の文字化けとその対策
グラフィカルインターフェイス内の仮想端末、すなわち`gnome-terminal`のような`Pseudo Terminal`では日本語の表示および入力には問題はありません。
しかし何らかのトラブルが起きて`Virtual Consol`を立ち上げた時、日本語の部分が文字化けして読めないことがふつうにあります。
`Virtual Consol`を起動するのはトラブル時だけだし、しかも滞在時間が短いため、日本語表示が文字化けしていてもそのままにしている方も多いと思います。
一応昔から対応策はあって、よく「ロケールを変更しましょう」と言われます。
具体的には以下のとおりです。

```bash
export LANG=C
```

`C`ロケール、すなわちPOSIX標準ロケールに設定せよ、ということです。
ただこの場合、日本語ファイル名はエスケープされた文字となり、どっちにしろ読むことができません。
これの解決策は以下のとおりです。

> かつては`kon`, `kon2`が利用されてきましたが、最近ではVESAフレームバッファコンソールが有効になっています。しかしコンソールで日本語の読み書きをするには`bogl-bterm`, `jfbterm`などのターミナルソフトウェアと入力システムを別途導入する必要があります。
> 参考：中島 熊和. 熊田 伸一郎. *CentOS 徹底入門 第3版*. 翔泳社, 2012, p. 159.

`kon`, `kon2`はサポート終了しており、非推奨です。
`bogl-bterm`, `jfbterm`も古くなってしまいました。`fbterm`を使いましょう。

さてVESAフレームバッファ（通常有効になっていますが）が有効になっているか調べるには以下のようにします。
```bash
$ sudo dmesg | grep -i framebuffer
[    0.724663] efifb: framebuffer at 0xf1000000, using 3072k, total 3072k
```
このように表示されれば、フレームバッファが有効になっています。
または、以下の感じでも確認できます。
```bash
$ ls /dev/fb*
/dev/fb0
```
もしフレームバッファが有効化されていない場合はあきらめます。[^3]
[^3]: なにかの理由があるはずなので。
どうしてもの場合は以下のようにします。（未検証）
```bash
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash vga=0x318"
sudo update-grub
```

さて、VESAフレームバッファの確認ができたら、`fbterm`などのインストールを行います。
Ubuntu 22.04 では`fbterm`パッケージは登録されています（`universe`セクション）。
マウス操作ができると便利ですので`gpm`デーモンもインストールします。
```bash
sudo apt install -y fbterm gpm
```
`fbterm`などのインストールが終わったら、`SetUIDビット`を使用使用して、特権ユーザーの権限を借りることにします。
```bash
sudo chmod u+s /usr/bin/fbterm
```
これによりフレームバッファーターミナルエミュレータとして`fbterm`で日本語表示と入力を行うことができます。

### 補足：インストールしないもの
これ以外、日本語フォントの導入や、日本語入力システムの導入は、すでに行われている場合必要ありません。
わたしの環境では、`ibus-mozc`, `HackGenNerd-Regular.ttf`がインストールされていましたので、それらのインストール作業は行いません。

### 仮想コンソールでの`fbterm`の設定や起動
仮想コンソールに入ったら、フォントやロケールの設定、`fbterm`の起動を行います。
まずフォントの場所を把握しておく必要があります。わたしの環境では以下のとおりでした。
`~/.local/share/fonts/HackGenNerd_v2.5.3/HackGenNerd-Regular.ttf`

参考：[あらためて FbTerm (と周辺の諸々)](https://zenn.dev/kusaremkn/articles/428c4cd34e4ff9#fnref-6d1c-1)

`fbterm`が`video`グループに属しているため、ユーザーを`video`グループに属させます。
この操作を行わないと`fbterm`が立ち上がらず、再起動を余儀なくされます。
`sudo usermod -aG video user-name`

`fbterm`の設定ファイル`~/.fbtermrc`を設定します。
これらの値は試行錯誤が必要です。
```diff
- font-names=mono
- font-size=12
+ font-names=HackGenNerd-Regular.ttf
+ font-size=20

- input-method=
+ input-method=ibus-mozc
```
※ `input-method`を指定していますが、実際にはエラーが出て`ibus-mozc`を使用できませんでした。

1. 仮想コンソールに切り替え
`Ctrl + Alt + F3`
<!-- 2. 環境変数の設定
`export LANG=ja_JP.UTF-8` -->
2. `fbterm`の起動
`fbterm`

※ `fbterm --vesa-mode=list`でリストを得られるのですが、`--vesa-mode=N`(Nは数値)を指定するとなぜか必ずブラックアウトしてしまいます。

<!-- とはいっても、これらを仮想コンソールに入るたびに入力するには手間がかかるので、予めスクリプトとしてまとめておきます。
通常、シェルがログイン時に読み込むファイルには
- `.bash_profile`
- `.bash_login`
- `.profile`
があります。わたしの環境(`Ubuntu 22.04`)では`.profile`が存在しますので、これに記述をします。

```bash
# 日本語ロケールの設定
export LANG=ja_JP.UTF-8

# fbtermの自動起動設定
## もしログインできなくなっても2秒間のうちに`Ctrl + C`を押下して`fbterm`の起動をキャンセルできるようにする
if [ "$(tty)" != "/dev/tty1" ] && [ "$(tty)" != "/dev/tty2" ]; then
    sleep 2
    fbterm -v -f /usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc
fi
```
設定を反映させるために`source ~/.profile`を実行してください。 -->


`fbterm`のヘルプを以下に載せておきます。
```bash
使用法: fbterm [オプション] [--] [コマンド [引数]]
Linux用の高速フレームバッファ/VESAベースのターミナルエミュレータ

  -h, --help                      このヘルプメッセージを表示して終了
  -V, --version                   FbTermのバージョンを表示して終了
  -v, --verbose                   詳細情報を表示
  -n, --font-names=TEXT           フォントファミリ名を指定
  -s, --font-size=NUM             フォントのピクセルサイズを指定
      --font-width=NUM            フォント幅を指定
      --font-height=NUM           フォント高さを指定
  -f, --color-foreground=NUM      前景色を指定
  -b, --color-background=NUM      背景色を指定
  -e, --text-encodings=TEXT       追加のテキストエンコーディングを指定
  -r, --screen-rotate=NUM         画面表示の向きを指定
  -a, --ambiguous-wide            曖昧な幅の文字を広く扱う
  -i, --input-method=TEXT         入力方法プログラムを指定
      --cursor-shape=NUM          デフォルトのカーソル形状を指定
      --cursor-interval=NUM       カーソルの点滅間隔を指定
      --vesa-mode=NUM             VESAビデオモードを強制
                  list            利用可能なVESAビデオモードを表示

これらのオプションの詳細については、~/.fbtermrc のコメントを参照してください。

```

参考：
- [コンソールで日本語入力](https://blog.goo.ne.jp/lm324/e/57d71e72919ca63167eb109466fec5fb)
- [フレームバッファの使用と日本語表示: Vine Linux](https://vinelinux.org/docs/vine6/cui-guide/frame-buffer.html)

## 最後に
日本語表示はできるようになりましたが、きれいに表示させるには至りませんでした。（一応読める程度）
ですが、参考文献リストを読むといろいろな可能性に気付かされました。

以上です。ありがとうございました。


## 参考文献
- 青木 蜂郎. *ふつうのLinuxプログラミング*. ソフトバンククリエイティブ, 2005, pp. 63-65.
- 中島 熊和. 熊田 伸一郎. *CentOS 徹底入門 第3版*. 翔泳社, 2012, p. 44.
- [コンソールで日本語入力](https://blog.goo.ne.jp/lm324/e/57d71e72919ca63167eb109466fec5fb)
- [フレームバッファの使用と日本語表示: Vine Linux](https://vinelinux.org/docs/vine6/cui-guide/frame-buffer.html)
- [fbtermの日本語表示．uim-fepとuim-anthyで日本語入力も．ついでに，vimの256色化も](https://qiita.com/Pseudonym/items/12e447557a5234bb265b)