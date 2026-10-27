# STM32 精讲 02：时钟树 RCC 与 GPIO 八种模式

> 第 1 周主线之二，基于 **STM32F103（Cortex-M3）**。本周门禁是"白板画出时钟树 + 说清 HSE→PLL→SYSCLK"，所以第 1~7 节要用**默写**的方式过，不要只读。
>
> 配套图：`../STM32F1_clock_tree.png`（全时钟树）、`../STM32F1_clock_tree_myconfig.png`（72MHz 路径高亮）。对照图读效果最好。

---

## 1. 五个时钟源（先分清"谁给谁"）

| 时钟源 | 频率 | 精度 | 主要用途 | 是否可关 |
|---|---|---|---|---|
| **HSI** 内部高速 RC | 8 MHz | ±1%（25°C），全温区差 | 备用时钟、HSE 起振失败时兜底、PLL 备用源（HSI/2=4MHz） | 可关（关掉就没了） |
| **HSE** 外部高速晶振 | 通常 8 MHz（4~16MHz） | 高（±20ppm 级） | **主时钟源**，经 PLL 倍频到 72MHz | 可关 |
| **LSI** 内部低速 RC | ≈40 kHz | 很糙 | **独立看门狗 IWDG**、RTC 备用 | 可关（IWDG 使能后自动） |
| **LSE** 外部低速晶振 | 32.768 kHz | 高 | RTC 走时（分频得到 1Hz） | 可关 |
| **PLL** 锁相环 | 输入×倍频 | 跟随源 | HSE×9=72MHz 或 (HSI/2)×16=64MHz | 可关 |

**关键概念**：
- 复位后默认跑 **HSI 8MHz**。也就是说"上电点灯但延时不符"是最常见的第一个 bug。
- PLL 源只有两个选择：`HSE` 或 `HSI/2`（由 `RCC_CFGR.PLLSRC` 选）。**没有"直接 HSE×9 之外"的选项**，也没有小数倍频。
- 晶振不起振 → `HSERDY` 永远不置位 → `while(!(RCC->CR & (1<<17)));` 死等。**这就是最常见的"程序卡在启动阶段"。**

---

## 2. 时钟树主线（HSE → PLL → SYSCLK → 总线）

```
HSI 8MHz ──┬─→ [SW] ────────────────┐
           └─(/2)→ 4MHz ──┐         │
                          ├─[PLLSRC]─→ PLL ×(2..16) ──┐
HSE 8MHz ──┬──────────────┘                          │
           │                                          │
           └─(/1 或 /2 via PLLXTPRE)──────────────────┤
                                                      │
                                     [SW = PLL] ──────┴──→ SYSCLK 72MHz
                                                              │
                        ┌─────────────────────────────────────┤
                        │                                     │
              [HPRE: AHB 预分频]                        [CSS 时钟安全系统]
                   /1,/2,/4.../512                          │
                        │                              (HSE 失效切 HSI)
                        ↓
                   HCLK 72MHz ───→ Cortex-M3 内核、SysTick、DMA、SRAM、FLASH
                        │
        ┌───────────────┴─────────────────┐
        │                                 │
 [PPRE1: APB1 预分频]            [PPRE2: APB2 预分频]
   /1,/2,/4,/8,/16                 /1,/2,/4,/8,/16
        │                                 │
   PCLK1 36MHz(MAX!)                PCLK2 72MHz(MAX)
        │                                 │
  TIM2/3/4/5/6/7, USART2/3,        GPIOA~E, EXTI, AFIO,
  SPI2, I2C1/2, USB, CAN,          USART1, SPI1, TIM1/8,
  IWDG, WWDG, PWR, BKP             ADC1/2
        │
   [USBPRE] → PLL/1.5 = 48MHz (USB)
        │
   [ADCPRE] → PCLK2/6 = 12MHz (ADC 时钟, MAX 14MHz)
```

### 背这张表（默写时按它写）

| 节点 | 值 | 由谁决定 |
|---|---|---|
| SYSCLK | **72 MHz** | `SW` + PLL 倍频 |
| HCLK (AHB) | **72 MHz** | `HPRE` = /1 |
| PCLK1 (APB1) | **36 MHz** | `PPRE1` = /2（**上限 36MHz**） |
| PCLK2 (APB2) | **72 MHz** | `PPRE2` = /1 |
| USB 时钟 | **48 MHz** | `USBPRE` = PLL/1.5 |
| ADC 时钟 | **12 MHz** | `ADCPRE` = PCLK2/6（上限 14MHz） |
| SysTick | HCLK 或 HCLK/8 | `SysTick->CTRL` 的 CLKSOURCE 位（HAL 默认用 HCLK） |
| FLASH 等待周期 | **2 WS** | 48~72MHz 必须 2WS；需开预取 |

