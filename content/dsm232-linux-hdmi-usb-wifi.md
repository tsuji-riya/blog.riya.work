---
title: D-Link DSM-232 で Linux を動かす(2) HDMI・USB・Wi-Fi を自前で動かすまで
date: 2026-11-04
tags:
  - 組み込み
  - U-Boot
  - Linux
  - リバースエンジニアリング
  - HDMI
---

[[dsm232-uboot-linux-porting|前回]]はセキュアブートに引っかかって「別の道を探す」で終わっていた。その続き。

結論から書くと、今は USB メモリを挿して電源を入れると Linux 7.2 と Alpine が上がってきて、HDMI にコンソールが出て、内蔵の Wi-Fi で `apk add` までできるようになった。純正のフラッシュには何も書いていない。

今回はやったことをコード込みで細かめに書いておく。カーネル側は Linux 7.2 に当てるパッチで、中身の大半は `arch/arm/mach-celestial/` の C コードと Device Tree。

## 純正 U-Boot に起動してもらう

ミニローダーで署名を見ている以上、自前の U-Boot を焼いて起動するのは諦めた。代わりに純正の U-Boot 2011.06 をそのまま使う。幸い `usb start`、`fatload`、`source` は入っていたので、USB メモリの FAT にカーネル(`uImage`、zImage の後ろに DTB をくっつけたもの)と initramfs(`uInitrd`)と起動スクリプト(`boot.scr`)を置いて、UART からこれを打てば起動する。

```
usb start; fatload usb 0:1 0x6000000 boot.scr; source 0x6000000
```

`boot.scr` の中身は、要するにこういうことをしている。

```
fatload usb 0:1 0x4000000 uImage
fatload usb 0:1 0x5000000 uInitrd
setenv bootargs console=tty1 root=${rootdev} ${dispargs} ${extraargs}
# 0x5000000 に置いた initrd をそのまま使わせる(U-Boot が上に動かすと Linux の範囲外になる)
setenv initrd_high 0xffffffff
bootm 0x4000000 0x5000000
```

この U-Boot は DTB を渡せないので、カーネルは `ARM_APPENDED_DTB` で後ろにくっつけた DTB を使い、U-Boot から来る ATAG の cmdline を DT の bootargs に足す設定にしている。もう一つ、U-Boot は ATAG でメモリを 256MB と渡してくるが、上の方は純正ファームがフレームバッファやデコーダ用に使う領域なので、DT 側の bootargs に `mem=137M` を入れて Linux に使わせないようにした。

NAND と SPI は Linux 側で全部 read-only にしてある。USB を抜けば普通の DSM-232 に戻るので、気楽にいじれる。

あと前回 NAND を 512MB と書いたのは間違いで、実機の起動ログを見たら Toshiba の TC58NVG0S3E、128MiB だった。GPL ソースの bootargs に書いてあった別機種の値を信じてしまっていた。

### hush の癖

起動スクリプトを書いていて一つ変なところでつまずいた。純正 U-Boot の hush は、`$` を含む行を変数展開したあとにもう一回パースし直すらしく、空の変数が引数ごと消える。

```
# x が空だと、展開後に "test = 16" として再パースされて真になる
if test "${x}" = 16; then ...; fi
# こう書けば x が空でも "test x = x16" で偽になる
if test x${x} = x16; then ...; fi
```

おかげで最初の版は分岐が全部ずれて HDMI すら出なかった。何度も実機で試すのが面倒になったので、最後は純正 U-Boot のバイナリを Unicorn で動かして、純正の hush に `boot.scr` を食わせるスクリプトを作った。`.rel.dyn` を当てて `set_default_env` を呼び、`parse_string_outer` にスクリプトを渡すだけで、`usb` や `bootm` はスタブにしてある。

```
python3 -I tools/display_emu/run_script.py out/display_emu/uboot.bin alpine/boot.cmd hdmi=uboot
```

これで、実機に持っていく前に「この変数の組み合わせだとどのコマンドが走って、bootargs がどうなるか」が分かる。

## 最初は U-Boot の logo を借りていた

純正 U-Boot には起動ロゴを出す `logo` というコマンドがあって、これを叩くと表示系(Display Feeder、以下 DF)と HDMI 送信部が一通り初期化され、`0x0C000000` の内容が画面に出る。Linux 側はそのフレームバッファを `simple-framebuffer` で拾って fbcon で文字を描くだけ。

