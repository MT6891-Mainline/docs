# chopin 音频移植交接（2026-10-23，第二版）

## 目标与硬件链路

chopin 没有 3.5mm 耳机口，外放链路是：

```
AP AFE (0x11210000, IRQ SPI 202, I2S3 输出)
      └─> CS35L41 x2 (i2c6: 0x40 主, 0x42 副, reset=GPIO30, switch=GPIO27, irq=GPIO31)
              └─> 扬声器
MT6359 (PMIC codec) 仅用于麦克风/耳机检测，外放不经过它
```

## 关键发现

- vendor 的 `sound/soc/mediatek/mt6873/mt6873-reg.h` 与主线
  `sound/soc/mediatek/mt8192/mt8192-reg.h` **寄存器名和值完全一致**
  （2923 个 define，值 diff 为 0）→ MT6893 的 AFE 直接以主线 mt8192 驱动为底座；
- vendor AFE 节点是 `mt6885-afe-pcm@11210000`，基址 0x11210000、IRQ SPI 202，
  与 mt8192 相同；
- 扬声器功放是 **CS35L41**（vendor defconfig：`CONFIG_SND_SOC_CS35L41_I2C=y` +
  `CONFIG_SND_SOC_CS35L41_CHOPIN=y`），两颗在 i2c6 的 0x40/0x42；
- **audiosys 复位不需要做**：vendor 内核从不复位 AFE（vendor DTS 的 afe 节点
  没有 `resets`），LK 交过来时 AFE 就处在 power-on 状态；而且 MT6893 的
  TOPRGU 音频位在本板子上不可信（上一轮加 `&watchdog 17` 导致花屏）。
  现在驱动用 `devm_reset_control_get_optional_exclusive`，DT 不给 `resets`
  就跳过复位；
- **音频 MTCMOS 需要显式打开**：vendor 的 scpsys 把
  `SCP_SYS_AUDIO` 当 clock 用（`<&scpsys SCP_SYS_AUDIO>`），主线没有对应
  驱动，AFE 的寄存器访问会落在没上电的音频岛上。主线
  `drivers/pmdomain/mediatek/mt6893-pm-domains.h` 已有完整的 MT6893 数据
  （audio: `ctl_offs 0x354`, `sta_mask BIT(21)`），所以只要在 DT 里补一个
  SPM syscon + `mediatek,mt6893-power-controller`，只描述 audio 一个域。

## 已完成

### 上一轮（分支 `audio-mt6893` 已有）

| 提交 | 内容 |
| --- | --- |
| `b6f3d2c` | `clk-mt6893-afe.c`：MT6893 audsys 32 个门控 + `CLK_AUD_*` 绑定 |
| `b7f9f97` | i2c：接受 vendor 的 `mediatek,imp_iic_wrap_[cens]` 和 `mediatek,i2c`，`i2c-0..8` 全部出现 |
| `d892224` | CS35L41 DT：`reset-gpios`/中断、GPIO30 的 `gpio-hog`（output-high）、`/pinctrl` 加 `gpio-controller`、禁用 vendor `gpio@10005000` |

已验证（上一轮）：i2c6 电气正常（0x11 背光、0x60/0x61、0x40/0x42 都能扫到）、
`6-0040: Cirrus Logic CS35L41 (35a40), Revision: B2`、`gpio-30` 由 `spk-reset`
持有并输出高、`6-0042` 只是 IRQ 和 `mt-eint` 冲突。

### 本轮（2026-10-23 晚）

1. **AFE 复位改成可选**（`sound/soc/mediatek/mt8192/mt8192-afe-pcm.c`）：
   `devm_reset_control_get_optional_exclusive()`，DT 没有 `resets` 就不复位。
   上一轮的 `-ENOENT: could not get audiosys reset` 随之消失，也不再碰
   TOPRGU（避免花屏）。
2. **`mt6893-xiaomi-chopin-audio.dtsi` 重写**：
   - `afe` 节点补齐 **56 个时钟**（audsys 20 + infracfg 2 + topckgen 31 +
     apmixed 2 + clk26m），顺序对齐 `mt8192.dtsi`；
     `top_apll1_ck`/`top_apll2_ck` 直接指向 `&{/apmixed@1000c000}` 的
     `CLK_APMIXED_APLL1/2`（topckgen 的音频 mux 的 parent 名就是
     `apll1`/`apll2`，是同一个 clock 对象）；
   - 新增 `syscon@10006000` + `power-controller`（`mediatek,mt6893-power-controller`），
     只描述 `MT6893_POWER_DOMAIN_AUDIO`；`afe` 加
     `power-domains = <&spm MT6893_POWER_DOMAIN_AUDIO>`；
   - 新增 `sound` 节点：`mediatek,mt6893-mt6359`、`mediatek,platform = <&afe>`、
     `pinctrl-0 = <&{/pinctrl/aud_gpio_i2s3_on}>`（I2S3 三根引脚 pinmux 1）、
     `speaker-codecs { sound-dai = <&cs35l41@40>, <&cs35l41@42>; }`。
