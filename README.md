# mult-output-power

基于 **STM32F070F6P6** 的多路可调输出电源固件。通过 CH224K 诱骗 USB-PD/QC 电源输出不同电压，配合继电器切换输出通道，并用 OLED 显示运行状态，支持按键设定输出电压/限流阈值与过流、欠压保护。

---
**基于开源项目**
https://oshwhub.com/course-examples/shu-kong-zhi-liu-dian-yuan-ban
## 1. 功能概览

| 功能   | 说明                                        |
| ---- | ----------------------------------------- |
| 电压档位 | 1.8 V / 3.3 V / 5 V / 9 V / 12 V 五档       |
| 电压设定 | CH224K 诱骗 5 V / 9 V / 12 V，再由 LDO 或直通通道输出 |
| 输出通道 | 3 路继电器（1V8 / Vbus / 3V3），任意时刻仅一路导通        |
| 限流保护 | 100 ~ 1500 mA，步进 100 mA（默认 500 mA）        |
| 欠压保护 | 按档位自动设定阈值（见 §6.3）                         |
| 人机交互 | 3 按键 + SSD1306 OLED，三页面循环（电源 / 电压 / 电流）   |
| 采样显示 | ADC 多采样 + 冒泡排序取中间 80 点均值，抑制抖动             |

---

## 2. 硬件平台

- **MCU**：STM32F070F6P6（Cortex-M0，TSSOP20，32 KB Flash / 6 KB RAM）
- **系统时钟**：HSI 8 MHz → PLL ×6 → **48 MHz**（Flash Latency 1）；HSI14 供 ADC，I2C1 时钟源为 HSI
- **CubeMX**：6.18.1，工程文件 [`mult-output-power.ioc`](mult-output-power.ioc)
- **USB-PD 诱骗**：CH224K（CFG1/2/3 三线配置）
- **显示**：SSD1306 128×64 OLED，I2C 地址 `0x78`（7 位地址 0x3C）
- **功率开关**：3 路继电器驱动（低电平有效）

---

## 3. 引脚分配

由 [`Core/Src/gpio.c`](Core/Src/gpio.c)、[`Core/Inc/main.h`](Core/Inc/main.h)、[`Core/Src/adc.c`](Core/Src/adc.c)、[`Core/Src/i2c.c`](Core/Src/i2c.c) 定义。

### 3.1 CH224K 诱骗配置

| 宏          | 引脚      | 模式   |
| ---------- | ------- | ---- |
| `CFG1_Pin` | **PF0** | 推挽输出 |
| `CFG2_Pin` | **PF1** | 推挽输出 |
| `CFG3_Pin` | **PA1** | 推挽输出 |

### 3.2 输出通道（继电器）

| 枚举           | 宏             | 引脚      | 对应输出              |
| ------------ | ------------- | ------- | ----------------- |
| `Relay_1V8`  | `CON_IO3_Pin` | **PA4** | 1.8 V 通道          |
| `Relay_Vbus` | `CON_IO1_Pin` | **PA2** | VBUS 直通（5/9/12 V） |
| `Relay_3V3`  | `CON_IO2_Pin` | **PA3** | 3.3 V 通道          |

输出有效电平由 `relay_init()` 的 `relay_open_level` 决定，当前代码传入 `GPIO_PIN_RESET`（**低电平导通**）。

### 3.3 按键

| 枚举        | 宏         | 引脚      | EXTI            |
| --------- | --------- | ------- | --------------- |
| `SET_KEY` | `SW1_Pin` | **PB1** | `EXTI0_1_IRQn`  |
| `ADD_KEY` | `SW2_Pin` | **PA7** | `EXTI4_15_IRQn` |
| `SUB_KEY` | `SW3_Pin` | **PA6** | `EXTI4_15_IRQn` |

均为**上拉输入 + 下降沿中断**（按下为低电平），中断优先级 2。

### 3.4 模拟与通信

| 功能       | 引脚                    | 说明          |
| -------- | --------------------- | ----------- |
| 电流采样     | **PA0 / ADC_IN0**     | 采样电阻 + 放大器  |
| 电压采样     | **PA5 / ADC_IN5**     | 电阻分压        |
| ADC 参考   | `ADC_CHANNEL_VREFINT` | 用于补偿 VDD 波动 |
| I2C1_SCL | **PA9**               | OLED        |
| I2C1_SDA | **PA10**              | OLED        |