```dts
framebuffer@c000000 {
	compatible = "simple-framebuffer";
	reg = <0x0c000000 0x003f4800>;	/* 3840 x 1080 */
	width = <1920>;
	height = <1080>;
	stride = <3840>;
	format = "r5g6b5";
};
```

これで 1080p60 のコンソールはすぐ出た。ただ、文字がうっすらちらつく。

`logo` の後ろには `hdmi.bin` という、ソースの無い HDMI 制御のバイナリがくっついている。これも Unicorn で動かして、表示まわりの MMIO アクセスを全部記録した。周辺機器の模擬は最小限で、HDMI 送信部の状態レジスタに「ケーブル接続済み」を返し、DDC には適当な 1080p テレビの EDID を返すくらい。記録を見ると、1080p60 を出しているのに AVI InfoFrame では「640×480(VIC 1)、BT.601、アスペクト比なし」と名乗っていた。純正の製品は 1080i までしか使っていなかったので、1080p 用の表が誰にも気づかれずに残っていたんだと思う。

なのでテレビは「640×480 の映像を拡大して表示しろ」と言われているつもりで、映像処理をかけていたらしい。`logo` のあとに `mw.l` で InfoFrame を書き直した。

```
setenv avifix 'mw.l 0x804004f8 30; mw.l 0x8040050c ${avisum}; mw.l 0x80400510 52;
  mw.l 0x80400514 a8; mw.l 0x80400518 80; mw.l 0x8040051c ${avivic}; mw.l 0x80400520 0;
  mw.l 0x804004f8 33'
```

`0x804004f8` が InfoFrame の送信有効ビットで、書き換えの間だけ止めている。データバイトは DB1 = `0x52`(YCbCr 4:4:4、アンダースキャン)、DB2 = `0xa8`(BT.709、16:9)、DB3 = `0x80`(IT コンテンツ)、DB4 = VIC。1080p60 は VIC 16 でチェックサムが `0xe5`。これでちらつきは消えた。

この HDMI 送信部は Silicon Image 系で、TI OMAP4 と同じレジスタ配置だった。I2C アドレス 0x72 のページがベースアドレス + レジスタ番号×4、もう一つのページが +0x400 から並んでいる。Linux の omapdrm にある `hdmi4_core.h` がほぼそのまま辞書として使えて、だいぶ助かった。

## reboot と poweroff

ここからは U-Boot に頼らず、Linux 側で動かしていった。基本的には GPL の純正ソースから値と手順を拾って、mainline の汎用ドライバに Device Tree で渡すだけ。

reboot は、純正 BSP の `arch_reset()` が `0x801E1000` の WDT に 16bit でタイムアウト 0 を書いて有効にしているだけだった。mainline の dw_wdt は 32bit アクセスなので使わず、同じことを machine の restart に書いた。

```c
static void cnc1800l_restart(enum reboot_mode mode, const char *cmd)
{
	/* arch_reset() と同じ: TORR = 0(最短), CCVR = 0, CR = 1(有効、リセット) */
	writew(0, wdt + 0x04);
	writew(0, wdt + 0x06);
	writew(0, wdt + 0x08);
	writew(0, wdt + 0x0a);
	writew(1, wdt + 0x00);
}

DT_MACHINE_START(CELESTIAL_DT, "Celestial CNC1800L (Device Tree)")
	.dt_compat	= cnc1800l_dt_compat,
	.init_machine	= cnc1800l_init_machine,
	.restart	= cnc1800l_restart,
MACHINE_END
```

restart は割り込み禁止の状態で呼ばれるので、`ioremap` は `init_machine` で先に済ませておく。

poweroff の方は、純正ソースのどこを探しても電源を切るコードが無かった。AC アダプタ直結で、ソフトから切る回路自体が無いんだと思う。Linux は `pm_power_off` が無いと reboot システムコールの POWER_OFF を HALT に読み替えるので、何も書かなくても `poweroff` で `System halted` と出て止まる。

## USB

純正の EHCI グルー(`ehci-cnc18xx.c`)がやっているのは、クロックブロックで USB のリセットを 1 回入れて、`0xB2100218` に 3 を足すことだけだった。コメントには「enable and 48MHz」とある。これなら mainline の generic-ehci / generic-ohci で足りる。