**为什么 APB1 只能 36MHz**：APB1 挂的是低速外设（USART2/3、I2C、SPI2、CAN、USB），芯片手册规定其时钟上限 36MHz。**把 APB1 也配成 /1（72MHz）是最典型的超频错误**——可能"能跑"，但 I2C/USART 会随机出错。

**注意一个反直觉点**：当 APB 预分频系数 ≠ 1 时，该总线上的**定时器时钟 = PCLK×2**。所以 TIM2（挂 APB1）实际计数时钟是 72MHz 而不是 36MHz。第 6 周配 PWM 频率时必须用对这个数，否则频率差一倍。

---

## 3. 关键寄存器（只看这 4 个就够配时钟）

### RCC_CR（时钟控制寄存器）

| 位 | 名称 | 作用 |
|---|---|---|
| 0 / 1 | HSION / HSIRDY | 开 HSI / HSI 稳定标志（只读） |
| 16 / 17 | HSEON / HSERDY | 开 HSE / HSE 稳定标志（只读） |
| 18 | HSEBYP | 旁路（用外部有源晶振时置 1，普通无源晶振必须 0） |
| 19 | CSSON | 时钟安全系统：HSE 挂了自动切 HSI 并产生中断 |
| 24 / 25 | PLLON / PLLRDY | 开 PLL / PLL 锁定标志（只读） |

> **PLLON 只能在 PLL 关闭时修改 PLL 配置**。所以顺序永远是：**先配 PLL（CFGR）→ 再开 PLL（CR）**。

### RCC_CFGR（时钟配置寄存器）—— 最核心

| 位 | 名称 | 取值 |
|---|---|---|
| 1:0 | SW | 00=HSI, 01=HSE, 10=PLL |
| 3:2 | SWS（只读） | 实际生效的源，**要轮询它确认切换成功** |
| 7:4 | HPRE | AHB 分频：0xxx=/1, 1000=/2, 1001=/4, 1010=/8, 1011=/16, 1100=/64, 1101=/128, 1110=/256, 1111=/512 |
| 10:8 | PPRE1 | APB1：0xx=/1, 100=/2, 101=/4, 110=/8, 111=/16 |
| 13:11 | PPRE2 | APB2：同上编码 |
| 15:14 | ADCPRE | 00=/2, 01=/4, 10=/6, 11=/8 |
| 16 | PLLSRC | 0=HSI/2, 1=HSE |
| 17 | PLLXTPRE | HSE 分频：0=/1, 1=/2 |
| 21:18 | PLLMUL | 0000=×2 … 0111=×9 … 1110=×16, 1111=×16 |
| 22 | USBPRE | 0=PLL/1.5, 1=PLL/2.5 |
| 26:24 | MCO | 时钟输出：PA8 输出 SYSCLK/HSI/HSE/PLL/2 |

**×9 的编码是 `0111`**（不是 9 的二进制 1001！这是编码表不是数字）。默写时最常错的就是这个。

### RCC_APB2ENR（外设时钟使能，**点灯第一坑**）

| 位 | 外设 | 宏 |
|---|---|---|
| 0 | AFIO | `RCC_APB2ENR_AFIOEN` |
| 2/3/4/5/6 | GPIOA/B/C/D/E | `IOPAEN` / `IOPBEN` / `IOPCEN` / `IOPDEN` / `IOPEEN` |
| 9 | ADC1 | `ADC1EN` |
| 12 | SPI1 | `SPI1EN` |
| 14 | USART1 | `USART1EN` |

> **时钟门控**：STM32 每个外设都有独立的时钟开关，复位后默认全关。**不开时钟 = 寄存器写不进去、IDR 恒 0、ODR 无效。** 现象：代码完全正确但灯不亮。这是新手第 1 大坑。

### FLASH_ACR（访问控制）

```c
FLASH->ACR |= FLASH_ACR_PRFTBE | FLASH_ACR_LATENCY_2;   /* 预取 + 2 等待周期 */
```

