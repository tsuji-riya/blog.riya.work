---
title: D-Link DSM-232 で Linux を動かす(4) USB を挿して電源を入れるだけで起動させる
date: 2026-11-20
tags:
  - 組み込み
  - U-Boot
  - Linux
  - リバースエンジニアリング
---

[[dsm232-linux-mainline-drivers|前回]]までは、USB メモリの Linux を起動するのに毎回 UART をつないでいた。U-Boot の自動起動を止めて、`usb start; fatload ...; source ...` と打つ。`bootcmd` を書き換えて `saveenv` してしまえば UART は要らなくなるけど、それは本体の設定を書き換えたままにするということで、あまりやりたくない。

今回の目標はこれ。

- USB メモリを挿したまま電源を入れたときだけ、USB の Linux が起動する
- 挿していなければ純正のファームウェアがそのまま起動する
- 本体の環境変数を書き換えたままにしない
- USB ハブの先に挿した USB メモリでも動く

結論から書くと、純正ファームウェアに入っている工場向けの USB アップデートの仕組みを入口にして、実機で動いた。U-Boot 本体には何も書いていない。

今回も解析と実装は Claude Code に任せた。実機での確認は、自分で UART のログを取って貼り返す形で進めた。

## 純正に USB 起動の機能はあるか

まずは純正の中に「USB に新しい Linux があったら起動する」ような機能が無いかを探した。

純正 U-Boot 2011.06 の起動コマンドはこれ。

```
bootcmd=dcache on;run ledon; run show_logo;boots
```

`boots` は NAND に置かれた Field と Factory の 2 つのイメージ(RSA で署名されていて、暗号化もされている)を読んで起動するコマンドで、USB は見ない。`USB_UPDATES2` という、名前からして USB からの更新に使いそうなフックもあったが、中身は空だった。U-Boot だけでは USB から起動させる手は無い。

純正 Linux 側も追った。起動スクリプトは `/etc/rcS` から `/etc/normalrc` に進み、その中で毎回 `usbupg -f` を実行している。`usbupg` は USB メモリの第 1 パーティション(`/dev/sda1`)を `/mnt/usb` にマウントし、`cameo_factory/factory.img` があれば `cd /mnt/usb/cameo_factory` してから `fwimg -f factory.img` に渡す。工場の生産ラインで使う仕組みだと思う。

ここで新しいカーネルを起動するような処理は無いので、純正には「USB を挿すだけで別の Linux を起動する」機能は無い。ただ、起動のたびに USB メモリの上のファイルを読んで何かするところまではやってくれる。そこで `fwimg` を見ることにした。

## fwimg の中身

本体から吸い出したファームウェアの `fwimg` はそのままでは読めなかったので、D-Link の GPL パッケージ(1.02.00)に入っていた `fwimg` を逆アセンブルした。

イメージの形式は U-Boot の legacy イメージと同じ 64 バイトのヘッダだが、細かいところが違う。

| 項目     | 内容                                                |
| -------- | --------------------------------------------------- |
| バイト順 | 全部リトルエンディアン(U-Boot はビッグエンディアン) |
| マジック | `0x70883967`(ファイル上は `67 39 88 70`)            |
| CRC32    | zlib と同じ表だが、初期値 0 で最後の反転もしない    |
| 名前     | `DSM232` を含んでいないと弾かれる                   |
| タイプ   | 0x1e バイト目。12 が app                            |

タイプごとの処理を見ると、カーネル、rootfs、nvram、uboot、miniloader といったタイプは対応する `/dev/mtdN` にそのまま書き込む。これは本体を壊しかねないので使わない。

面白いのは app タイプ(12)で、データを `/mnt/jffs2/app.tar.gz` に書き出し、展開して、こう実行する。

```
chmod 755 /mnt/jffs2/app.sh;/mnt/jffs2/app.sh
```

`system()` で呼んでいるので、`app.sh` は root で動く。署名の検証は無い。RSA の署名を見ているのは U-Boot の `boots` だけで、`fwimg` はヘッダと CRC と名前しか見ていなかった。つまり USB メモリに `cameo_factory/factory.img` を置いておけば、純正ファームウェアが起動するたびに好きなシェルスクリプトを root で走らせてもらえる。