3. **CS35L41 DT 收尾**（`mt6893-xiaomi-chopin-i2c-spi.dtsi`）：
   - 两颗都加 `sound-name-prefix`（"Right"/"Left"），两个同名 codec 的
     kcontrol/widget 不再撞名，省掉硬编码 i2c 适配器号的 `codec_conf`；
   - 删掉 0x42 上 vendor 格式的四 cell 中断（pinctrl irq domain 只有两 cell，
     且与 mt-eint 冲突）和 vendor-only 的 reset/switch/irq-gpio 属性。
4. **机器驱动** `sound/soc/mediatek/mt8192/mt6893-mt6359.c`（新文件）：
   - 用 mtk-soundcard 公共框架（`mtk_soundcard_common_probe`），
     legacy 路径下从 `speaker-codecs` 的 phandle 取 codec，不写死 i2c 地址；
   - FE：`Playback_1` → AFE `DL1`（DPCM）；BE：`I2S3` → 两颗 CS35L41
     （`cs35l41-pcm`），`SND_SOC_DAIFMT_I2S | CBC_CFC`；
   - BE `hw_params` 照 mt8189：按 `rate*32` 设 codec SCLK、按 `rate*128`
     设 CPU MCLK、两颗 amp 分别取 slot 0/1；
   - 约束：44100/48000，1~2 声道（44100 会走 APLL1，48000 走 APLL2）；
     BE 用 `be_hw_params_fixup` 强制 **S16_LE**：SCLK=32 fs 正好是
     两个 16-bit slot，S24/S32 会让 I2S 跑 48/64 fs 而功放仍按 32 fs 收；
   - `sound/soc/mediatek/Kconfig` 新增 `SND_SOC_MT6893_MT6359`，
     `mt8192/Makefile` 加一行。
5. 编译通过：`Image`、`dtbs` 均无新增 warning；
   `out-audio/boot-stockdt.img` 已重新打包。

### 第一次上机结果（2026-10-23 19:5x）

**AFE 起来了**（上一轮的硬阻塞解除）：

```
mt8192-audio 11210000.syscon:audio-controller: no reserved memory found, pre-allocating buffers instead
mt8192-audio 11210000.syscon:audio-controller: mt8192_dai_pcm_register()
mt8192-audio 11210000.syscon:audio-controller: mtk_afe_combine_sub_dai(), num of dai 39
```

没有复位错误、没有 `devm_clk_get ... fail`、`mtk-pm-domains` 也没报
上电失败 → 音频电源域/时钟/可选复位这套是通的。

发现并修掉的问题：

- `cs35l41 6-0042: error -EINVAL: request_irq(0) ... Failed to request IRQ`
  → 上一轮把 0x42 的中断删掉后，mainline cs35l41 仍然无条件
  `devm_request_threaded_irq()`。0x42 因此 probe 失败，机器驱动的
  `snd_soc_of_get_dai_link_codecs()` 一直 `-EPROBE_DEFER`，声卡注册不了
  （日志里就停在 `mt6893_mt6359 sound: audio-routing not found: using legacy probe`，
  最后 `No soundcards found`）。
  修复（提交 `8616042`）：`cs35l41->irq > 0` 才申请中断，PM hook 同样加保护；
  功放 IRQ 只用于报错，没有它不影响放音。
- `mt6359-sound: failed to parse dts` 是预期内的：它的 DT 数据节点
  （PMIC 下的 `mt6359codec`/`audio-codec`）还没补，本轮外放不经过它，
  下一轮做麦克风时再加。

### 第二次上机（同一晚）：DPCM 连线 + 功放上电

1. 手动打开 AFE 连线矩阵后，`snd_pcm_open`/`hw_params` 全部通过：
   ```
   Rate set to 48000Hz ... period_size = 12000, buffer_size = 48000
   ```
   证明 I2S3 时钟、APLL2、codec sysclk、channel map 都是对的。