**等待周期对照**（CPU 72MHz 而 Flash 最快 24MHz，必须插入等待）：

| SYSCLK | 等待周期 |
|---|---|
| 0~24 MHz | 0 WS |
| 24~48 MHz | 1 WS |
| 48~72 MHz | **2 WS** |

不设等待周期 → 高主频下取指出错、跑飞或随机 HardFault。

---

## 4. 寄存器版 72MHz 配置（8 步，附"为什么这个顺序"）

```c
/* ============ 把 SYSCLK 从复位默认的 HSI 8MHz 切到 HSE 8MHz × 9 = 72MHz ============ */
static void RCC_Config_72MHz(void)
{
    /* 1. 开 HSE ------------------------------------------------------------ */
    RCC->CR |= RCC_CR_HSEON;                       /* HSEON = bit16 */
    while (!(RCC->CR & RCC_CR_HSERDY)) { }         /* 死等 HSERDY = bit17 */
    /*    ↑ 卡在这里 = 晶振没起振：查晶振/负载电容/焊接，别怀疑代码 */

    /* 2. Flash：开预取 + 2 个等待周期（高主频的保命设置） ------------------ */
    FLASH->ACR |= FLASH_ACR_PRFTBE | FLASH_ACR_LATENCY_2;

    /* 3. 先把总线分频设好，再去切主频 ------------------------------------- */
    RCC->CFGR &= ~(RCC_CFGR_HPRE | RCC_CFGR_PPRE1 | RCC_CFGR_PPRE2);
    RCC->CFGR |=  RCC_CFGR_HPRE_DIV1                /* AHB  = SYSCLK/1 = 72M */
               |  RCC_CFGR_PPRE1_DIV2               /* APB1 = HCLK/2   = 36M */
               |  RCC_CFGR_PPRE2_DIV1;              /* APB2 = HCLK/1   = 72M */
    /*    ↑ 必须在切 PLL 之前！否则中间会有一瞬间 APB1 跑 72MHz */

    /* 4. 配 PLL：源 = HSE，倍频 = ×9 -------------------------------------- */
    RCC->CFGR &= ~(RCC_CFGR_PLLSRC | RCC_CFGR_PLLXTPRE | RCC_CFGR_PLLMULL);
    RCC->CFGR |=  RCC_CFGR_PLLSRC_HSE               /* PLLSRC = 1 */
               |  RCC_CFGR_PLLXTPRE_HSE_Div1        /* 不分频 */
               |  RCC_CFGR_PLLMULL9;                /* ×9 ← 编码 0111 */
    /*    ↑ 此时 PLL 仍是关的，所以能改配置 */

    /* 5. 开 PLL，等锁定 --------------------------------------------------- */
    RCC->CR |= RCC_CR_PLLON;
    while (!(RCC->CR & RCC_CR_PLLRDY)) { }

    /* 6. 切 SYSCLK = PLL --------------------------------------------------- */
    RCC->CFGR &= ~RCC_CFGR_SW;
    RCC->CFGR |=  RCC_CFGR_SW_PLL;                  /* SW = 10 */

    /* 7. 轮询 SWS 确认真的切过去了（硬件可能拒绝） ------------------------- */
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL) { }

    /* 8.（可选）HSE 失效自动切 HSI，防止晶振掉了直接死机 ------------------- */
    RCC->CR |= RCC_CR_CSSON;

    /* 现在：SYSCLK = 72MHz，SystemCoreClock 变量也要同步更新 */
    SystemCoreClockUpdate();
}
```

**要能解释的三件事**

1. **为什么先设分频再切 PLL**：避免超频窗口。虽然只持续几微秒，但规范做法如此，也是 ST 官方 `SetSysClockTo72()` 的做法。
2. **为什么 `while` 等待是必须的**：HSERDY / PLLRDY 是硬件置位，晶振起振要 ms 级，不等就配 PLL，实际是用未稳定的时钟。
3. **为什么 `SWS` 要轮询**：`SW` 是"我要什么"，`SWS` 是"实际是什么"。硬件切换失败（目标源没就绪）时 `SW` 会保持，`SWS` 不会跟着变。

---

## 5. HAL / CubeMX 视角 + 怎么验证时钟真的对了

### CubeMX 操作路径（今天就做一遍）

