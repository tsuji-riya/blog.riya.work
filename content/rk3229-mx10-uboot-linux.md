---
title: AliExpress で買った安い TV box (RK3229) に U-Boot と Linux を載せた話
date: 2026-10-13
tags:
  - 組み込み
  - U-Boot
  - Linux
  - Rockchip
  - リバースエンジニアリング
---

最近ラズパイの値段が高い。ちょっと Linux が動く小さいボードが欲しいだけなのに、あの値段を出すのはなんか違うなと思って、AliExpress で TV box の基板を探して買った。Rockchip RK3229 の "MX10" 系というやつで、DDR3 が 1GiB、eMMC が 8GiB。Android TV box としてはよくある構成で、値段はラズパイよりだいぶ安い。

せっかくなので Android は消さずに、SD カードから **mainline の U-Boot と Linux** を起動するところまでやってみた。

## まず困ったこと: GPL ソースが無い

こういう箱は Android 7.1.2 + Rockchip の SDK で動いていて、カーネルは 3.10.104 のベンダー版。本来なら GPL なのでソースが出ているはずなんだけど、この箱にはソースが付いてこなかった。

ソースがないなら実物から読むしかない、ということで方針を決めた。

- 内蔵ストレージ全体をダンプして、パーティション(`kernel` / `resource` / `uboot` / `trust` など)に切り分ける
- `resource` パーティションの中に、ベンダーカーネルが使っている DTB(`rk-kernel.dtb`)が入っているので、それを取り出して dts に戻す
- それを見ながら、同じ SoC の mainline の dts(`rk3229-xms6` が近い)を下敷きにして、この箱用の dts を書く
- ベンダーの U-Boot やブートローダは `strings` で眺めて、何ができて何ができないかを調べる

ベンダーのバイナリを逆アセンブルして中身を再現する、みたいな方向には行かなかった。必要だったのは「どのペリフェラルがどのアドレスで、どのクロック・電源・ピンを使っているか」という情報で、それは DTB にほぼ全部書いてある。ここが一番ありがたかった。

あとブートローダの段階で必要な `trust.img`(TOS)は、箱純正のものをそのまま使うことにした。DTS の PSCI の書き方がそれを前提にしているので、ここを自前で作り直す理由がなかった。

## 起動の仕組みを調べる

ダンプを読んで分かったこと。

- RK3229 の BootROM は **USB メモリから起動できない**。USB は PC から書き込むための maskrom モードだけ
- 一方で **SD カードからは BootROM が直接起動できる**。内蔵に Android が入ったままでも、SD に有効なローダがあればそちらが使われる(Multitool for RK3228/RK3229 がやっているのと同じ)
- ベンダーの U-Boot には USB ホスト(マスストレージ)のコードが無かった

なので内蔵フラッシュには一切書かずに、SD カードに

1. 第一段ローダ(DDR 初期化 + miniloader、sector 0x40)
2. 自分でビルドした mainline U-Boot(sector 0x4000)
3. 純正の trust(sector 0x6000)
4. FAT32 の `/boot`(extlinux.conf、zImage、dtb)と ext4 の rootfs(Alpine Linux)

を並べたイメージを作る方式にした。第一段ローダは Multitool のものをそのまま使っている。

ベンダーの U-Boot にはセキュアブート関連の文字列(`loader's key not match` とか)が入っていたので少し身構えたけど、この SD の構成で自前の U-Boot はちゃんと動いた。少なくともこの箱では、SD 経由で署名なしのローダが普通に起動する。

## 実際にハマったところ

### Linux が HDMI の初期化で止まる

U-Boot は HDMI に出るのに、Linux は

```
dwhdmi-rockchip 200a0000.hdmi: supply avdd-1v8 not found, using dummy regulator
```

を最後にハングする。デバッグ用の `pr_info` を `dw-hdmi.c` に仕込んで GRF / CRU のレジスタを出してみたら、HDMI コントローラのレジスタを読む前に止まっていた。

原因は電源ドメイン。RK322x の HDMI コントローラは `PD_VIO` というドメインの中にいて、ブートローダがこれを idle のままにしていると、最初のレジスタ読み出しでバスごと止まる。`rk322x.dtsi` の hdmi ノードに

```
power-domains = <&power RK3228_PD_VIO>;
```

を1行足すだけで直った。原因が分かれば1行なのが悔しい。

### U-Boot の USB キーボードが反応しない

手元にあった Apple の Magic Keyboard(2024年版の USB-C)を U-Boot のコンソール入力に使おうとしたら、`usb tree` には出るのに `stdin` に `usbkbd` が出てこない。

このキーボードは interface 0 が "Device Management"、interface 1 が Boot Keyboard という構成で、U-Boot の USB ドライバの照合は interface 0 とデバイス ID しか見ない。だから汎用のデバイスドライバに束縛されてしまっていた。デバイス ID (`05ac:0320` など)をキーボードドライバのテーブルに足して解決。

その後も、

- Apple のキーボード用の quirk が、割り込みキューを作る前に return していた
- このボードのフルスピード機器は OHCI コントローラにつながり、OHCI の制御転送ポーリングは使い物にならなかったので、割り込みキュー方式に切り替えた

と続いて、ようやく入力できるようになった。

### キー入力が 3 回に 1 回くらい抜ける

動くようになったと思ったら、打っても 1/3 くらいの文字が入らない。OHCI の割り込みキューが、DMA で書き込まれたバッファのキャッシュを無効化しないまま読んでいたのが原因だった。古いキャッシュを読んでキー押下を取りこぼしていたわけで、読む前にキャッシュを invalidate するようにしたら抜けなくなった。

## ビルド環境

作業は Mac だったので、Docker のコンテナ内でビルドしている。Linux のソースを Mac のファイルシステムに置くと大文字小文字の区別がなくて(`xt_CONNMARK.h` と `xt_connmark.h` みたいな衝突で)ずっと dirty になってしまうので、clone は Docker ボリュームの中でやる。`-j8`。

ソースは普通のフォークにして、ビルドスクリプトや SD イメージを作るツール類は別のリポジトリにまとめた。

- U-Boot: [tsuji-riya/u-boot](https://github.com/tsuji-riya/u-boot) の `rk3229` ブランチ(v2026.07 ベース)
- Linux: [tsuji-riya/linux](https://github.com/tsuji-riya/linux) の `rk3229` ブランチ(v7.2 ベース)
- ツール類: Docker のビルドスクリプト、SD イメージの組み立て、rootfs の設定など(別リポジトリ)

## 今の状態

動いたもの:

- SD カードからの起動
- U-Boot のシリアルと HDMI のコンソール、USB キーボード
- Linux 7.2 の起動、HDMI 出力、Alpine Linux のログイン

まだ確かめていないもの: Ethernet、Wi-Fi、音声、USB メモリからの起動(U-Boot 側は USB を先に探す設定にはしてある)。

ラズパイほど手軽ではないけど、安い基板でも DTB さえ取り出せれば mainline まで持っていける、というのは分かった。GPL ソースが出ていない機器でも、手元の実物から読めるものは読める。次は Ethernet あたりを見る予定。