2. 但第一次 write 报 `-EIO`。`sound/core/pcm_lib.c: wait_for_avail()` 里的
   `playback write timeout (DMA or IRQ trouble?)` —— 即 start 之后 AFE 没有
   推进指针/产生 period 中断。真正原因在 dmesg 末尾：

   ```
   mt8192_afe_gpio_select(), error, gpio type 15 not prepared
   cs35l41 6-0042: Enable(1) failed: -110
   ASoC: PRE_PMU: Left Main AMP event failed: -110
   cs35l41 6-0040: Enable(1) failed: -110
   ASoC: PRE_PMU: Right Main AMP event failed: -110
   ```

   两颗 CS35L41 的 `GLOBAL_EN` 上电轮询 `PUP_DONE` 100 ms 超时（`-110`），
   DAPM 上电失败 → trigger 失败 → write 超时。所以现在剩下的是**功放上电**
   这一环，AFE/时钟/路由已经全部打通。

   修复/诊断（提交 `1adee27`、`acc263e`）：
   - `late_probe` 自动打开 `I2S3_CH1 DL1_CH1` / `I2S3_CH2 DL1_CH2`，不再需要
     amixer；
   - `cs35l41_global_enable()` 在 PUP 超时时打印
     `PWR_CTRL1 / IRQ1_STATUS1 / IRQ1_STATUS2 / IRQ1_STATUS3`
     （STATUS3 bit1 = PLL_LOCK），用来区分"没有 SCLK"还是"供电/boost"问题；
   - initramfs 里加了 `i2c-tools`（`i2ctransfer` 可以直接读功放寄存器）。

   `mt8192_afe_gpio_select type 15 not prepared` 是因为机器驱动没有调用
   `mt8192_afe_gpio_init()`；I2S3 的 pinmux 目前靠 sound 节点的
   `pinctrl-0` 生效。如果下一步确认 BCLK 没出来，就把 gpio_init 补上并把
   `aud_gpio_i2s3_on/off` 加到 sound 节点的 pinctrl-names 里。

### 第三次上机：功放 PUP 超时的寄存器解码

```
Enable(1) failed: -110 (PWR_CTRL1=0x1 STATUS1=0x400000 STATUS2=0x0 STATUS3=0x20000080)
```

| 寄存器 | 值 | 含义 |
| --- | --- | --- |
| `PWR_CTRL1` | `0x1` | `GLOBAL_EN` 写入成功，但没有回退 → 芯片确实收到使能 |
| `IRQ1_STATUS1` | `0x400000` | `PUP_DONE`(bit24) **未置位**；bit22 是 BST/AMP 类事件 |
| `IRQ1_STATUS3` | `0x20000080` | bit7 = **BST_DCM_UVP_ERR**（boost DCM 欠压），bit1 = **PLL_LOCK 未置位** |

电源侧已排除：`vaud18 / vio18 / vcn18 / vaux18 / vs2 / vusb` 全部 `enabled`，
VA 在、I2C 也在。剩下的唯一解释是**功放没收到 SCLK**（BCLK）：
PLL 不锁 → 芯片上电序列失败 → boost 报 DCM UVP。

对应的修复（提交 `f897abc`、`aabcbcb`）：

- 机器驱动调用 `mt8192_afe_gpio_init(card->dev)`——之前没调用，
  所以 I2S3_EN 事件里的 `mt8192_afe_gpio_request(..., I2S3)` 只是打警告，
  从来没有真正选过 I2S3 pinmux；
- sound 节点补上 `pinctrl-names`（`aud_gpio_i2s3_on/off`、`aud_clk_mosi_*`、
  `aud_dat_mosi/miso_*`），让 helper 能按名字找到状态；`default` 仍是
  I2S3 ON，保证 card probe 后就有 BCLK。

### 第四次上机：pinmux 已由 AFE helper 接管，但功放仍无 PLL

pinmux 检查（播放失败、DAPM 已下电之后）：

```
pin 32 (GPIO32): sound (GPIO UNCLAIMED) function func0 group GPIO32
pin 33 (GPIO33): sound ... func0 group GPIO33
pin 35 (GPIO35): sound ... func0 group GPIO35
```

owner 是 `sound`，说明 `mt8192_afe_gpio_init()` 已经生效（状态表里有
`aud_gpio_i2s3_on/off`）；但此刻显示 func0 是因为流失败后 DAPM 走了
POST_PMD，选中了 `aud_gpio_i2s3_off`，所以这个采样点看不到 ON 状态。

功放依旧是同样的 `PLL_LOCK=0 / BST_DCM_UVP_ERR`。为了确认 AFE 在功放
上电的 100ms 窗口里到底有没有开 I2S3，`mt6893-mt6359.c` 在 BE hw_params
后 60ms 打一行：

```
AFE dump: I2S_CON3=%#x DAC_CON0=%#x DL1_BASE=%#x DL1_CUR=%#x IRQ_MCU_EN=%#x
```

- `I2S_CON3` bit0 = I2S_EN（I2S3 端口是否真开）；
- `DAC_CON0` bit0 = AFE_ON，bit4 = DL1_ON；
- `DL1_BASE/CUR` = memif 的地址/当前指针。

### 第五次上机：AFE 侧寄存器正常，差异在 I2S 位宽/SCLK