1. `New Project` → 选芯片 `STM32F103C8`（或你的具体型号）→ `Start Project`
2. `Pinout & Configuration` → `System Core` → **RCC** → `High Speed Clock (HSE)` 选 **Crystal/Ceramic Resonator**
3. `Clock Configuration` 页：把 `Input frequency` 填 **8**，`HSE` 路径选中，`PLL Source Mux` 选 HSE，`PLLMul` 选 **×9**，`System Clock Mux` 选 **PLLCLK**，确认 `APB1 Prescaler = /2`、`APB2 = /1`、`ADC Prescaler = /6`
4. 右上角会实时显示 `HCLK 72MHz / APB1 36MHz / APB2 72MHz` —— **把这个界面截图存进笔记**，它就是时钟树的答案
5. `Project Manager` → Toolchain 选 `MDK-ARM V5`，生成代码

### 生成的 `SystemClock_Config()` 逐行读

```c
void SystemClock_Config(void)
{
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    /* 1) 开 HSE + 配 PLL */
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState       = RCC_HSE_ON;          /* 对应 CR.HSEON */
    RCC_OscInitStruct.HSEPredivValue = RCC_HSE_PREDIV_DIV1; /* 对应 PLLXTPRE */
    RCC_OscInitStruct.HSIState       = RCC_HSI_ON;
    RCC_OscInitStruct.PLL.PLLState   = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource  = RCC_PLLSOURCE_HSE;   /* PLLSRC = HSE */
    RCC_OscInitStruct.PLL.PLLMUL     = RCC_PLL_MUL9;        /* ×9 */
    if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) { Error_Handler(); }

    /* 2) 设总线分频 + 切 SYSCLK，FLASH_LATENCY_2 就是那 2 个等待周期 */
    RCC_ClkInitStruct.ClockType      = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK
                                     | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource   = RCC_SYSCLKSOURCE_PLLCLK;  /* SW = PLL */
    RCC_ClkInitStruct.AHBCLKDivider  = RCC_SYSCLK_DIV1;          /* /1  */
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;            /* /2  */
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;            /* /1  */
    if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_2) != HAL_OK) { Error_Handler(); }
}
```

**读完对照第 4 节的 8 步**，你会发现 HAL 只是把"置位 + 等待"封装成了带超时和错误返回的函数。**能看懂这一层，面试问"时钟怎么配的"就不会只会背 CubeMX。**

### 三种验证手段（从弱到强）

| 手段 | 做法 | 能证明什么 |
|---|---|---|
| ① **延时闪烁自检**（无仪器时最实用） | `HAL_Delay(500)` 闪灯，秒表量 10 个周期 | 若慢了约 9 倍，说明其实还在跑 HSI 8MHz。**时钟配错最典型的症状** |
| ② **MCO 引脚输出** | `RCC->CFGR |= RCC_CFGR_MCO_SYSCLK;`（PA8 配置为复用推挽） | 示波器/逻辑分析仪直接读出频率，**最硬的证据** |
| ③ **读寄存器回显** | 串口打印 `RCC->CFGR` 的 `SWS` 位 + `SystemCoreClock` | 确认实际切换成功、软件变量与硬件一致 |

```c
/* 手段③的最小实现 */
printf("SWS=%lu SystemCoreClock=%lu\r\n",
       (RCC->CFGR >> 2) & 0x3, SystemCoreClock);   /* 期望 SWS=2, 72000000 */
```

> 逻辑分析仪（几十元，8 通道）是本计划最早该买的工具：**第 5 周验串口、第 7 周解 I2C、这周看 MCO 都靠它**，比示波器便宜且够用。趁本周下单。

---

## 6. 超频红线与常见错误（默写时一并写上）

| 项 | 红线 | 越界后果 |
|---|---|---|
| SYSCLK | ≤ 72 MHz | 随机死机 / 发热 / HardFault |
| HCLK | ≤ 72 MHz | 同上 |
| PCLK1 | **≤ 36 MHz** | I2C/USART 波形错乱，CAN 通信失败 |
| PCLK2 | ≤ 72 MHz | 高速外设异常 |
| ADC 时钟 | **≤ 14 MHz** | ADC 采样值偏差、噪声大 |
| FLASH 等待周期 | 48~72MHz 必须 2WS | 取指错误、跑飞 |

**五个高频错误**

