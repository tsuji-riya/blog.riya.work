---
title: D-Link DSM-232 で Linux を動かす(3) ドライバを作り直して mainline に寄せる
date: 2026-11-20
tags:
  - 組み込み
  - Linux
  - リバースエンジニアリング
  - HDMI
  - DRM
---

[[dsm232-linux-hdmi-usb-wifi|前回]]で HDMI・USB・Wi-Fi は動くようになった。ただ中身を見返すと、`arch/arm/mach-celestial/` の C コードが 600 行近くあって、そこで USB のリセットも、SDIO のクロックも、HDMI の初期化も、起動時に一回だけまとめて叩いていた。動きはするけど、Linux のやり方からはだいぶ外れている。

なので今回は、汎用ドライバで代わりが利くところはそちらに任せ、無いものは普通の Linux ドライバとして書き直した。ついでに、前回やり残していた EDID、CEC、`/dev/watchdog`、ボタン、Wi-Fi の MAC アドレスまで片付けた。

今回の作業は Claude Code に実機への SSH を渡して、パッチの作り直しからビルド、実機での試験まで任せて進めた。

結果、パッチは 13 本になった。machine の C コードは無くなり、`mach-celestial` に残っているのは Kconfig だけ。汎用の DT マシンとして起動する。

| パッチ     | 中身                                              |
| ---------- | ------------------------------------------------- |
| 0001       | SoC と DSM-232 の基本(タイマ、割り込み、UART、DT) |
| 0002       | spi-dw の 16bit アクセス                          |
| 0003       | NAND コントローラと SPI フラッシュ                |
| 0004       | GPIO(新規ドライバ)                                |
| 0005       | クロック(新規ドライバ)、リセット、ピンマックス    |
| 0006       | ウォッチドッグと reboot                           |
| 0007       | USB                                               |
| 0008       | SDIO と Wi-Fi                                     |
| 0009       | HDMI の DRM ドライバと CEC                        |
| 0010       | LED、ボタン、kexec                                |
| 0011〜0013 | Wi-Fi の MAC を U-Boot の環境変数から取る         |

追加行は前の 1642 行から 2551 行に増えた。DRM ドライバと CEC、それに DT バインディングを足した分で、machine のコードは 588 行から 0 行になっている。

## 試験のやり方を変えた

前回までは、カーネルを作るたびに USB メモリを抜いて書き直していた。今回は Alpine のイメージに dropbear(SSH サーバ)と kexec-tools を入れて、試験用のカーネルを SSH 越しに kexec で起動するようにした。

```sh
# initrd は USB の FAT にある uInitrd から(U-Boot のヘッダ 64 byte を飛ばす)
dd if=/mnt/boot/uInitrd of=/tmp/initrd bs=64 skip=1
# Wi-Fi の設定と SSH のホスト鍵も cpio で足しておく。USB の root が上がらなくても SSH はつながる
(cd / && ls etc/wpa_supplicant.conf etc/dropbear/* | cpio -o -H newc | gzip) >> /tmp/initrd
kexec -l /tmp/zImage --dtb=/tmp/cnc1800l-dsm232.dtb --initrd=/tmp/initrd \
	--command-line="$(cat /proc/cmdline) panic=10"
setsid sh -c 'sleep 1; sync; kexec -e' &
```

失敗したら電源を入れ直せば USB の普通のカーネルに戻るし、`panic=10` を付けておけば panic でも勝手に再起動する。

細かいところで 2 回つまずいた。1 回目は `kexec -e` を SSH のコマンドの最後に `&` で付けていたら、SSH が切れたときに一緒に殺されて何も起きなかった。`setsid` で切り離して解決。2 回目は `--initrd` を付け忘れていて、`VFS: Cannot open root device "LABEL=dsm232root"` で panic した。root を LABEL で探すのは initramfs の `/init` の仕事なので、initramfs が無いとカーネルには見つけられない。

## mainline のドライバに任せたもの

### ウォッチドッグと reboot