AFE dump（功放 PUP 窗口内，hw_params 后 60ms）：

```
AFE dump: I2S_CON3=0xa09 DAC_CON0=0x1 DL1_BASE=0xd0240000 DL1_CUR=0x0 IRQ_MCU_EN=0xffffffff
```

- `I2S_CON3=0xa09`：I2S_EN=1、WLEN=16-bit、FMT=I2S、OUT_MODE=10(48k)；
- `DAC_CON0=0x1`：AFE_ON=1（DL1_ON 还没到 trigger，正常）；
- `DL1_BASE=0xd0240000`：DMA 地址合法。

所以 AFE 认为自己已经在跑 I2S3，但功放 PLL 仍然不锁。

**决定性证据来自 stock Android 的内核日志**（`n10p/console-ramoops`）：

```
I2S3: mt6885_i2s_hw_params_fixup(), fix format to 32bit
speaker_amp 6-0040: cs35l41_codec_set_sysclk: clk_id=0, src=0, freq=3072000
speaker_amp 6-0040: cs35l41_codec_set_sysclk: 0x2c04 <== 0x430
mtk_dai_i2s_config(), id 31, rate 48000, format 10
mtk_afe_fe_hw_params(), DL1 ... fmt 6
```

也就是原厂播放时：FE 是 S16，BE 固定 **S32_LE**，BCLK=64fs=**3.072MHz**，
功放 `PLL_CLK_CTRL=0x430`（REFCLK_FREQ=0x21=3.072MHz）。我们之前用 S16 +
1.536MHz，对应 `PLL_CLK_CTRL` 是另一个值，板子上从未这样跑过。

提交 `c7a95e5`：
- BE `be_hw_params_fixup` 改回 **S32_LE**；
- codec SCLK 改为 `rate * 64`（48k → 3.072MHz，与 0x430 完全一致）；
- CS35L41 的 GPIO 配置从 vendor 的 `cirrus,gpio-config2` 子节点摊平为
  `cirrus,gpio2-src-select`/`cirrus,gpio2-output-enable`（mainline 只认后者，
  之前功放中断输出一直是 HIZ）。

### 第六次上机：找到真正的缺口 —— APLL12 div3 门控是关的

```
AFE dump: I2S_CON3=0x1a0b   (HD_EN 也置上了)
topckgen dump: 0x320=0x770008df  0x328=0x30010707 (div3=48)
```

- `0x320` bit11=1：I2S3 的 mux 已经指向 `aud_2_sel`（APLL2）；
- `0x328` 的 div3=48：分频值已经被写过；
- **`0x320` bit3=1：APLL12 div3 的门控是关的**（mtk 的 gate 是
  `CLK_GATE_SET_TO_DISABLE`，1=关闭）。

也就是说 `mt8192_mck_enable()` 里 `clk_prepare_enable(apll12_div3)` 这一环
没有生效，BCLK 根本不存在 → 功放 PLL 不锁 → PUP 超时。

原因：`I2S3_MCLK_EN` 这个 supply 的 `mtk_afe_i2s_mclk_connect()` 回调
以 `i2s_priv->mclk_rate` 为条件，而它在 DAPM 电源检查时仍是 0（或是
dai_priv 索引问题），所以 MCLK_EN 从未上电、`mtk_mclk_en_event` 从未执行。

提交 `d61115f`：在 BE `hw_params` 里直接
`mt8192_mck_enable(afe, MT8192_I2S3_MCK, mclk_fs)`，`hw_free` 里对应
`mck_disable`，并打印 `sysclk_ret / mck_enable_ret`。下次上机 topckgen
的 0x320 bit3 应该变成 0。

### 第七次上机：修正上一轮的误判 + 交给驱动做复位

上一轮把 `0x320` bit3 当成 DIV3 的门控，那是 **mt8192 的布局**。MT6893
（`clk-mt6893-topckgen.c`）是：

| 时钟 | 门控 | 分频 |
| --- | --- | --- |
| `APLL12_CK_DIV3` | `0x320` **bit5** | `0x324[31:24]` |
| `APLL_I2S3_MCK_SEL` | — | `0x320` bit11 |

实测 `0x320=0x770008df` → bit5=0（**门控是开的**）、bit11=1（已选
`aud_2_sel`）。所以 `mt8192_mck_enable()` 一直是成功的
（`mck_enable_ret=0`），BCLK 时钟树并没有问题。之前"门控关着"的结论作废。

AFE 侧（`I2S_CON3` 的寄存器写序列）与厂商 `mt6873-dai-i2s.c` 逐位一致，
功放 PLL 寄存器也按原厂 `0x430` 写。剩下的差异只在**功放自身**：