1. 忘记 `SystemInit()` 里 `SystemCoreClockUpdate()` 后 `SystemCoreClock` 没更新 → `HAL_Delay` 时间全错。
2. CubeMX 里 HSE 选了 `Crystal/Ceramic` 但板子用的是**有源晶振**（应选 `BYPASS Clock Source`）→ 死等 HSERDY。
3. `PLLMUL` 写成了 `9`（十进制）而不是编码 `0111` → 实际倍频随机，主频诡异。
4. 只开 GPIO 时钟忘了开 AFIO（用 EXTI 重映射时）→ 中断不响应（第 4 周踩）。
5. 改了时钟但没改 FLASH 等待周期 → 72MHz 下偶发跑飞，难查。

---

## 7. 白板默写模板（第 7 周之前每周默一次）

纸上按这个顺序画，画完对照 `../STM32F1_clock_tree.png` 订正：

```
① 左边画 5 个源：HSI 8M / HSE 8M / LSI 40k / LSE 32.768k / (PLL 由 HSE 或 HSI/2 供)
② HSE 8M ──→ PLL ×9 ──→ 72M ──SW──→ SYSCLK
    HSI 8M ──/2→ 4M ─────↗（PLLSRC 选择）
③ SYSCLK ──HPRE /1──→ HCLK 72M ──┬─ 内核/SysTick/DMA/SRAM/Flash
                                  ├─PPRE1 /2──→ PCLK1 36M ── TIM2-7,USART2/3,SPI2,I2C,CAN,USB
                                  └─PPRE2 /1──→ PCLK2 72M ── GPIO,EXTI,USART1,SPI1,TIM1/8,ADC
④ PLL/1.5 ──→ USB 48M ；  PCLK2/6 ──→ ADC 12M
⑤ 标注：Flash 2WS + 预取 ； 定时器时钟 = PCLK×2（分频≠1 时）
```

**60 秒口述稿（照读三遍，再合上纸自己说）**

> "F103 复位后默认用内部 HSI 8MHz。为了跑 72MHz，我先打开外部 8MHz 晶振 HSE 并等 HSERDY，同时把 Flash 设成 2 个等待周期加预取。然后把总线分频先设好：AHB 不分频、APB1 二分频、APB2 不分频，因为 APB1 上限是 36MHz。接着配置 PLL：源选 HSE，倍频 ×9，得到 72MHz，开 PLL 等 PLLRDY，最后把 SW 切到 PLL 并轮询 SWS 确认。于是 SYSCLK 72MHz、HCLK 72MHz、PCLK1 36MHz、PCLK2 72MHz，USB 由 PLL/1.5 得到 48MHz，ADC 由 PCLK2/6 得到 12MHz。"

---

## 8. GPIO 八种模式与 CNF/MODE 编码表（本周必须默写）

F103 每个 GPIO 引脚用 **4 个位**配置，在 `GPIOx_CRL`（PIN0~7）或 `GPIOx_CRH`（PIN8~15）里。**低 2 位是 MODE，高 2 位是 CNF**。

```
GPIOB->CRL 的位布局（32 位，每 4 位管一个引脚）：

 bit31..28 | 27..24 | 23..20 | 19..16 | 15..12 | 11..8 | 7..4 | 3..0
   PB7     |  PB6   |  PB5   |  PB4   |  PB3   |  PB2  |  PB1 | PB0
 CNF7 MODE7|   ...                                                    
```

### MODE[1:0] —— 先定"输入还是输出、多快"

| MODE | 含义 |
|---|---|
| `00` | **输入模式**（复位默认值） |
| `01` | 输出模式，最大 10 MHz |
| `10` | 输出模式，最大 2 MHz |
| `11` | 输出模式，最大 50 MHz |

> 关键：**MODE=00 才叫输入**。CNF 的含义随 MODE 变化，同一组 CNF 在输入/输出下意思完全不同。

### CNF[1:0] —— 再看"哪种输入/哪种输出"

**当 MODE = 00（输入模式）时：**

| CNF | 模式 | HAL 名称 | 特点 |
|---|---|---|---|
| `00` | **模拟输入** | `GPIO_MODE_ANALOG` | 断开施密特触发器，直连 ADC/DAC |
| `01` | **浮空输入** | `GPIO_MODE_INPUT` | 高阻，电平完全由外部决定（复位默认） |
| `10` | **上拉/下拉输入** | `GPIO_MODE_INPUT` + `GPIO_PULLUP/PULLDOWN` | 内部 40kΩ 上下拉，**由 ODR 位决定：ODR=1 上拉，ODR=0 下拉** |
| `11` | 保留 | — | 不要用 |