> ADC 采用**扫描模式 + 间断转换**，通道顺序固定为 `IN0 → IN5 → VREFINT`，因此代码中以 `tempValue[0]/[1]/[2]` 顺序取值。

---

## 4. 目录结构

```
mult-output-power-v2.0/
├── Core/                     # CubeMX 生成的初始化与中断代码
│   ├── Inc/                  #   main.h / adc.h / i2c.h / gpio.h / stm32f0xx_hal_conf.h
│   └── Src/                  #   main.c / adc.c / i2c.c / gpio.c / stm32f0xx_it.c ...
├── app/                      # 应用层
│   ├── task_app.c            #   业务主逻辑：UI、按键、电源开关、保护
│   └── task_app.h
├── hardware/                 # 硬件抽象层（每个模块 = 句柄结构体 + 初始化 + 操作）
│   ├── hw_key.c/.h           #   按键扫描（单击/双击/长按识别）
│   ├── hw_con.c/.h           #   继电器控制（互斥导通）
│   └── hw_ch224.c/.h         #   CH224K 诱骗电压配置
├── middle/                   # 中间件
│   ├── mid_oled.c/.h         #   SSD1306 I2C 驱动（含绘点/线/字符）
│   └── mid_oledfont.h        #   字库
├── Drivers/                  # STM32F0xx HAL/LL 驱动 + CMSIS
├── cmake/
│   ├── stm32cubemx/          #   CubeMX 自动生成（会被重新生成覆盖）
│   └── gcc-arm-none-eabi.cmake
├── CMakeLists.txt            # 根构建脚本（用户可修改）
├── CMakePresets.json         # Debug / Release 预设
├── STM32F070xx_FLASH.ld      # 链接脚本
├── startup_stm32f070x6.s     # 启动文件
└── mult-output-power.ioc     # CubeMX 工程配置
```

---

## 5. 软件架构

```
        ┌──────────────────────────────────────┐
        │  Core/Src/main.c                     │
        │  HAL_Init → 时钟 → 外设 → init_task  │
        │  while(1){ trigger_scanf();          │
        │            protectiveScan(); }       │
        └───────────────┬──────────────────────┘
                        │
        ┌───────────────▼──────────────────────┐
        │  app/task_app.c   （应用层）          │
        │  · 状态机 systemparam                │
        │  · OLED 界面 defaultShowUI / setShowUI│
        │  · CurrentShowUI 实时显示             │
        │  · OpenPowerSwitch / ClosePowerSwitch │
        │  · protectiveScan 采样与保护          │
        └───────┬───────────────┬──────────────┘
                │               │
   ┌────────────▼───┐  ┌────────▼─────────────┐
   │ hardware/      │  │ middle/              │
   │  hw_key        │  │  mid_oled            │
   │  hw_con        │  │  (SSD1306 via I2C1)  │
   │  hw_ch224      │  └──────────────────────┘
   └────────────────┘
                │
        ┌───────▼──────────────────────────────┐
        │  Core/  HAL / LL 驱动                 │
        └──────────────────────────────────────┘
```

**分层约定**：`hardware/` 与 `middle/` 只依赖 `main.h`（HAL 层），不含业务逻辑；`app/` 调用下层模块并持有全部系统状态。

### 5.1 系统状态

`task_app.c` 内的 `struct SystemParam`（[`task_app.h`](app/task_app.h)）是唯一全局状态：

| 字段             | 初值           | 含义        |
| -------------- | ------------ | --------- |
| `isOpen`       | `DISABLE`    | 输出开关状态    |
| `currentPage`  | `POWER_PAGE` | 当前页面      |
| `setVolValue`  | `VOL_5V0`    | 设定电压档位    |
| `protectValue` | `500`        | 限流阈值 (mA) |
| `curValue`     | `0`          | 当前电流 (mA) |
| `volValue`     | `0`          | 当前电压 (V)  |

> ⚠️ 命名注意：`VOL_PAGE`/`CUR_PAGE`/`POWER_PAGE` 定义在 [`app/task_app.h`](app/task_app.h)；而 `VOL_1V8`… `VOL_12V` 定义在 [`hardware/hw_ch224.h`](hardware/hw_ch224.h)。两组宏名字相近但含义完全不同（页面号 vs 电压档位）。

---

## 6. 运行逻辑

### 6.1 主循环

[`Core/Src/main.c`](Core/Src/main.c) 中：