イメージを作るスクリプトはこれだけ。

```python
MAGIC, OS_LINUX, ARCH_ARM, TYPE_APP, COMP_NONE = 0x70883967, 5, 2, 12, 0
HDR = struct.Struct("<7I4B32s")

def crc(b):
	"""CRC-32 table of zlib, but starting at 0 and without the final inversion (vendor crc32)."""
	return zlib.crc32(b, 0xFFFFFFFF) ^ 0xFFFFFFFF

def make(data, name=b"DSM232_App", stamp=None):
	stamp = int(time.time()) if stamp is None else stamp
	fields = [MAGIC, 0, stamp, len(data), 0, 0, crc(data), OS_LINUX, ARCH_ARM, TYPE_APP, COMP_NONE, name]
	fields[1] = crc(HDR.pack(*fields))
	return HDR.pack(*fields) + data
```

最初は普通の `zlib.crc32` で書いていて、ヘッダの CRC が合わなかった。GPL パッケージには純正の `mkimg` も入っていたので、i386 のコンテナで `mkimg -T app` を動かして同じ入力からイメージを作らせ、出力を比べて CRC の違いに気づいた。今はバイト単位で一致している。

## どうやって Linux に渡すか

`app.sh` を root で動かせることは分かった。次は、そこからどうやって USB の Linux を起動するか。

最初に考えたのは、純正 Linux から kexec で USB のカーネルに飛ぶ方法。でも純正カーネルは `KEXEC` が無効だった。加えて、`normalrc` はハードウェアのウォッチドッグを `-t 1 -T 2` で動かしているので、餌やりが止まると 2 秒でリセットされる。カーネルモジュールで無理やりやる手も無くはないけど、割に合わない。

`app.sh` から `bootcmd` を書き換えて保存するのは簡単だが、それでは本体の設定を書き換えたままになってしまう。

そこで、U-Boot の環境変数に「1 回だけ使う起動コマンド」を書いて再起動し、U-Boot 自身に元へ戻させることにした。`app.sh` が書くのは次の 2 つ。

```
bootcmd=setenv bootcmd 'dcache on;run ledon; run show_logo;boots';saveenv;run dsmusb;run bootcmd
dsmusb=setenv dsmusb;saveenv;usb start;for d in 0 1 2 3;do if fatload usb $d:1 6000000 boot.scr;then setenv usbdev $d;source 6000000;fi;done;usb stop
```

再起動すると U-Boot はこの `bootcmd` を実行して、次の順に進む。

1. `bootcmd` を元の値に戻して `saveenv` する。この時点で、次に電源を入れたときは何があっても純正で起動する。
2. `run dsmusb` で、`dsmusb` が自分自身を消してもう一度 `saveenv` する。これで保存されている環境変数は元どおりになる。
3. USB ストレージの 0〜3 番を順に見て、`boot.scr` が読めたらそれを実行する。`boot.scr` は最後に `bootm` するので戻ってこない。
4. どこにも無ければ `usb stop` して、最後の `run bootcmd` が元に戻した値を実行し、純正で起動する。

USB を探すループを `bootcmd` に直接書かず、別の変数に分けたのには理由がある。純正 U-Boot の hush に `for` があることは文字列から分かっていたけど、本当にこの書き方で動くかは試すまで分からなかった。ループを解釈できなかったとしても、`bootcmd` 自体はすでに元に戻って保存されているので、`dsmusb` が残るだけで純正は起動する。

`app.sh` から書くときの順番も気にした。`dsmusb` を先に書き、`bootcmd` を後に書く。この 2 回の書き込みの間で電源が切れても、`bootcmd` は元のままなので普通に起動する。

環境変数を書く純正の `/dlink/env` にも癖があった。`env set 名前 値…` は、引数をつなげて 256 バイトのスタック上のバッファにコピーするが、長さを確かめていない。長い値を渡すとスタックが壊れる。なので値はそれぞれ 240 バイト未満に抑えた。実際の長さは `bootcmd` が 88 バイト、`dsmusb` が 142 バイト。