```dts
ehci: usb@80200000 {
	compatible = "generic-ehci";
	reg = <0x80200000 0x1000>;
	interrupt-parent = <&intc>;
	interrupts = <22>;
};

ohci: usb@80210000 {
	compatible = "generic-ohci";
	reg = <0x80210000 0x1000>;
	interrupt-parent = <&intc>;
	interrupts = <27>;
};
```

リセットとクロックは machine の初期化でやる。

```c
static void __init cnc1800l_usb_init(void __iomem *clk)
{
	u32 val = readl(clk + CLOCK_SOFT_RESET);

	writel(val & ~BIT(4), clk + CLOCK_SOFT_RESET);
	udelay(100);
	writel(val | BIT(4), clk + CLOCK_SOFT_RESET);
	writel(readl(clk + CLOCK_USB_ULPI_BYPASS) | 0x3, clk + CLOCK_USB_ULPI_BYPASS);
}
```

最初は「U-Boot で USB が使えているんだから設定は残っているだろう」と思っていた。でも純正 U-Boot のパッチを見ると、`bootm` の前に呼ばれる `usb stop` の中でこのレジスタを 0 に戻していた。

```c
/* 純正 U-Boot の ehci_hcd_stop() */
*((volatile unsigned long *)0xb2100218) = 0x0;
```

もう一つ細かい話だと、純正 Linux のグルーは EHCI のレジスタを `0x80200000 + 0x100` から使っていて、U-Boot は `0x80200000` から使っている。U-Boot の方は実機で USB 起動できているので、こちらに合わせた。実際それで問題なく動いている。

## Wi-Fi

SDIO 接続の AR6003。純正 BSP では SDIO のホストが `mshci` という名前になっていたが、レジスタの定義を見たら DesignWare の SD/MMC そのものだった(CTRL が 0x00、FIFO が 0x100、という並び)。なので mainline の dw_mmc に任せる。

純正の probe はその前に、クロック出力を全部有効にして SDIO のピンマックスを設定し、GPIO 10 を High にしていた。コメントには「GPIO 10 is Wi-Fi reset」とある。

```c
/* 純正 mshci-cnc1800l.c の probe 冒頭 */
(*((volatile unsigned long *)(IO_ADDRESS(0xB2100140)))) = 0xffffffff;
(*((volatile unsigned long *)(IO_ADDRESS(0xB2110024)))) = 0x01;
mdelay(100);
gpio_hw_set_direct(10, 1);
gpio_hw_write(10, 1);
```

クロックとピンマックスは USB と同じく machine の初期化でやって、GPIO 10 は `mmc-pwrseq-simple` に任せることにした。問題は GPIO で、この SoC の GPIO は DesignWare ではない独自ブロックだった。純正ドライバを読むと方向レジスタが逆極性になっている。

```c
/* 純正 cnc18xx_gpio.c: GPIO_READ(入力)でビットを立てている */
if (dir == GPIO_READ)
	gpio_base[REG_GPIO_SWPORTA_DDR >> 2] |= flags;
else if (dir == GPIO_WRITE)
	gpio_base[REG_GPIO_SWPORTA_DDR >> 2] &= ~flags;
```

GPIO ドライバを一本書くことになるかと思ったが、mainline の汎用 gpio-mmio が「1 = 入力」の方向レジスタ(`dirin`)に対応していた。データの出力・入力・方向の 3 つのレジスタを `reg-names` で渡すだけで済んだ。

```dts
gpio: gpio@80260060 {
	compatible = "wd,mbl-gpio";
	reg = <0x80260060 0x4>, <0x80260020 0x4>, <0x80260024 0x4>;
	reg-names = "dat", "set", "dirin";
	gpio-controller;
	#gpio-cells = <2>;
};

mmc: mmc@80290000 {
	compatible = "snps,dw-mshc";
	reg = <0x80290000 0x1000>;
	interrupt-parent = <&intc>;
	interrupts = <18>;
	clocks = <&pclk>, <&pclk>;	/* 純正の get_max_clk() が 47.25MHz 固定 */
	clock-names = "biu", "ciu";
	bus-width = <4>;
	non-removable;
	cap-sdio-irq;
	cap-sd-highspeed;
	mmc-pwrseq = <&wifi_pwrseq>;
};

wifi_pwrseq: wifi-pwrseq {
	compatible = "mmc-pwrseq-simple";
	reset-gpios = <&gpio 10 GPIO_ACTIVE_LOW>;
	post-power-on-delay-ms = <100>;
};
```