- 板级树用 gpio-hog 把 GPIO30 拉高，codec 驱动的 probe **从未发过复位
  脉冲**（mainline 探测序列是 low 2ms → high 2ms → 等 OTP_BOOT_DONE）；
- 厂商只给 0x42 配了 GPIO2 中断输出，我们之前给两颗都配了 push-pull，
  共享 EINT 会打架。

提交 `0980289`：删掉 hog，两颗 CS35L41 都加
`reset-gpios = <&{/pinctrl} 30 GPIO_ACTIVE_HIGH>`，GPIO2 只在 0x42 上配。
dump 同时改成 MT6893 的寄存器位置（`0x324` div3），并打印功放
`PLL_CLK_CTRL`。

### 第八次上机：reset-gpios 尝试失败（gpiod 永远 defer），已回退

给两颗 CS35L41 加 `reset-gpios` 后，codec 探测变成：

```
cs35l41 6-0040: deferred probe pending: cs35l41: Failed to get reset GPIO
platform sound: deferred probe pending: mt6893_mt6359: get speaker codecs failed
Playback open error: -2,No such file or directory
```

`devm_gpiod_get_optional(..., "reset", GPIOD_OUT_LOW)` 返回 `-EPROBE_DEFER`
且一直不消失——这个 pinctrl 节点的 gpiochip 对消费者不可解析（不是
`-EBUSY` 的共享复位路径）。codec 因此不再 probe，声卡不注册。

提交 `1cbc306` 回退：GPIO30 重新用板级 hog 拉高，等单独查清 gpiochip
lookup 的问题（`/sys/kernel/debug/gpio`、`/sys/bus/gpio/devices/`）后再
考虑驱动复位。

### 第九次上机：defer 的真因是「共享 GPIO」框架

`cat /sys/kernel/debug/gpio`：

```
gpiochip0: 227 GPIOs, parent: platform/10005000.pinctrl, pinctrl_paris
 gpio-30  (                    |shared              ) out lo
```

gpiochip 本身是好的（`gpiochip0`，227 线）。问题在于**两颗 CS35L41 都在
DT 里把 GPIO30 写成 `reset-gpios`**，内核 `gpiolib-shared.c` 的共享 GPIO
机制因此接管了这条线（label 变成 `shared`，并给每个 consumer 建
auxiliary device），而 cs35l41 用的是普通的 `devm_gpiod_get_optional()`，
拿到的是 `-EPROBE_DEFER` 并且永远等不到。

提交 `19afce3`：只有 `cs35l41@40` 声明 `reset-gpios`（它 probe 时发出的
复位脉冲同时复位 0x42，而 0x42 在它之后 probe），板级 hog 删除。

### 第十次上机：数字链路全部正确，PUP_DONE 本身不可靠

```
topck: 0x320=0x770008df div3_gate=0 i2s3_sel=1 0x324=0x1f070707 div3=31 0x0e0=0xc030601
codec PLL_CLK_CTRL=0x430
```

- `div3_gate=0`（使能）、`i2s3_sel=1`（APLL2）、div3=31；
  这颗板的 APLL2 实际被配成 190.464MHz，190.464/31 = **6.144MHz** → mclk 正确；
- `PLL_CLK_CTRL=0x430` 与 stock Android 日志里的值**逐位相同**
  （REFCLK=0x21=3.072MHz、CLK_SEL=SCLK、PLL_EN=1）。

也就是说 AFE、时钟、功放 PLL 配置全部与量产一致，但 `PUP_DONE`
依旧不置位、`PLL_LOCK` 也不报。对照厂商驱动
（`n10p/.../cs35l41/cs35l41.c:1100`）可以确认：**厂商在
POST_PMU 里写完 `pup_patch` + `GLOBAL_EN` 后只 `usleep_range(1000)`，
从不检查 `PUP_DONE`**。这颗板的 PUP_DONE / PLL_LOCK 状态位不可靠，
而 mainline 把它当成致命错误，直接让 DAPM 上电失败、PCM 流被杀掉。

提交 `bb8c3b3`：INT_BOOST 的 PUP 轮询缩短到 5ms，超时只 `dev_warn`
并继续（保留状态转储），与厂商行为一致。

### 第十一次上机：写超时的真因——连线矩阵开关从来没打开过

PUP 不再致命之后，流能进到 trigger，但：

```
mt8192-audio ...: hw_ptr or hw_base = 0 err
AFE dump: DAC_CON0=0x11 (AFE_ON=1, DL1_ON=1) DL1_BASE=0xd0240000 DL1_CUR=0x0
Write error: -5
```