前回は「純正の `arch_reset()` が 16bit アクセスだから」と、dw_wdt を使わずに machine の restart を書いていた。ところが純正 Linux のウォッチドッグドライバ(`cnc18xx_wdt.c`)の方は、普通に 32bit でアクセスしていた。実機で 32bit 読みをすると上位 16bit が 0 で返ってくる。それでも使う値は下位 16bit に収まっていて、バージョンレジスタが `0x322A` と半分だけ読めてしまう以外は困らなかった。

```dts
/* the vendor cnc18xx_wdt.c uses 32 bit accesses, TOP 15 = ~40s at PCLK */
watchdog@801e1000 {
	compatible = "snps,dw-wdt";
	reg = <0x801e1000 0x100>;
	clocks = <&pclk>;
};
```

これで mainline の dw_wdt が `/dev/watchdog` と restart ハンドラの両方を出してくれる。`reboot` で再起動することも実機で確かめた。

### リセットとピンマックス

純正の `clock_*_reset()` は、クロックブロックの `0xB2100200` のビットを 0 にしてリセット、1 に戻して解除、という作りだった。これは mainline の reset-simple がそのまま扱える(`snps,dw-low-reset` が「0 でリセット」の互換)。

ピンマックスは 1 機能 1 ビットで、純正の `cnc1800l_mux.c` は 16bit でアクセスしていた。これも pinctrl-single の `bit-per-mux` で書ける。

```dts
rst: reset-controller@b2100200 {
	compatible = "snps,dw-low-reset";
	reg = <0xb2100200 0x4>;
	#reset-cells = <1>;
};

pinmux: pinmux@b2110000 {
	compatible = "pinctrl-single";
	reg = <0xb2110000 0x30>;
	#pinctrl-cells = <2>;
	pinctrl-single,bit-per-mux;
	pinctrl-single,register-width = <16>;
	pinctrl-single,function-mask = <0x1>;

	sdio_pins: sdio-pins {
		pinctrl-single,bits = <0x24 0x1 0x1>;
	};
};
```

USB の EHCI/OHCI は `resets = <&rst 4>` とクロックを DT で渡すだけになり、machine の初期化からリセットのコードが消えた。SDIO は `pinctrl-0 = <&sdio_pins>` で dw_mmc がピンを切り替える。

一つ注意点があった。dw-low-reset には「一回リセットを入れる」操作(`.reset`)が無いので、HDMI の送信部のようにパルスを入れたいところは assert → `udelay` → deassert と書く。

### LED とボタン

GPIO 2 が前面の LED、GPIO 9 がボタン。純正の U-Boot 環境変数に `ledon=gpio write 2 1` があり、純正の `csgpio.c` はボタンを「押している間 0」で読んでいた。それぞれ gpio-leds と gpio-keys に任せた。ボタンは何の機能か分からないので、`KEY_PROG1` にしてある。

```dts
keys {
	compatible = "gpio-keys";

	button {
		label = "button";
		linux,code = <KEY_PROG1>;
		gpios = <&gpio 9 GPIO_ACTIVE_LOW>;
	};
};
```

## 自分で書いたドライバ

### GPIO

前回は mainline の gpio-mmio で済ませていた。でも gpio-mmio で扱えるのは最初の 32 本(ポート A)だけで、割り込みも使えない。ボタンを gpio-keys で読むには割り込みが要るので、ドライバを書いた。41 本、ポート A と B の 2 つ。

純正の `cnc18xx_gpio.c` を読むと、割り込みはレベルだけで、極性レジスタの意味がデータシートと逆だったらしい。コメントにそのまま「the document is wrong」と書いてあった。実際、極性ビット 1 が「Low の間に割り込み」だった。

エッジ割り込みが無いので、エッジは極性を切り替えて作る。今のレベルの「逆」を待つように極性を設定しておき、割り込みが来たら今のレベルを読んで、また逆を待つ。

```c
/* wait for the level that is not there now: the next edge */
static void cnc_gpio_arm_edge(struct cnc_gpio *g, unsigned int off)
{
	/* polarity 1 = interrupt while low (vendor: "the document is wrong") */
	cnc_gpio_rmw(g, PORT(off)->pol, BIT_OF(off), cnc_gpio_get(&g->gc, off));
}
```