```c
init_task();
while (1)
{
    trigger_scanf();    // 按键处理
    protectiveScan();   // 采样 + 保护
}
```

`init_task()` 完成：默认参数 → 按键/CH224K/继电器句柄初始化 → OLED 初始化并刷新默认界面 → ADC 校准。

### 6.2 按键处理

- 按键中断（`HAL_GPIO_EXTI_Callback`）只置位 `key_trigger_flag`，不做实际处理。
- `trigger_scanf()` 检测到标志后，以 10 ms 周期轮询按键，最多 500 次（约 5 s），识别到按下即调用 `key_handle()` 并退出。
- `hw_key.c` 的 `key_scanf()` 按 **10 ms 调用一次** 设计：
  - 按下计数 `keyCnt`：`≥2` 判定有效单击，`≥100`（约 1 s）判定长按。
  - 双击间隔 `DOUBLE_TIME = 40`（约 400 ms），结束后按 `keyCount` 输出 `KeyPress` / `KeyDoublePress` / `KeyLongPress`。

**页面循环**（`SW1`，仅在输出关闭时允许切页）：

```
POWER_PAGE → VOL_PAGE → CUR_PAGE → POWER_PAGE → ...
```

**各页面按键行为**：

| 页面           | `SW2`（ADD）  | `SW3`（SUB）  |
| ------------ | ----------- | ----------- |
| `POWER_PAGE` | 开启输出        | 开启输出        |
| `VOL_PAGE`   | 电压档位 +1（循环） | 电压档位 −1（循环） |
| `CUR_PAGE`   | 限流 +100 mA  | 限流 −100 mA  |

> `POWER_PAGE` 下两个按键功能相同：输出关闭时开启，输出开启时关闭。

### 6.3 输出开关

`OpenPowerSwitch()` 先设置 CH224K 诱骗电压，再导通对应继电器：

| 设定档位  | CH224K 诱骗 | 导通继电器             | 输出通路            |
| ----- | --------- | ----------------- | --------------- |
| 1.8 V | 5 V       | `Relay_1V8`（PA4）  | 5 V → LDO 1.8 V |
| 3.3 V | 5 V       | `Relay_3V3`（PA3）  | 5 V → LDO 3.3 V |
| 5 V   | 5 V       | `Relay_Vbus`（PA2） | VBUS 直通         |
| 9 V   | 9 V       | `Relay_Vbus`（PA2） | VBUS 直通         |
| 12 V  | 12 V      | `Relay_Vbus`（PA2） | VBUS 直通         |

`ClosePowerSwitch()` 将 CH224K 复位为 5 V 并断开全部继电器。

`control_relayx()` 内部**先调用 `relay_close_all()`**，保证任意时刻只有一路导通，实现通道互斥。

### 6.4 采样与保护

`protectiveScan()` 仅在 `isOpen == ENABLE` 时执行：

1. 连续 3 次触发 ADC，依次读取 `IN0`（电流）、`IN5`（电压）、`VREFINT`。
2. 用 VREFINT 归一化，抵消 VDD 波动：
   ```
   电流(mA) = (raw_IN0  / raw_VREFINT) * 1.24 / 50 / 0.02 * 1000
   电压(V)  = (raw_IN5  / raw_VREFINT) * 1.24 * 13.3 / 3.3
   ```
   即电流通道为 **20 mΩ 采样电阻 + 50 倍放大**（≈1 V/A）；电压通道分压比 **13.3/3.3 ≈ 4.03**（量程约 13.3 V）。
3. **保护判定**：电流超 `protectValue` 或电压低于档位阈值即立即关断输出并回到默认界面。欠压阈值：
   | 档位 | 1.8 V | 3.3 V | 5 V | 9 V | 12 V |
   | -- | ----- | ----- | --- | --- | ---- |
   | 阈值 | 1 V   | 2 V   | 4 V | 8 V | 11 V |
4. **显示滤波**：将每次结果存入 100 点环形缓冲，攒满 100 点后冒泡排序，取中间 80 点（索引 10~89）求平均再刷新界面。

---

## 7. 编译

### 7.1 依赖

- `arm-none-eabi-gcc`（需在 `PATH` 中，或由 IDE 指定；本机路径 `F:\CodeTool\DevEnv\GNU-tools-for-STM32\bin`）
- CMake ≥ 3.22、Ninja

### 7.2 命令行构建

```bash
cmake --preset Debug
cmake --build --preset Debug
```