チップの種類は、純正 rootfs のファームの置き場所が `ath6k/AR6003/hw2.1.1/` だったことから分かった。起動スクリプトで insmod していたのも `ath6kl.ko` で、純正も中身は mainline 系の ath6kl だった。なので mainline の `ath6kl_sdio` と linux-firmware のファームでそのまま動いた。

```
mmc0: new high speed SDIO card at address 0001
ath6kl: ar6003 hw 2.1.1 sdio fw 3.2.0.144 api 3
```

細かいところでは、Alpine の linux-firmware は zstd 圧縮なので `CONFIG_FW_LOADER_COMPRESS_ZSTD` が要った。もう一つ、ping が 27〜322ms とばらつく問題があった。`iw dev wlan0 set power_save off` で 20ms 台に落ち着いたので、今はカーネルで `# CONFIG_CFG80211_DEFAULT_PS is not set` にしている。純正の起動スクリプトも同じコマンドで省電力を切っていた。

純正の board data(送信出力の校正データ)は GPL パッケージに入っているけど、ライセンスの記載が無いのでリポジトリには入れていない。手元に置けばビルド時に取り込まれ、無ければ linux-firmware の汎用品が使われる。

## HDMI を Linux で初期化する

一番時間がかかったのがここ。

材料は 3 つ。GPL の U-Boot パッチにあるクロックと DF の設定コード、エミュレータで記録した `hdmi.bin` の書き込み列、それに GPL の Linux 側に入っていた Silicon Image の HDMI ドライバ(`SiITX*.c`)。これを並べ直して、Linux の起動時に一回だけ流すコードを書いた。

クロックは純正の `clock_dispset_clockmoderaw()` をそのまま追いかけた。TVE 用の PLL を止めて 148.5MHz に設定し直し、分周を 1080p60 用にする。

```c
static void __init cnc1800l_display_clocks(void __iomem *clk)
{
	rmw(clk + CLOCK_PLL_CLK_EN, 0x50, 0);		/* 表示と TMDS のクロックを止める */
	udelay(200);
	rmw(clk + CLOCK_PLL_SLEEP_N, BIT(1), 0);	/* TVE PLL を寝かせる */
	udelay(10);
	rmw(clk + CLOCK_PLL_SLEEP_N, BIT(0), 0);
	rmw(clk + CLOCK_SOFT_RESET, BIT(13), 0);
	/* 148.5MHz: 純正の disp_pll_cfg[] の tve_m 2, tve_fb 21 */
	rmw(clk + CLOCK_PLL_TVE_DIV, 0xffff, 0x8000 | 21 << 8 | 2);
	udelay(500);
	rmw(clk + CLOCK_PLL_SLEEP_N, 0, BIT(0));	/* 起こす */
	udelay(100);
	rmw(clk + CLOCK_PLL_SLEEP_N, 0, BIT(1));
	rmw(clk + CLOCK_TVE_DIV_N, 0xffff, 2 << 12 | 2 << 8 | 4 << 4 | 2);
	rmw(clk + CLOCK_SOFT_RESET, 0, BIT(13));
	udelay(50);
	rmw(clk + CLOCK_PLL_CLK_EN, 0, 0x50);
}
```

`BIT(13)` のところは純正のバグをそのまま写している。純正の `clock_disp_reset()` は `(_disp1) ? 0xffffdfff : 0xffffefff` と、引数ではなく定数で分岐しているので、disp0 を指定しても必ず bit13 を叩く。それで動いている以上、ここで直すと別物になるのでそのまま。

DF は、出力タイミングと画像レイヤの設定。純正の `df_reg_fmt.h` にビットフィールドの定義があるので、値の意味も分かる。