DL1 的 memif 已 enable、BASE 合法，但 **CUR 一直是 0**：没有消费者来取数据。
原因是我们自动打开矩阵开关的代码放在 `late_probe`，而
`snd_soc_card_late_probe()` 在 `snd_soc_dapm_new_widgets()` **之前**
（`soc-core.c:2156` vs `:2164`），此时 `I2S3_CH1 DL1_CH1` 这些 DAPM mixer
控件还不存在，`snd_soc_card_get_kcontrol()` 返回 NULL——所以从没真正打开
过（也解释了之前必须手动 `amixer sset`）。

提交 `1bfac58`：改成 late_probe 里 `schedule_delayed_work(1s)`，等控件创建
完再 put；dump 增加 `AFE_CONN0/CONN1`（DL1_CH1=bit5、DL1_CH2=bit6）。

### 第十二次上机：手动打开矩阵后 CUR 仍为 0 → 怀疑 APLL2 没跑

手动 `amixer sset` 打开 `I2S3_CH1 DL1_CH1` / `I2S3_CH2 DL1_CH2` 后，
`DL1_CUR` 仍然是 0、write 仍超时。memif 已 enable（`DAC_CON0=0x11`）、
BASE 正确、矩阵开关也开了，但 memif 没有被"拉动"——这说明**采样时钟本身
没有在跑**。mux/div3 的配置（`0x320` bit5=0、bit11=1、`0x324` div3=31）
只说明时钟树"选择"正确，APLL2 这颗 PLL 是否真的 enabled 还没验证过。

提交 `952bbbe`：`hw_params` 里显式 `clk_get`+`clk_prepare_enable(
"top_apll2_ck")`（`hw_free` 里释放），并在 dump 里打印
`APLL2: CON0/PCW/CON1`（CON0 bit0 = PLL enable）。

### 第十三次上机：APLL2 是开的，剩下 aud_i2s3_b 门控

```
APLL2 root enable: 0 (rate 196608000)
APLL2: CON0=0x181 en=1 PCW=0x4000000 CON1=0x80000001
AFE conn: CONN0=0x20 DL1_CH1=1 CONN1=0x40 DL1_CH2=1
```

APLL2 已 enabled（CON0 bit0=1，196.608MHz）；`0x324` 里 div3=31 是
mtk 的**一基**写法（寄存器值+1=32），实际 mclk=196.608/32=6.144MHz，
BCLK=3.072MHz，与功放 `PLL_CLK_CTRL=0x430` 完全对应。矩阵两位也都是 1。
但 `DL1_CUR` 仍然 0。

mt8192 AFE 驱动的时钟表里**没有 I2S 时钟门**（`aud_i2s3_b` 等），
也就是说 `AUDIO_TOP_CON1` bit6 这个 I2S3 模块门控从来没人开过；它若为 1
（mtk 的 audio 门是低有效，0=开），I2S3 模块就不会产生 BCLK/LRCK，
memif 也就永远不会被取数。

提交 `13139e1`：`hw_params` 里直接 `regmap_clear_bits(0x4, BIT(6))` 打开它，
并在 dump 里增加 `TOP0/TOP1/TOP2`、`AFE_DL1_CON0`、`AFE_IRQ_MCU_STATUS`。

### 第十四次上机：I2S 门控其实一直是开的，采样点太早

```
AFE cg: TOP0=0xb0f9403c TOP1=0x0 TOP2=0x0 i2s3_b=0 dl1con=0xa313020 irqst=0x0 irqcnt=0x2e0
```

`TOP1=0x0` → 所有 I2S 门（含 `aud_i2s3_b` bit6）本来就是 **0 = 使能**，
所以上一轮的“门没开”假设不成立。`dl1con=0xa313020`（DL1_MODE=0xa=48k）、
`irqst=0`。

但注意：period_size=12000 帧 @48k = **250ms**，而 dump 是在 hw_params 后
**60ms** 采样的——如果 `AFE_DL1_CUR` 只在 period 边界更新，60ms 时读到 0
是正常的，`irqst=0` 也正常。上一轮把“CUR=0”当成 memif 没跑，可能又是
采样时机误导。

提交 `d0206ff`：dump 延后到 hw_params 后 **400ms**（超过一个 period），
这样 `DL1_CUR` 是否推进就没有歧义。

### 第十五次上机：AFE 中断为 0，aud_afe 门控竟然是关的

```
318: 0 ... GICv3 234 Level asyr          ← 播放前后都是 0
AFE cg: TOP0=0xb0f9403c ... i2s3_b=0 irqst=0 irqcnt=0x2e0
```

- AFE 的 GIC 中断（`asyr`，SPI 234）在播放前后计数都是 **0**；
- 400ms（超过一个 period）后 `DL1_CUR` 仍是 0；
- `TOP0=0xb0f9403c` 的 **bit2 = 1**，即 `aud_afe` 门控是**关的**
  （mtk audio 门低有效，0=开）。