或直接调用已配置好的构建目录：

```bash
cmake --build build/Debug --target mult-output-power -j 14
```

产物位于 `build/Debug/mult-output-power.elf`，同时生成 `mult-output-power.map`。

### 7.3 CLion

直接用 CLion 打开工程根目录，选择 `Debug` 预设构建即可（构建目标 `mult-output-power`）。

### 7.4 构建配置要点

根 [`CMakeLists.txt`](CMakeLists.txt) 中用户代码的接入方式：

```cmake
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
        app/task_app.c  hardware/hw_ch224.c  hardware/hw_con.c
        hardware/hw_key.c  middle/mid_oled.c)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/app
        ${CMAKE_CURRENT_SOURCE_DIR}/hardware
        ${CMAKE_CURRENT_SOURCE_DIR}/middle)
```

> **新增源文件时需同步修改此处。** 头文件路径必须加入 `target_include_directories`，否则会出现 `fatal error: xxx.h: No such file or directory`。

### 7.5 关于 CubeMX 重新生成

- 根 `CMakeLists.txt` 标注为 *generated only once*，重新生成**不会**覆盖你的修改。
- `cmake/stm32cubemx/CMakeLists.txt` **会**被重新生成覆盖，不要在其中做修改。
- `Core/` 下代码请写在 `/* USER CODE BEGIN */` 与 `/* USER CODE END */` 之间，避免被覆盖。

---

## 8. 已知问题与改进建议

按优先级排列，供后续迭代参考：

1. **`systemparam.volValue` / `curValue` 始终为 0**：`protectiveScan()` 只用局部变量 `tempVolValue` / `tempCurValue` 计算并直接送入 `CurrentShowUI()`，从未回写这两个字段。而开启输出时 `key_handle()` 调用的是 `CurrentShowUI(systemparam.volValue, systemparam.curValue)`，因此**每次开输出都会先显示 `0.0V / 0mA`**，要等攒满 100 点采样后才出现真实值。建议在保护扫描的均值分支中回写 `systemparam`，或在开输出时不显示数值而显示占位。
2. **`VOL_PAGE` 下允许改档位但不重设诱骗电压**：输出开启时 `SW1` 被禁止切页，但若通过其他途径改变 `setVolValue`，需重新开关输出才会生效。
3. **限流边界与设计意图不一致**：`CUR_PAGE` 中 `if (protectValue <= 1400) protectValue += 100;` 实际可达 **1500 mA**；`if (protectValue >= 200) protectValue -= 100;` 实际可达 **100 mA**。若上下限应为 1400/200，需改为 `< 1400` 与 `> 200`。
4. **`raw_VREFINT` 可能为 0**：`tempValue[2]` 未做非零判断即参与除法；VREFINT 读取失败时会得到 `inf`/`NaN` 并可能误触发保护。建议增加 `if (tempValue[2] == 0) return;`。
5. **`cumsumNum` 越界风险**：缓冲长度为 100，写入后自增，判断条件为 `cumsumNum >= 99`。逻辑当前正确，但若中途修改缓冲区长度需同步调整，建议改为环形索引。
6. **`bubble_sort()` 对 `len == 0` 不安全**：`len - 1` 为无符号运算会下溢；当前固定传 100 无实际影响。
7. **浮点与 `sprintf` 开销**：Cortex-M0 无 FPU，浮点运算与 `sprintf` 全在软件中完成，主循环较慢。若后续响应不足，可改为定点整数运算。
8. **按键长按/双击已实现但未使用**：`KeyDoublePress`、`KeyLongPress` 在 `key_handle()` 中未处理，可用于增加快捷操作（如长按复位、双击快速开关）。

---

## 9. 开发约定

- 每个硬件模块采用统一模式：**句柄结构体 + `xxx_init()` 返回句柄 + 操作函数接收句柄指针**，不使用全局硬件对象。新增外设请沿用该风格。
- 新增硬件模块放入 `hardware/`，软件中间件放入 `middle/`，业务逻辑放入 `app/`。
- 函数上方保留 `Function Content / Function Parameter / Return Value` 注释块。

---

## 10. 许可

`Drivers/` 下的 STM32 HAL 驱动与 CMSIS 组件版权归 STMicroelectronics / ARM 所有，遵循各自目录中的 `LICENSE.txt`。应用层代码（`app/`、`hardware/`、`middle/`）作者署名见各文件头部。