## 再起動ループを止める

もう一つ考えておくことがあった。USB の Linux の起動に失敗して純正に戻ってくると、USB メモリは挿さったままなので、また `app.sh` が走って再起動する。これだと無限ループになる。

そこで `app.sh` は、環境変数を書く前に USB メモリの上に `cameo_factory/armed` という印のファイルを作る。USB の Linux は起動したらこれを消す。純正に戻ってきたときに `armed` が残っていれば、前回の USB 起動は Linux まで届かなかったということなので、印だけ消して何もしない。次に電源を入れたときにもう一度試す。

## USB ハブ越しにも対応する

USB ハブの先に USB メモリを挿したり、他の USB ディスクと一緒に挿したりすると、USB メモリが 0 番になるとは限らない。U-Boot 側は、上の `dsmusb` で 0〜3 番を順に探し、見つかった番号を `usbdev` に入れる。`boot.scr` はその番号から読む。

```
if test x${usbdev} = x; then setenv usbdev 0; fi
fatload usb ${usbdev}:1 0x4000000 uImage
fatload usb ${usbdev}:1 0x5000000 uInitrd
```

`x${usbdev}` という書き方は[[dsm232-linux-hdmi-usb-wifi|2 回目]]に書いた hush の癖のせい。UART から手で `boot.scr` を読んだときは `usbdev` が空なので 0 を使う。

Linux 側も、ルートファイルシステムを `/dev/sda2` と決め打ちするのをやめて、ラベルで探すようにした。`boot.scr` は `root=LABEL=dsm232root` を渡し、initramfs の `/init` が busybox の `findfs` でデバイスを探す。

```sh
if [ "${root#LABEL=}" != "${root:-}" ]; then
	for i in $(seq 20); do dev=$(findfs "$root" 2>/dev/null) && break; echo "waiting for $root"; sleep 1; done
	root=${dev:-$root}
fi
```

`armed` を消す処理も、FAT のパーティションをラベル(`DSM232`)で探してからマウントしている。

一つだけ、純正の `usbupg` は `/dev/sda1` しか見ないという制限が残る。USB ディスクを何本もつなぐと、この USB メモリが `sda` にならず、`app.sh` が走らないことがある。USB メモリ 1 本だけなら、ハブ越しでも `sda` になる。

## app.sh

最終的な `app.sh` はこうなった。

```sh
#!/bin/sh
rm -f "$0"	# fwimg deletes it only after we return, and the reboot may come first
# trace every step on the console (UART), next to fwimg's "Installing App.."
exec 2>&1
set -x
mark=/mnt/usb/cameo_factory/armed
# $mark left from the last arming: that USB boot did not reach Linux on the stick (it deletes $mark).
# Stay in the vendor firmware instead of a reboot loop; the next power-on tries again.
if [ -e $mark ]; then rm -f $mark; sync; exit 0; fi
env=/dlink/env
cur=$($env get bootcmd)
# no bootcmd, already armed, or a value we cannot put in '...'
case $cur in '' | *dsmusb* | *"'"*) exit 1 ;; esac
set -- "setenv bootcmd '$cur';saveenv;run dsmusb;run bootcmd" \
	'setenv dsmusb;saveenv;usb start;for d in 0 1 2 3;do if fatload usb $d:1 6000000 boot.scr;then setenv usbdev $d;source 6000000;fi;done;usb stop'
for v; do [ ${#v} -lt 240 ] || exit 1; done
touch $mark && sync || exit 1
# dsmusb first: a power cut between the two writes then leaves bootcmd unchanged.
# The vendor env tool always exits with 1, so check each write by reading it back.
$env set dsmusb "$2"
[ "$($env get dsmusb)" = "$2" ] || exit 1
$env set bootcmd "$1"
[ "$($env get bootcmd)" = "$1" ] || exit 1
sync
(sleep 2; reboot) &
```

元の `bootcmd` は書き込みのときに `$env get bootcmd` で読んだ値をそのまま使う。決め打ちにしないので、本体ごとに `bootcmd` が違っても元に戻せる。`bootcmd` に `dsmusb` がもう入っていたら、前回書いたまま U-Boot が動いていないということなので、何もしない。