割り込みハンドラでは、エッジ扱いのピンなら極性を付け直し、立ち上がりだけ・立ち下がりだけを頼まれていれば、合わない方は捨てる。

```c
if (edges & BIT(bit)) {
	int level = cnc_gpio_get(gc, off);

	cnc_gpio_arm_edge(g, off);
	if (!((level ? g->rising[p] : g->falling[p]) & BIT(bit)))
		continue;
}
generic_handle_domain_irq(gc->irq.domain, off);
```

ボタンを何回か押すと、`KEY_PROG1` の 1 と 0 がきれいに対で出た。

### クロック

前回 machine のコードで直接叩いていたクロックを、common clock framework のドライバにした。扱うのは表示用の PLL(TVE PLL)と、その下の表示用・TMDS 用の分周、それとクロックの有効ビットいくつか。それ以外のクロックは、ブートローダが設定したままの固定クロックとして DT に書いてある。

PLL は 27MHz ×(fb + 1)/ m で、設定値は純正の `disp_pll_cfg[]` の表から取った。任意の周波数は作らず、表にある組み合わせから一番近いものを選ぶ。

```c
/* PLL settings of the vendor disp_pll_cfg[] table: 27MHz * (fb + 1) / m */
static const struct { u8 m, fb; } cnc_pll_cfg[] = {
	{ 4, 15 },	/* 108MHz: 27MHz SD modes, / 4 */
	{ 2, 21 },	/* 297MHz: 148.5 and 74.25MHz HD modes, / 2 and / 4 */
	{ 15, 27 },	/* 50.4MHz: 640x480 (25.2MHz), / 2 */
	{ 5, 23 },	/* 129.6MHz: 1024x768 (64.8MHz), / 2 */
	{ 2, 15 },	/* 216MHz: 1280x1024 (108MHz), / 2 */
};
```

周波数を変えるときは、純正の `clock_disptve_pllmode()` と同じく、スリープビットを bit1 → bit0 の順に落として設定を書き、500µs 待ってから bit0 → bit1 の順に起こす。分周は mainline の `clk-divider`、有効ビットは `clk-gate` をそのまま使った。

途中のコミットで一度、起動すると画面が消えるようになった。表示用のクロックを DRM ドライバより先のコミットでクロックドライバに載せたので、その時点では誰も使っておらず、起動の最後の `clk_disable_unused` に止められていた。表示と TMDS のゲートに `CLK_IGNORE_UNUSED` を付けて解決。

### HDMI を DRM ドライバに

前回の HDMI は、起動時に 1080p60 決め打ちで一回初期化し、`simple-framebuffer` で fbcon に渡していた。今回はこれを `drivers/gpu/drm/tiny/` の DRM ドライバ(simple display pipe)にした。これで EDID を読んでモードを選べるし、解像度もピクセルフォーマットも切り替えられる。

EDID は、HDMI 送信部の DDC マスタ(前回 HDCP で使ったもの)を I2C アダプタとして出して、DRM の `drm_edid_read_ddc()` に読ませている。DDC マスタは「アドレス・オフセット・長さ」を渡して読み書きするハードなので、I2C のメッセージ列をその形に読み替える。E-DDC のセグメント付き読み出しにも対応した。

```c
if (num == 3 && m->addr == DDC_SEGMENT_ADDR && m->len == 1 && !(m->flags & I2C_M_RD)) {
	seg = m->buf[0];
	cmd = DDC_CMD_ENH_RD;
	m++;
	num--;
}
if (num == 1 && (m->flags & I2C_M_RD))
	ret = ddc_run(p, m->addr, 0, 0, DDC_CMD_CUR_RD, m->buf, m->len);
else if (num == 1 && m->len)
	ret = ddc_run(p, m->addr, 0, m->buf[0], DDC_CMD_SEQ_WR, m->buf + 1, m->len - 1);
else if (num == 2 && m->len == 1 && !(m->flags & I2C_M_RD) &&
	 (m[1].flags & I2C_M_RD) && m[1].addr == m->addr)
	ret = ddc_run(p, m->addr, seg, m->buf[0], cmd, m[1].buf, m[1].len);
else
	return -EOPNOTSUPP;
```