**当 MODE = 01/10/11（输出模式）时：**

| CNF | 模式 | HAL 名称 | 特点 |
|---|---|---|---|
| `00` | **通用推挽输出** | `GPIO_MODE_OUTPUT_PP` | 强驱动高/低，能拉电流 |
| `01` | **通用开漏输出** | `GPIO_MODE_OUTPUT_OD` | 只能拉低；高电平靠外部上拉 |
| `10` | **复用推挽输出** | `GPIO_MODE_AF_PP` | 引脚交给片上外设（UART TX、SPI、PWM） |
| `11` | **复用开漏输出** | `GPIO_MODE_AF_OD` | 外设接管 + 开漏（I2C 的 SCL/SDA 就是它） |

### 八种模式速查（考试式）

| # | 模式 | CNF | MODE | 4 位值 | 典型用途 |
|---|---|---|---|---|---|
| 1 | 输入浮空 | 01 | 00 | `0b0100` = 0x4 | 外部已有上拉的按键、USART RX、复用输入 |
| 2 | 输入上拉 | 10 | 00 | `0b1000` = 0x8（ODR=1） | 按键接 GND（按下为 0） |
| 3 | 输入下拉 | 10 | 00 | `0b1000` = 0x8（ODR=0） | 按键接 VCC（按下为 1） |
| 4 | 模拟输入 | 00 | 00 | `0b0000` = 0x0 | ADC 采集、低功耗 |
| 5 | 推挽输出 | 00 | 11 | `0b0011` = 0x3 | LED、继电器、普通输出 |
| 6 | 开漏输出 | 01 | 11 | `0b0111` = 0x7 | 电平转换、线与、共享总线 |
| 7 | 复用推挽 | 10 | 11 | `0b1011` = 0xB | UART TX、PWM 输出、SPI SCK/MOSI |
| 8 | 复用开漏 | 11 | 11 | `0b1111` = 0xF | I2C SCL/SDA（硬件 I2C 时） |

> 2 和 3 的 4 位值**完全相同**，区别只在 `ODR` 位——这就是上拉/下拉的"隐藏开关"，也是最容易漏的一步。

---

## 9. 每个模式"什么时候用"（比背表更重要）

### 推挽 vs 开漏：本质区别

```
推挽输出（Push-Pull）：      开漏输出（Open-Drain）：
  PMOS 拉高 ┐                 上管永远关闭 ┐
  NMOS 拉低 ┘  两个管子轮流     只有 NMOS   ┘  只能拉低
  → 高低电平都由芯片提供        → 输出低：NMOS 导通，强下拉
                              → 输出高：管子关断 = 高阻，电平靠【外部上拉电阻】
        ┌── VCC                      ┌── VCC
        │                            │
       [ ] 上管                      [R] 外部上拉（必须有！）
        ├──→ 输出                     ├──→ 输出
       [ ] 下管                     [NMOS]
        │                            │
       GND                          GND
```

| 场景 | 选谁 | 原因 |
|---|---|---|
| 点 LED、驱动普通负载 | **推挽** | 需要主动输出高电平驱动能力 |
| 两个输出引脚"线与"（任一拉低即低） | **开漏** | 推挽直接对接会短路烧管 |
| **3.3V 芯片驱动 5V 器件** | **开漏 + 上拉到 5V** | 电平转换的标准做法 |
| I2C 总线（SDA/SCL） | **开漏**（+ 上拉 4.7kΩ） | 多主多从共享总线，防止冲突 |
| 与外部已有上拉的器件通信 | 开漏或不输出，别用推挽硬拉 | 避免"对拉" |

> **开漏实验（周六必做）**：把 LED 那一路改成开漏输出，**把外部上拉电阻去掉**。现象：低电平能亮（或灭，取决于接线），高电平时引脚处于高阻，LED 完全没反应。**能解释这个现象，就证明你真懂推挽/开漏。** 再加一个 4.7kΩ~10kΩ 上拉到 3.3V，现象恢复。

### 浮空 vs 上拉/下拉