`factory.img` はビルドのときに作って、USB イメージの FAT に `cameo_factory/factory.img` として入れるようにした。

## 実機で 1 回目: 再起動しない

USB イメージを書いて、挿したまま電源を入れた。UART にはこう出た。

```
Firmware found
gc_warning: Firmware Upgrading
...
Installing App..
Done.
...
upgrade success!
gc_warning: Firmware Upgrade Success
```

`fwimg` は app タイプを受け付けて、`app.sh` まで実行している。GPL 版で読んだ動きは、実機の 1.02.29 でも同じだった。ただ、そのあと再起動しない。

`app.sh` の先頭に `set -x` を入れて、各行を UART に出すようにしてもう一度試した。

```
Installing App..
+ mark=/mnt/usb/cameo_factory/armed
+ [ -e /mnt/usb/cameo_factory/armed ]
+ env=/dlink/env
+ /dlink/env get bootcmd
+ cur=dcache on;run ledon; run show_logo;boots
...
+ touch /mnt/usb/cameo_factory/armed
+ sync
+ /dlink/env set dsmusb setenv dsmusb;saveenv;usb start;for d in 0 1 2 3;do ...
+ exit 1
Done.
```

`env set` は何もエラーを出していないのに、その直後の `|| exit 1` で抜けている。`/dlink/env` の `main` を見直すと、最後がこうなっていた。

```
00008b14: mov     r3, #1
00008b18: mov     r0, r3
00008b1c: sub     sp, fp, #4
00008b20: pop     {fp, pc}
```

`get` でも `set` でも、成功しても失敗しても `main` は必ず 1 を返す。終了コードは当てにならないので、書いたあとに `get` で読み戻して、値が一致するかで確かめるようにした。

書く前に busybox のコンテナで `/dlink/env` の偽物を作り、`app.sh` の分岐をひととおり試してはいた。でもその偽物は普通に 0 を返していたので、この問題は見つからなかった。偽物は自分の思い込みで作っているので、思い込みの部分は試せない。

## 実機で 2 回目: 起動した

`factory.img` だけ作り直して USB メモリに上書きし、`armed` を消してからもう一度電源を入れた。今度は純正の上で `app.sh` が環境変数を書いて再起動し、U-Boot が `dsmusb` のループで USB メモリから `boot.scr` を読んで、Linux 7.2 と Alpine が起動した。純正 U-Boot の hush の `for` も、この書き方で問題なく動いた。

USB を抜いて電源を入れれば、純正のファームウェアがいつもどおり起動する。

## 本体に何が残るか

- **U-Boot 本体(NAND の 0x20000)**: 何も書いていない。`app.sh` が書くのは `/dlink/env` 経由の環境変数(`/dev/mtd1`)だけで、U-Boot の `saveenv` が書くのも環境変数の領域だけ。
- **環境変数(SPI の 0xc000)**: 中身は元に戻る。ただ、最後に保存するのは U-Boot の `saveenv` なので、変数の並び順や、U-Boot が起動中に足した変数(`stdin` など)の分だけ、バイト単位では元と違う可能性がある。起動には影響しない。
- **書き込み回数**: USB から 1 回起動するたびに、環境変数の領域を 4 回書き換える(純正 Linux で 2 回、U-Boot で 2 回)。
- **jffs2**: `fwimg` が `app.tar.gz` と `app.sh` を作り、終わったら消す。ファイルとしては残らない。

書き換えの途中で電源が切れると、U-Boot は環境変数が壊れていると判断して工場出荷時の値に戻す。純正は起動するが、環境変数を手で直した部分は消える。

## これから

- 環境変数がバイト単位でも元に戻っているかは、まだ確かめていない。Alpine からは SPI の先頭 128KiB が read-only の MTD として見えているので、読み出して吸い出しておいたダンプと比べる。
- USB ハブ越しの起動は、仕組みは入れたがまだ実機で試していない。U-Boot 2011.06 がハブの先の USB 1.1 機器を読めるかも分からない。
- 純正の `usbupg` が `/dev/sda` しか見ない制限は、純正側の話なので回避できない。