```c
writel(1, df + DF_UPDATE);			/* 設定中は反映を止める */
writel(0x4405, df + DF_OUTIF0_CTRL);		/* DispEn, IsHD, DownMode, DitherEna */
writel(1920 << 16 | 2200, df + DF_OUTIF0_X);	/* 有効, 総画素 */
writel(1080 << 16 | 1125, df + DF_OUTIF0_Y);
writel(148 << 20 | 44 << 10 | 88, df + DF_OUTIF0_HSYNC);	/* BP, 幅, FP */
writel(36 << 16 | 5 << 8 | 4, df + DF_OUTIF0_VSYNC);
writel(0, df + DF_UPDATE);

writel(1, df + DF_UPDATE);
writel(0x12, df + DF_GFX0_FORMAT);		/* RGB565 */
writel(0x0c000000, df + DF_GFX0_ADDR);
writel(3840, df + DF_GFX0_PITCH);
writel(1919 << 16, df + DF_GFX0_X);
writel(1080 << 16, df + DF_GFX0_Y);
writel(0x49, df + DF_GFX0_CTRL);		/* 有効、RGB → YUV 変換 */
writel(0, df + DF_UPDATE);
```

並べ直す途中で、エミュレータの解釈の誤りも一つ見つかった。`hdmi.bin` は U-Boot の jump table 経由で関数を呼ぶ。私はその 46 番を「TVE(アナログ出力)へのレジスタ書き込み」だと思っていた。GPL の U-Boot パッチの `include/_exports.h` を見たら、`memset` の次に足されていたのは `clock_hdmi_reset` で、引数も 0 → 1 のリセットパルスの形だった。TVE の初期化関数は 1080p だと何もせずに return するので、HDMI には関係なかった。

### 信号が出ない

最初に試したら、ログには `cnc1800l-display: HDMI 1920x1080p60` と出るのにテレビは信号なし。

ここで当てずっぽうに直すのはやめて、クロック・ピンマックス・DF・HDMI 送信部のレジスタを全部ダンプする引数(`cnc1800l_display.dump=1`)をカーネルに足した。Alpine の busybox には `devmem` が入っていなかったので、カーネルから出すことにした。HDMI 送信部はレジスタが 8bit で 4 バイトおきに並んでいるので、1 バイトずつ詰めて出す。

```c
for (o = 0; o < 0x800; o += 64) {
	for (i = 0; i < 16; i++)
		b[i] = readl(hdmi + o + i * 4);
	pr_info("cnc1800l-display: %s %08x: %16ph\n", tag, HDMI_BASE + o, b);
}
```

U-Boot の `logo` で映っている状態と、Linux で初期化した状態を並べるとこうなった。

```
映る:     before 80400000: 01 00 36 97 00 00 19 64 37 87 00 00 83 01 00 4f
映らない: after  80400000: 01 00 36 97 00 00 19 64 07 87 00 00 83 01 00 00
```

クロックと DF はきれいに一致していて、目立つ違いは 9 バイト目、HDMI 送信部の `SYS_CTRL1`(0x72:08)だった。映るときは `0x37`、自分のは `0x07`。純正の `SiITXDefs.h` を見ても bit0〜2 しか定義が無いが、Silicon Image のこの世代では bit4/5 が HSYNC / VSYNC を取り込むビット。これが落ちていて、送信部に映像が入っていなかった。送信部が測った入力の総画素数のレジスタ(0x72:3A〜3D)も、映るときは `98 08 65 04`(2200 と 1125)、自分のは 0 だった。

原因は自分のエミュレータだった。知らないレジスタは全部 0 を返すようにしていたので、`hdmi.bin` が「読んで一部だけ変えて書く」ことをしても、実機なら最初から立っているビットがエミュレータでは消える。実機のコールド状態のダンプを見たら、`SYS_CTRL1` はリセット直後から `0x74` で bit4/5 が立っていた。トレースの値をそのまま写したら、消えたビットまで写してしまっていたというわけ。DF の画像フォーマット(`0x02` と `0x12`)も同じ理由でずれていた。

### 信号は来るが真っ黒

そこを直すと、テレビは 1080p60 として認識するようになった。でも画面は真っ黒のまま。

今度は初期化を段階に分けて、カーネル引数のビットマスクで一つずつ実行できるようにした。