- **浮空输入**：想读什么电平完全看外部。**悬空时会因感应电荷随机跳变**，表现为按键"乱触发"、功耗偏高。
- **上拉/下拉输入**：芯片内部约 40kΩ 电阻，开关不接时可读到确定的默认电平。**按键接 GND 用上拉**（松开=1，按下=0），**按键接 VCC 用下拉**（松开=0，按下=1）。
- **模拟输入**：数字部分断开，引脚直接进 ADC。**配了 ADC 却忘了改成模拟输入**，是第 6 周 ADC 采集不准的常见原因。

---

## 10. 寄存器版完整代码：点灯 + 按键（周六主线）

> ⚠️ 先把你的板子原理图找出来，确认三件事：**LED 在第几号引脚、高电平亮还是低电平亮、按键接 GND 还是 VCC、有没有外部上下拉**。看不懂原理图就先抄本节的引脚号做实验，但**必须去对照一次原理图**——这是本周的隐藏目标（第 11 周要能独立看懂任意开发板原理图）。

以 `D:\keilproject\Fire-F103` 那块野火板为例（**请以自己原理图为准**），下面假设 LED 在 PB0（高电平亮）、按键在 PB1（接 GND，按下为低）：

```c
#include "stm32f10x.h"
#include <stdint.h>

#define LED_PIN    0u          /* PB0 */
#define KEY_PIN    1u          /* PB1 */

static void delay_ms(uint32_t ms)      /* 用 SysTick 做粗延时，不依赖 HAL */
{
    /* 72MHz 下，SysTick 重装载 9000 → 每 1ms 中断 8000 次……这里用简单循环近似 */
    volatile uint32_t i;
    while (ms--) {
        for (i = 0; i < 7200u; i++) { __NOP(); }   /* ≈1ms @72MHz，仅供演示 */
    }
}

int main(void)
{
    /* ---- 1. 开时钟（AFIO 这里不用，所以只开 GPIOB） ---- */
    RCC->APB2ENR |= RCC_APB2ENR_IOPBEN;

    /* ---- 2. 配 PB0 为推挽输出 50MHz：CNF=00 MODE=11 → 0b0011 ---- */
    GPIOB->CRL &= ~(0xFu << (LED_PIN * 4));          /* 先清零这 4 位 */
    GPIOB->CRL |=  (0x3u << (LED_PIN * 4));          /* 0b0011 推挽输出 50MHz */
    GPIOB->BSRR = (1u << LED_PIN);                   /* 初始灭（高电平亮则清零 ODR） */

    /* ---- 3. 配 PB1 为上拉输入：CNF=10 MODE=00 → 0b1000，且 ODR=1 选上拉 ---- */
    GPIOB->CRL &= ~(0xFu << (KEY_PIN * 4));
    GPIOB->CRL |=  (0x8u << (KEY_PIN * 4));          /* 0b1000 上拉/下拉输入 */
    GPIOB->ODR |=  (1u << KEY_PIN);                  /* ODR=1 → 上拉（关键一步！） */

    while (1) {
        if ((GPIOB->IDR & (1u << KEY_PIN)) == 0) {   /* 读 IDR，按下=0 */
            delay_ms(20);                            /* 软件去抖（第4周细讲状态机去抖） */
            if ((GPIOB->IDR & (1u << KEY_PIN)) == 0) {
                GPIOB->BSRR = (1u << LED_PIN);       /* 点亮 */
            }
        } else {
            GPIOB->BSRR = (1u << (LED_PIN + 16));    /* BSRR 高 16 位 = 复位，熄灭 */
        }
    }
}
```

**代码里要能解释的 4 点**

1. `CRL &= ~(0xF << (pin*4))` 为什么必须先清零：**读改写**，否则残留位会凑出别的模式（典型 bug：以为设了推挽，实际是复用）。
2. `BSRR` 为什么优于 `ODR |= x`：`BSRR` 是**原子写**（写 1 置位/复位，不影响其他位），`ODR` 是读改写，中断里并发操作会丢状态。
3. `ODR` 那行为什么必须有：上拉/下拉共用 `CNF=10`，**上/下拉靠 ODR 选**。
4. `IDR` 读的是引脚真实电平（**只读寄存器**），`ODR` 读的是你上次写的输出值——**输入模式下不要用 ODR 判断按键**。

---

## 11. HAL 版对照 + CubeMX 操作路径