HDCP の認証も同じアダプタの上で動くので、前回の DDC のコードはここに一本化された。

出せるモードは、クロックドライバの `clk_round_rate()` に聞いて決める。HDMI が許す ±0.5% に収まらないものは捨てる。

```c
static enum drm_mode_status cnc_mode_valid(struct drm_simple_display_pipe *pipe,
					   const struct drm_display_mode *mode)
{
	struct cnc_drm *p = container_of(pipe, struct cnc_drm, pipe);
	long rate = mode->clock * 1000L;

	if (mode->flags & DRM_MODE_FLAG_INTERLACE)
		return MODE_NO_INTERLACE;
	if (mode->clock > 148500)
		return MODE_CLOCK_HIGH;
	/* +-0.5% allowed by HDMI */
	if (abs_diff(clk_round_rate(p->disp_clk, rate), rate) > rate / 200)
		return MODE_NOCLOCK;
	return MODE_OK;
}
```

表示を有効にする処理は、前回の初期化コードをモードに合わせて一般化したもの。前回手で組み立てていた AVI InfoFrame も、DRM の `drm_hdmi_avi_infoframe_from_display_mode()` に作らせて、色空間(YCbCr 4:4:4)とアンダースキャンだけ上書きしている。

```c
reset_control_assert(p->disp_rst);
clk_set_rate(p->disp_clk, m->clock * 1000);
clk_set_rate(p->tmds_clk, m->clock * 1000);
reset_control_deassert(p->disp_rst);
clk_prepare_enable(p->disp_clk);

writel(1, p->df + DF_UPDATE);
writel(ctrl, p->df + DF_OUT_CTRL);
writel(m->hdisplay << 16 | m->htotal, p->df + DF_OUT_X);
writel(m->vdisplay << 16 | m->vtotal, p->df + DF_OUT_Y);
writel((m->htotal - m->hsync_end) << 20 | (m->hsync_end - m->hsync_start) << 10 |
       (m->hsync_start - m->hdisplay), p->df + DF_OUT_HSYNC);
...
drm_hdmi_avi_infoframe_from_display_mode(&avi, conn, m);
avi.colorspace = HDMI_COLORSPACE_YUV444;
avi.colorimetry = hd ? HDMI_COLORIMETRY_ITU_709 : HDMI_COLORIMETRY_ITU_601;
avi.scan_mode = HDMI_SCAN_MODE_UNDERSCAN;
```

手元の Xiaomi のモニターで、EDID が読めて、1080p60(XRGB8888 と RGB565)、1080p50、720p60、480p、1024x768 のカラーバーが正しい色で出た。HDCP の認証は毎回 1〜2 回目で通る。驚いたのは TMDS 側の PLL 設定で、リセット値のままで 25MHz から 148.5MHz まで全部出た。

EDID が読めなかったときは、HDMI のテレビだとみなす。これまで試したテレビは全部 HDMI だったので。

### CEC

HDMI 送信部には CEC のブロック(Silicon Image の CPI)も入っていて、レジスタは送信部の `+0x1000` から並んでいた。純正の Silicon Image のドライバを辞書にして、Linux の CEC フレームワークのアダプタを書いた。割り込みは使わずに 20ms ごとに状態を見ている。

```c
if (p->cec_tx_wait) {
	if (s0 & CEC_TX_SENT) {
		p->cec_tx_wait = 0;
		cec_transmit_attempt_done(p->cec, CEC_TX_STATUS_OK);
	} else if (s1 & CEC_TX_RETRY_OV) {
		p->cec_tx_wait = 0;
		cec_transmit_attempt_done(p->cec, CEC_TX_STATUS_NACK);
	} else if (!--p->cec_tx_wait) {
		cec_transmit_attempt_done(p->cec, CEC_TX_STATUS_ERROR);
	}
}
```