也就是说整个播放期间 **AFE 核心时钟（CLK_AUD_AFE）不在**：memif 没有时钟，
CUR 不动、IRQ 不产生、I2S 也不出 BCLK——所有现象一次解释完。AFE 的时钟本该
由 `mt8192_afe_enable_clock()` 在 runtime resume 时打开；说明流期间 AFE 设备
处于 runtime suspend（或时钟被放掉）。

提交 `285a1a1`：`hw_params` 里显式 `pm_runtime_resume_and_get(afe->dev)`，
`hw_free` 里释放；dump 增加 `pm=`（`pm_runtime_active`）。

### 第十六次上机：pm 字段被截断，改成短行 + 显式开 aud_afe_clk

`AFE cg` 那行太长，`pm=` 被串口截掉了；`TOP0` bit2 仍是 1
（`aud_afe` 门关）、`DL1_CUR=0`、`irqcnt` 只有 0x2e0 附近（不是
period_size 12000）。

提交 `4510f9b`：

- `hw_params` 里显式 `clk_get(afe->dev, "aud_afe_clk")` +
  `clk_prepare_enable()`，并打印 `AFE pm_get=.. aud_afe_clk_enable=..`；
- dump 拆成两行，短行里带 `pm=` 和 `afegate=`（TOP0 bit2），避免截断。

### 第十七次上机：真因——AFE 的时钟全部是 NULL！

```
AFE memif: ... pm=1 afegate=1
```

`pm=1`（AFE 设备是 runtime-active）但 `aud_afe` 门控仍然是关的。
结合 `mt8192-afe-clk.c` 的初始化代码就明白了：

```c
afe_priv->clk[i] = devm_clk_get(afe->dev, aud_clks[i]);
if (IS_ERR(afe_priv->clk[i])) {
	dev_warn(... "devm_clk_get %s fail, ret %ld" ...);
	afe_priv->clk[i] = NULL;      /* 包括 -EPROBE_DEFER！ */
}
```

而 `clk_prepare_enable(NULL)` 返回 0。所以只要 AFE 先于 audsys 时钟
provider 探测（拿到 `-517`），**整个 AFE 的 40+ 个时钟就全是 NULL**，
所有 `clk_prepare_enable()` 都"成功"，实际什么都没开：

- `aud_afe` 门控从未打开 → memif 无时钟（CUR=0、无中断）；
- `mt8192_mck_enable()` 变成空操作 → 之前看到的 `0x324` div3 / mux 位
  其实是 **bootloader 留下的**；
- APLL2 之所以是开的，是因为我们后来单独 `clk_get`+enable 过。

这解释了目前为止的**全部**现象，包括功放收不到 SCLK。

提交 `658aecb`：`mt8192_init_clock()` 遇到 `-EPROBE_DEFER` 直接返回，
让 AFE 等 audsys 时钟就绪后重新 probe。

### ✅ 成功：扬声器出声了

上一轮“强制清 `AUDIO_TOP_CON0` bit2”之后，`speaker-test` 正常跑完
（`Time per period = 5.0`，Left/Right 都播），**两颗 CS35L41 扬声器出声**。

根因链条（全部由同一个问题引起）：

1. AFE 的时钟寄存器（audsys `AUDIO_TOP_CON0/1`）和 AFE 自身寄存器
   **共用同一个 syscon regmap**；
2. AFE runtime **resume 时先 `mt8192_afe_enable_clock()`、后
   `regcache_cache_only(false)`**，此时时钟框架对 audsys 门控的写入落在
   cache 里、没进硬件 → `aud_afe` 等门控一直关着；
3. 于是：AFE 核心无时钟 → DL1 memif 不取数（CUR=0）、周期中断不产生、
   I2S3 不出 BCLK → CS35L41 的 PLL 永远不锁、`PUP_DONE` 不上报。

修复（提交 `12637f7` 验证、`4804cb7` 收尾）：

- `mt8192_afe_runtime_resume()` 里把 `regcache_cache_only(false)` 提到
  `mt8192_afe_enable_clock()` **之前**；
- 机器驱动 `hw_params` 里再显式清一次 `AUDIO_TOP_CON0` bit2（aud_afe）和
  `AUDIO_TOP_CON1` bit6（i2s3_b），并 `mt8192_mck_enable(I2S3, 128fs)`；
- 清理掉全部调试 dump（AFE/topckgen/apmixed/regmap 打印、60→400ms 采样）。

保留的其它修复：DPCM 矩阵开关（延迟 work）、`I2S3_HD_Mux`、S32_LE +
3.072MHz SCLK、CS35L41 单消费者 reset-gpios、GPIO2 中断输出、PUP_DONE 容错、
AFE 时钟 `-EPROBE_DEFER` 传递。