```c
/* CubeMX 里点 PB0 = GPIO_Output、PB1 = GPIO_Input（Pull-up），生成后： */
GPIO_InitTypeDef GPIO_InitStruct = {0};

__HAL_RCC_GPIOB_CLK_ENABLE();                 /* 对应 RCC->APB2ENR |= IOPBEN */

/* PB0：推挽输出 */
GPIO_InitStruct.Pin   = GPIO_PIN_0;
GPIO_InitStruct.Mode  = GPIO_MODE_OUTPUT_PP;  /* → CNF=00 MODE=11 */
GPIO_InitStruct.Pull  = GPIO_NOPULL;
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH; /* → 50MHz */
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

/* PB1：上拉输入 */
GPIO_InitStruct.Pin  = GPIO_PIN_1;
GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
GPIO_InitStruct.Pull = GPIO_PULLUP;           /* 注意：F1 的 HAL 在 Init 里写 ODR 实现上下拉 */
HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

/* 使用 */
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);       /* 内部就是 BSRR */
HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_1) == GPIO_PIN_RESET) { /* 按下 */ }
HAL_Delay(20);                                            /* 基于 SysTick，依赖时钟正确 */
```

> **先寄存器后 HAL** 的学习顺序很重要：面试问"HAL_GPIO_Init 干了什么"，能答出"把 Mode/Speed 翻译成 CRL/CRH 的 CNF/MODE 四位、Pull 翻译成 ODR 位、再开 APB2 时钟"，这一层答出来就是加分项。

---

## 12. GPIO 六个常见坑

| # | 坑 | 现象 | 对策 |
|---|---|---|---|
| 1 | **忘记开 GPIO 时钟** | 代码全对但灯不亮、IDR 恒 0 | 先 `RCC->APB2ENR \|= IOPxEN` |
| 2 | 上拉/下拉忘了写 `ODR` | 按键读数随机、悬空 | `CNF=10` 后必须设 ODR |
| 3 | 开漏输出没接上拉 | 只能拉低，高电平浮空 | 外接 4.7k~10kΩ 上拉 |
| 4 | 输入模式把 `MODE` 写成非 00 | 引脚意外变成输出 | 输入时 MODE 必须 `00` |
| 5 | 动到 **PA13/PA14**（SWDIO/SWCLK） | 下载器连不上，板子"变砖" | 别配这两个脚；要用 PB3/PB4/PA15 先关 JTAG |
| 6 | 配了 ADC 但引脚还是浮空输入 | 采样值乱跳、不准 | ADC 引脚必须配**模拟输入** |

**第 5 条的救援代码**（要复用 PB3/PB4/PA15，即 JTAG 引脚）：

```c
RCC->APB2ENR |= RCC_APB2ENR_AFIOEN;
AFIO->MAPR = (AFIO->MAPR & ~(7u << 24)) | (2u << 24);   /* SWJ_CFG=010：关 JTAG，留 SWD */
```

---

## 13. 本周动手清单（做完打勾）

- [ ] 装在纸上的默写：时钟树（第 7 节模板）+ GPIO 八种模式编码表（第 8 节）
- [ ] CubeMX 配出 72MHz 并**截图**「Clock Configuration」页存进笔记
- [ ] 读生成的 `SystemClock_Config()`，在旁边手写对应寄存器位（CR/CFGR 具体位）
- [ ] `code/gpio_reg_led.c` 在 Keil 里编译通过并下载运行（按自己原理图改引脚）
- [ ] **开漏实验**：去掉上拉 → 观察现象 → 加上拉 → 恢复，写 3 行结论
- [ ] **时钟自检实验**：故意把 PLLMUL 改成 ×4，观察 `HAL_Delay(500)` 的实际时长变化，再改回 ×9
- [ ] 把 LED 闪灯改成"按键按下常亮、松开常灭"，并把去抖延时从 20ms 改成 0ms 观察抖动现象
- [ ] 三个提交推到 `stm32-projects`：`F103-Template`、`GPIO-Demo-Reg`、`GPIO-Demo-HAL`

---

## 一句话总结

> **时钟：HSE 8M ×9 = 72M → HCLK 72M → APB1 除以 2 得 36M（上限）、APB2 得 72M；Flash 给 2 个等待周期。**
> **GPIO：`CRL/CRH` 每脚 4 位 = `CNF`+`MODE`；输入看 CNF 选模拟/浮空/上下拉（上下拉由 ODR 定），输出看 CNF 选推挽/开漏/复用；开漏想输出高必须外接上拉。**