論理アドレス 4 を取り、物理アドレス 1.0.0.0 で送信すると、「ACK なし」の検出までは動いた。ただ手元のモニターは CEC に対応していないので、相手との通信はまだ確かめられていない。

ここで一回、本体を固めた。CEC のレジスタを devmem2 で読んだら、そのまま何も応答しなくなって、電源を抜くしかなかった。CEC のクロック(クロック有効レジスタの bit 26)が止まっている間にそのブロックへアクセスすると、バスごと止まるらしい。ドライバは CEC のクロックを先に有効にしてから触るようにしてある。

## Wi-Fi の MAC アドレスを U-Boot の環境変数から取る

ここが今回一番の寄り道だった。

前回の時点では、Wi-Fi は linux-firmware の汎用 board data で動いていた。純正の board data(`bdata_jp.bin`)はライセンスの記載が無いのでリポジトリには入れていないが、手元の GPL パッケージから取り出してビルド時に入れると、警告は消えた。

```
ath6kl: Failed to get board file ath6k/AR6003/hw2.1.1/bdata.bin (-2), trying to find default board file.
ath6kl: WARNING! No proper board file was not found, instead using a default board file.
```

その代わり、MAC アドレスが変わって DHCP の IP も変わった。新しい MAC は `40:2c:f4:…` で始まる AzureWave(Wi-Fi モジュールのメーカー)のもの。一方、純正の `start_ath.sh` は U-Boot の環境変数 `wifiaddr`(D-Link の `C8:D3:A3:…`)を `softmac` というファイルに書いてドライバに渡していた。mainline の ath6kl にはこの仕組みが無い。

### board data を書き換えても効かない

AR6003 は board data の 0x16 にある MAC を使い、0x04 には「全体の 16bit ワードを XOR すると決まった値になる」ためのチェックサムがある。なので最初は、board data の MAC を書き換えてチェックサムを直し、ドライバを unbind / bind して読み込み直してみた。

結果、MAC は変わらなかった。そもそも board data ファイルに書いてある MAC は `40:2c:f4:de:7d:43` で、実際の MAC とも違う。AR6003 は起動時にモジュールの OTP(一回だけ書き込めるメモリ)のコードを走らせ、そこに焼かれた MAC で board data を上書きしているようだ。ath6kl は `ip link set address` にも対応していない。

### OTP の後で書き込む

ath6kl の起動手順は、board data をターゲットの RAM に転送 → OTP を実行 → ファームを転送、という順番になっている。だったら、OTP が終わった後で RAM 上の board data の MAC を書き換えればいい。チェックサムは XOR なので、変えたバイトの差分を XOR し込めば保てる。

```c
static int ath6kl_set_dt_mac_address(struct ath6kl *ar)
{
	u8 mac[ETH_ALEN], buf[AR6003_MAC_ADDRESS_OFFSET + ETH_ALEN];
	u32 address;
	int i, ret;

	if (ar->target_type != TARGET_TYPE_AR6003)
		return 0;

	ret = of_get_mac_address(ar->dev->of_node, mac);
	if (ret == -EPROBE_DEFER)
		ath6kl_warn("MAC address in DT not available yet\n");
	if (ret)
		return 0;

	ret = ath6kl_bmi_read_hi32(ar, hi_board_data, &address);
	if (!ret)
		ret = ath6kl_bmi_read(ar, address, buf, sizeof(buf));
	if (ret)
		return ret;

	for (i = 0; i < ETH_ALEN; i++)
		buf[AR6003_BOARD_CHECKSUM_OFFSET + i % 2] ^=
			buf[AR6003_MAC_ADDRESS_OFFSET + i] ^ mac[i];
	memcpy(buf + AR6003_MAC_ADDRESS_OFFSET, mac, ETH_ALEN);

	return ath6kl_bmi_write(ar, address, buf, sizeof(buf));
}
```

MAC は `of_get_mac_address()` で DT から取る。この関数は DT の `mac-address` プロパティだけでなく、nvmem のセルも見てくれる。SDIO の関数は、mmc ホストの子ノード `wifi@1` を書けば自動で DT のノードと結び付く。DT に MAC が無ければ何もしないので、モジュール自身の MAC のまま動く。