## 当前可用镜像

```
out-audio/boot-stockdt.img   md5 aa03e88f9eac34b4f9382b0ee848db17
（内核 7.2.8-g4804cb7d6e2f，HEAD 4804cb7）
```

**刷 `boot-stockdt.img`**（boot.img 里放原厂 DTB 给 LK，内核 `setup_arch()`
再用内嵌的我们自己的树替换，AFE/声卡都在内嵌树里）：

```bash
fastboot flash boot_a out-audio/boot-stockdt.img
fastboot reboot
```

## 上机要看的日志

USB 串口（`screen /dev/ttyACM0 115200`）或 expdb 里 grep：

```sh
dmesg | grep -iE "afe|audio|mt6893|cs35l41|genpd|sound|asoc"
```

重点：

1. `mtk-pm-domains`：`... power-controller` 是否 probe、audio 域是否上电
   （失败会是 `failed to power on domain` / `-ETIMEDOUT`）；
2. `11210000.syscon:audio-controller` 是否 probe 成功（不再有
   `could not get audiosys reset`）；时钟缺失只剩 `devm_clk_get ... fail` 警告；
3. `mtk-afe-pcm`/`mt6893_mt6359` 是否注册成功：
   `mt6893_mt6359 sound` 卡出现在 `/proc/asound/cards`；
4. 有卡后：

```sh
aplay -l
speaker-test -t sine -c 2 -D plughw:0,0
```

（本轮已把 Alpine 的 `alsa-utils` 装进 `port/rootfs-alpine`，initramfs 里
自带 `aplay` / `speaker-test` / `amixer`，不需要另带工具。）

## 下一步 / 已知风险

1. 如果 audio 域上电超时：把 `pd_ignore_unused` 保留、先去掉
   `power-domains` 再试（说明 SPM 序列和这代的差异），日志里看
   `scpsys` 的 print；
2. 如果卡注册失败：先看 `6-0042` 是否 probe 成功（少了任何一颗 codec，
   卡会一直 `-EPROBE_DEFER`）；必要时先把 BE 的 `speaker-codecs` 减到
   0x40 一颗验证链路；
3. 扬声器有卡但无声：查 I2S3 pinmux（`/sys/kernel/debug/pinctrl/.../pinmux-pins`
   里 GPIO32/33/35 是否 `I2S3`）、APLL2 是否 enable、amp 的 `VA/VP`
   供电（现在走 dummy regulator，vendor 也不控制这两路，靠 PMIC 默认）；
4. **麦克风**（下一轮）：MT6359 ADDA 链路还没接进声卡，需要照
   `mt8192-mt6359-rt1015-rt5682.c` 加 primary codec BE（`ADDA` ↔
   `mt6359-snd-codec-aif1`）+ MTKAIF 协议/校准；MT6359 的 codec 平台设备
   （`mt6359-sound`）由 MFD 自动创建，已经就绪；
5. 若 CS35L41 需要固件（DSP 保护模式）才出声，把
   `cs35l41-dsp1-spk-prot.wmfw`/`.bin` 放进 initramfs 的 `/lib/firmware`
   （`mkinitramfs.sh` 的 `GPU_DRIVER` 同类机制）；
6. 修好外放后再补 `headset`/`accdet`（无 3.5mm 口，优先级低）。

## 配置要点

声音子系统已改成内建（`SOUND=y`、`SND=y`、`SND_SOC=y`、
`SND_SOC_CS35L41{,_I2C,_LIBRARY}=y`、`SND_SOC_MT8192=y`、
`SND_SOC_MT6893_MT6359=y`）——之前是 `=m`，`.ko` 没进设备导致
codec 根本没 probe。

## 附：扬声器最大音量的差异（未改，按用户要求保留）

CS35L41 的模拟增益寄存器 `AMP_GAIN_CTRL`（0x6C04）：

| 来源 | 值 | 效果 |
| --- | --- | --- |
| mainline 默认（当前移植） | `0x00000000` | PCM/PDM 增益 0 dB |
| 原厂 chopin tune | `0x00000273` | PCM/PDM 增益 19 ≈ +19.5 dB |

所以当前最大音量大约只有 Android 的七成左右。ALSA 已暴露
`Left/Right Analog PCM Volume`（numid 232/214，最大 20 级），手动拉满即可：

```sh
amixer -c 0 sset 'Left Analog PCM Volume' 100%
amixer -c 0 sset 'Right Analog PCM Volume' 100%
# 想持久化再 alsactl store
```

若以后要开机即用原厂增益，在 `mt6893-mt6359.c` 的 I2S3 后端 `.init` 里给两个
codec 各写一次 `CS35L41_AMP_GAIN_CTRL = 0x273` 即可。用户当前选择“够用，先不改”。