```c
if (steps & BIT(0))
	cnc1800l_display_clocks(clk);
if (steps & BIT(1))
	cnc1800l_display_df(df);
if (steps & BIT(2)) {
	/* hdmi.bin が最初に呼ぶ clock_hdmi_reset() と同じパルス */
	rmw(clk + CLOCK_SOFT_RESET, BIT(19), 0);
	udelay(10);
	rmw(clk + CLOCK_SOFT_RESET, 0, BIT(19));
}
if (steps & BIT(3))
	cnc1800l_display_hdmi(hdmi);
```

U-Boot で `logo` を実行して映っている状態の上から、`cnc1800l_display.steps=2` のように一つずつかけてみる。DF(2)だけなら映ったまま、HDMI 送信部のレジスタ(8)を書くと黒。

さらに絞るために、U-Boot のプロンプトだけで試した。画面の上半分を赤く塗っておいて、送信部のレジスタを一個ずつ書き換えて様子を見る。

```
mw.w 0x6100000 0 1; logo 16 2 0x6100000 1 1 0 0
mw.w 0xc000000 f800 100000            # RGB565 の赤で上半分を塗る
mw.l 0x8040003c 0                     # HDCP_CTRL = 0
```

音声の InfoFrame を止めても何も起きない。ソフトリセットをかけると色が化ける。HDCP_CTRL を 0 にすると、黒くなった。HDCP ブロックのリセットだけ解除した `4` にしてもやっぱり黒い。暗号化していないと映らない、ということになる。

GPL の Linux ドライバを読み直したら、初期化の途中にこんなコメントがあった。

```c
	// 2012-5-29 added by xp, set the default output color to black when HDCP authentication not success.
	siiTXSetVideoBlank(0x80, 0x10, 0x80);
```

この送信部は HDCP の認証が通っていない間、映像の代わりに `VID_BLANK` の単色を出す作りで、その色を黒(YCbCr で 80 10 80)にしてあった。黒い画面の正体はこれだった。

### HDCP の認証を書く

それならと、同じドライバの `siiTXHDCP_Authentication()` と同じ手順を Linux 側に書いた。テレビ側の HDCP は DDC の I2C アドレス 0x74 にいて、送信部の DDC マスタ(0x72:ED〜F5)経由で読み書きする。読み出しはこんな感じ。

```c
static void ddc_start(void __iomem *hdmi, u8 off, u8 len)
{
	hdmi_sys(hdmi, SYS_DDC_CMD, DDC_CMD_CLEAR_FIFO);
	hdmi_sys(hdmi, SYS_DDC_ADDR, HDCP_RX);		/* 0x74 */
	hdmi_sys(hdmi, SYS_DDC_ADDR + 1, 0);		/* segment */
	hdmi_sys(hdmi, SYS_DDC_ADDR + 2, off);
	hdmi_sys(hdmi, SYS_DDC_ADDR + 3, len);
	hdmi_sys(hdmi, SYS_DDC_ADDR + 4, 0);
}

static int hdcp_rx_read(void __iomem *hdmi, u8 off, u8 *buf, u8 len)
{
	int i = 0, tries = len + 50;

	ddc_start(hdmi, off, len);
	hdmi_sys(hdmi, SYS_DDC_CMD, DDC_CMD_SEQ_RD);
	while (i < len && tries > 0) {
		if (hdmi_rd(hdmi, SYS_DDC_FIFO_CNT)) {
			buf[i++] = hdmi_rd(hdmi, SYS_DDC_DATA);
		} else {
			tries--;
			usleep_range(2000, 3000);
		}
	}
	return ddc_done(hdmi, i == len);
}
```

認証の本体はこう。テレビの BKSV を読み、送信部が作った乱数 An と送信部の AKSV をテレビに書き、BKSV を送信部に書くと R0 の計算が始まる。両側の R0 が一致したら暗号化を有効にする。