### U-Boot の環境変数を nvmem にする

MAC の出どころは U-Boot の環境変数。SPI フラッシュのダンプを見ると、`0xC000` から環境変数らしい `name=value` の並びがあった。先頭 4 バイトが CRC32 だとすると、どこまでを計算すれば合うかで大きさが分かる。

```python
crc = struct.unpack('<I', d[0xc000:0xc004])[0]
for size in (0x1000, 0x2000, 0x4000, 0x8000):
    print(hex(size), zlib.crc32(d[0xc004:0xc000 + size]) == crc)
# 0x2000 だけ True
```

8KiB と分かった。mainline には U-Boot の環境変数を nvmem のセルとして出す仕組み(`u-boot,env`)があって、`ethaddr` は MAC アドレスとして 6 バイトに変換してくれる。`wifiaddr` にも同じ変換をかけるよう、1 行足した。

```c
if (!strcmp(var, "ethaddr") || !strcmp(var, "wifiaddr")) {
	info.raw_len = strlen(value);
	info.bytes = ETH_ALEN;
	info.read_post_process = u_boot_env_read_post_process_ethaddr;
}
```

最初はパーティションのノードに直接 `compatible = "u-boot,env"` を付けた。すると nvmem のデバイスは確かにできたのに、ath6kl からは `-ENOENT` でセルが見つからない。調べると、同じパーティションに nvmem のデバイスが 2 つできていた。MTD が自分で作るものと、`u_boot_env` ドライバが作るもの。セルを探すときに DT のノードで照合すると、先に見つかる MTD の方(セルが無い方)に当たっていた。

mainline の今の推奨は、パーティションの下に `nvmem-layout` を置いて、MTD の nvmem にセルを載せる書き方だった。

```dts
partition@c000 {
	label = "u-boot-env";
	reg = <0x0000c000 0x00004000>;
	read-only;

	/* 8KiB env (CRC32 checked), then erased flash */
	nvmem-layout {
		compatible = "u-boot,env";
		env-size = <0x2000>;

		wifiaddr: wifiaddr {
			#nvmem-cell-cells = <1>;
		};
	};
};
```

```dts
&mmc {
	...
	#address-cells = <1>;
	#size-cells = <0>;

	/* AR6003, with the Wi-Fi MAC address the vendor firmware uses */
	wifi@1 {
		reg = <1>;
		nvmem-cells = <&wifiaddr 0>;
		nvmem-cell-names = "mac-address";
	};
};
```

これで MAC が D-Link の `C8:D3:A3:…` になり、DHCP で新しい IP をもらって、普通につながった。ちなみに手元のダンプとこの本体とで `wifiaddr` の値が違っていたので、MAC は本体ごとに振られているようだ。ダンプを取った本体と今いじっている本体が別物だった。

## 手を付けなかったもの

- **HDMI の音声**: DSM-232 の音声出力は HDMI だけ。音声は DSP ベースのブロック(`0xB1200000`)で、DSP のファームは GPL パッケージにあるものの、ホストとのやり取りはバイナリの `libcsaud.a` の中にしか無い。それに手元のモニターにはスピーカーが無く、音が出たかを確かめられない。スピーカー付きのテレビが手に入るまで保留。
- **I2C**: `0x80170000`(IRQ 2)と `0x80174000`(IRQ 3)に I2C らしきブロックがあるが、DesignWare ではない独自の作り。この基板で何がつながっているかもまだ分からない。U-Boot の `i2c probe` で何がぶら下がっているかを見てから考える。
- **SPI の DMA**: Linux から見える SPI フラッシュは 128KiB の read-only なので、要らない。
- **LAN**: そもそも LAN 端子が無い。

もう一つ気になっているのが、何時間か放置すると SSH にも ping にも応答しなくなること。kexec で試験していたときも、USB から普通に起動したときも起きた。UART のログを取っていなかったので、まだ原因は分からない。次はここから。