```c
	if (hdcp_rx_read(hdmi, RX_BKSV, bksv, 5))
		return -EIO;
	for (i = 0; i < 5; i++)
		ones += hweight8(bksv[i]);
	if (ones != 20)		/* KSV は 1 と 0 が 20 個ずつ */
		return -ENXIO;

	ctrl = (hdmi_rd(hdmi, SYS_HDCP_CTRL) & ~HDCP_ENC_EN) | HDCP_CP_RESETN;
	hdmi_sys(hdmi, SYS_HDCP_CTRL, ctrl);
	/* 乱数生成器を少し回してから止める(純正の GenerateAN()) */
	hdmi_sys(hdmi, SYS_HDCP_CTRL, ctrl & ~HDCP_AN_STOP);
	msleep(10);
	hdmi_sys(hdmi, SYS_HDCP_CTRL, ctrl | HDCP_AN_STOP);

	for (i = 0; i < 8; i++)
		an[i] = hdmi_rd(hdmi, SYS_HDCP_AN + i);
	if (hdcp_rx_write(hdmi, RX_AN, an, 8))
		return -EIO;
	for (i = 0; i < 5; i++)
		aksv[i] = hdmi_rd(hdmi, SYS_HDCP_AKSV + i);
	if (hdcp_rx_write(hdmi, RX_AKSV, aksv, 5))
		return -EIO;
	msleep(150);

	/* BKSV を送信部に書くと R0 の計算が始まる */
	for (i = 0; i < 5; i++)
		hdmi_sys(hdmi, SYS_HDCP_BKSV + i, bksv[i]);
	/* ... Ri ready を待って、テレビの R0(0x08)と送信部の Ri を比べる ... */

	hdmi_sys(hdmi, SYS_HDCP_CTRL, hdmi_rd(hdmi, SYS_HDCP_CTRL) | HDCP_ENC_EN);
```

鍵は送信部のハードウェアに入っていて、AKSV はコールド状態のダンプにも最初から出ていた。こちらからは触らない。純正ドライバが毎回やっていることを Linux から呼んでいるだけだ。暗号化の切り替えは AVMUTE の間にやる、という純正のお作法(`siiTXSendCP_Packet()`)にも合わせている。

一回目は BKSV がおかしな値になって失敗した。テレビの HDCP の受け口は、信号が来てからしばらく DDC にちゃんと答えてくれないらしい。純正 U-Boot のログを見返すと、あちらも最初の何回かは失敗してから認証が通っていた。

```
###################DDC Time Out!################
{DownStreamAuthState}: No HDCP Dev with HDCP Slave Addr
...
{DownStreamAuthState}: Authenticated
```

なので認証を起動処理から外して、delayed work で 1 秒おきに再試行するようにした。

```
cnc1800l-display: HDCP not ready (-19), retrying
cnc1800l-display: HDCP authenticated (try 2)
```

これで、やっと U-Boot の `logo` なしで tty が出た。

### ホットプラグ

ケーブルを抜くと HDCP は切れるので、挿し直されたら送信部の設定と認証をやり直す必要がある。純正の `siiTXHDCP_CheckHPDChanged()` は、割り込みレジスタ(0x72:71)の bit6 で HPD の変化を見ていたので、最初は同じようにした。

すると、画面が数秒おきについたり消えたりし始めた。ログはこうなっていた。

```
cnc1800l-display: HDMI plugged, setting up again
cnc1800l-display: HDCP authenticated (try 1)
cnc1800l-display: HDMI plugged, setting up again
cnc1800l-display: HDCP authenticated (try 1)
...
```

送信部を設定し直すと、その処理自体がこのフラグを立ててしまう。それを次の確認で「挿し直された」と読んでまた設定し直す、というループだった。結局フラグは見るのをやめて、HPD のレベルが一度落ちてまた立ったら挿し直し、と判断するようにした。テレビの入力切り替えみたいに HPD が変わらないまま HDCP だけ切れることもあるので、そっちは 2 秒ごとに HDCP のリンク確認値 Ri をテレビ側と比べて拾う。Ri は 128 フレームごとに更新されるので、テレビ側を読む前後で送信部の値を 2 回読み、どちらかと合えば良しにしている。

```c
static void cnc1800l_display_hdcp_work(struct work_struct *work)
{
	...
	if (!(hdmi_rd(hdmi, SYS_SYS_STAT) & SYS_STAT_HPD)) {
		hdcp_on = false;
		hdcp_unplugged = true;
		goto again;
	}
	replug = hdcp_unplugged;
	hdcp_unplugged = false;
	if (hdcp_on && !replug) {
		if (++hdcp_polls % 2)
			goto again;
		if (hdcp_link_ok(hdmi)) {
			hdcp_bad = 0;
			goto again;
		}
		if (++hdcp_bad < 2)	/* DDC の読み損ない 1 回では動かない */
			goto again;
	}
	if (hdcp_on || replug)
		cnc1800l_display_hdmi(hdmi);	/* 送信部を設定し直す */

	hdmi_avmute(hdmi, true);
	ret = cnc1800l_display_hdcp(hdmi);
	hdmi_avmute(hdmi, false);
	...
again:
	schedule_delayed_work(to_delayed_work(work), HZ);
}
```

今はケーブルを抜き差ししても、入力を切り替えて戻しても、数秒で画面が戻る。

```
cnc1800l-display: HDMI unplugged
cnc1800l-display: HDMI plugged, setting up again
cnc1800l-display: HDCP not ready (-5), retrying
cnc1800l-display: HDCP authenticated (try 2)
cnc1800l-display: HDMI link lost, setting up again
cnc1800l-display: HDCP not ready (-5), retrying
cnc1800l-display: HDCP authenticated (try 2)
```

## rootfs を USB の ext4 に

initramfs だけだと設定が再起動で消えるので、USB イメージを 512MB にして、2 個目のパーティションに ext4 の rootfs を入れた。ビルドは Docker の中で、root 権限もループデバイスも使わずにやりたい。そこで `fakeroot` の中で所有者を root に `chown` してから、`mke2fs -d` でディレクトリをそのまま ext4 に焼いている。

```sh
truncate -s 448M out/root.img
fakeroot sh -c 'chown -R 0:0 alpine/out/rootfs && mke2fs -q -t ext4 -L dsm232root -d alpine/out/rootfs out/root.img'
truncate -s 512M "$img"
printf 'label: dos\nstart=2048, size=129024, type=6, bootable\nstart=131072, type=83\n' | sfdisk -q "$img"
dd if=out/part.img of="$img" bs=1M seek=1 conv=notrunc status=none
dd if=out/root.img of="$img" bs=1M seek=64 conv=notrunc,sparse status=none
```

切り替えは initramfs の `/init` がやる。`boot.scr` が渡す `root=/dev/sda2` を見て、USB ストレージが出てくるのを待ってから `switch_root` する。見つからなければそのまま initramfs で動く。

```sh
for a in $(cat /proc/cmdline); do case $a in root=*) root=${a#root=} ;; esac; done
if [ "${root#/dev/}" != "${root:-}" ]; then
	for i in $(seq 20); do [ -b "$root" ] && break; echo "waiting for $root"; sleep 1; done
	mkdir -p /newroot
	if mount "$root" /newroot && [ -x /newroot/sbin/init ]; then
		for m in dev proc sys run; do mount --move /$m /newroot/$m; done
		exec switch_root /newroot /sbin/init
	fi
fi
exec /bin/busybox init
```

ext4 に移って `apk update` したら、今度はこう言われた。

```
WARNING: updating and opening https://dl-cdn.alpinelinux.org/alpine/v3.24/main/armhf/APKINDEX.tar.gz: TLS: server certificate not trusted
```

この機械には RTC が無くて、起動直後の時計は 1970 年になっている。そりゃ証明書は有効期間前だ。`/etc/wpa_supplicant.conf` を一度作っておけば、起動時に Wi-Fi につないで NTP で時刻を合わせるスクリプトを inittab から走らせるようにした。

```sh
#!/bin/sh
conf=/etc/wpa_supplicant.conf
[ -f "$conf" ] || exit 0
for i in $(seq 30); do [ -e /sys/class/net/wlan0 ] && break; sleep 1; done
wpa_supplicant -B -i wlan0 -c "$conf" || exit 1
udhcpc -i wlan0 -b
ntpd -p pool.ntp.org
```

## これから

セキュアブートには手を付けず、純正 U-Boot から USB で起動するという形で、HDMI、USB、Wi-Fi、reboot まで実機で動くところまで来た。HDMI は「エミュレータが 0 を返していた」のと「HDCP が通らないと映らない」の二つで長いことハマったけど、どちらも推測をやめてレジスタを見比べたところから進んだ。

音声と EDID(今は 1080p60 決め打ち)はまだ。Wi-Fi も純正の校正データは使っていないし、MAC アドレスも汎用 board data のものになっている。USB なしで起動させるには NAND を書き換える必要があるけど、純正の起動経路を壊さずにやる方法が見つかるまでは手を出さないでおく。
