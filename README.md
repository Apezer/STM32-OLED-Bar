# STM32-OLED-Bar

STM32F103C8T6 微控制器上实现的 OLED 进度条显示项目，具有中断驱动的计数器和动态进度条动画。

## 项目特性

- ✅ **OLED 显示驱动**：完整的 OLED（128×64）I2C 软件协议实现
- ✅ **进度条动画**：平滑的 0-100% 进度条显示（1% 精度步进）
- ✅ **中断驱动**：计数器由 TIM2 1ms 中断更新，主循环只负责显示
- ✅ **任务分级**：1ms 基础中断，支持 5ms 和 1s 分级任务执行
- ✅ **实时计数显示**：屏幕显示中断计数和当前进度百分比
- ✅ **CMake 构建系统**：支持现代化的 C++ 风格项目管理

## 硬件要求

- **MCU**：STM32F103C8T6（72MHz ARM Cortex-M3）
- **OLED 屏幕**：128×64 像素，I2C 接口
- **连接方式**：
  - OLED SCL → PB8（软件 I2C 时钟）
  - OLED SDA → PB9（软件 I2C 数据）
- **LED 指示灯**：PC13（可选）

## 文件结构

```
A04_OLED_Show/
├── Core/                          # STM32CubeMX 生成的核心代码
│   ├── Inc/                      # 头文件
│   │   ├── main.h
│   │   ├── gpio.h
│   │   ├── tim.h
│   │   └── stm32f1xx_it.h       # 中断处理
│   └── Src/                      # 源文件
│       ├── main.c               # 主程序（显示逻辑）
│       ├── stm32f1xx_it.c       # 中断处理实现
│       └── ...
├── Hardware/                      # 自定义硬件驱动
│   ├── OLED.h / OLED.c          # OLED 显示驱动
│   ├── OLED_Font.h              # 字库数据
│   ├── LED.h / LED.c            # LED 控制
│   └── ...
├── Drivers/                       # STM32 HAL 库
├── cmake/                         # CMake 配置
├── CMakeLists.txt                # 构建配置
└── README.md                      # 本文件
```

## 软件架构

### 主要模块

#### 1. **OLED 驱动** (`Hardware/OLED.c`)
- 软件 I2C 协议实现
- 128×64 像素缓存管理
- 字符串、数字显示函数
- **进度条渲染**：`OLED_ShowProgressBar(row, col, width, percent)`

#### 2. **中断处理** (`Core/Src/stm32f1xx_it.c` 和 `main.c`)
```c
volatile uint32_t count = 0;   // 1ms 中断计数
volatile uint8_t progress = 0; // 进度条百分比（0-100）

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim->Instance == TIM2)
  {
    if(count % 5 == 0)          // 每 5ms 执行一次
    { /* 自定义任务 */ }
    
    if(count % 20 == 0)         // 每 20ms 更新进度条
    { 
      progress++;               // 增加进度
      if (progress > 100) progress = 0;
    }
    
    if(count % 1000 == 0)       // 每 1s 执行一次
    { count = 0; }              // 计数器重置
    
    count++;                     // 中断计数递增
  }
}
```

#### 3. **主程序显示** (`Core/Src/main.c`)
```c
while (1)
{
  OLED_Clear();
  OLED_ShowString(1, 1, "Progress Bar");
  OLED_ShowProgressBar(2, 1, 100, progress);  // 显示进度条
  
  sprintf(text, "Count: %lu", count);        // 显示计数
  OLED_ShowString(3, 1, text);
  
  sprintf(text, "Progress: %3d%%", progress);// 显示百分比
  OLED_ShowString(4, 1, text);
  
  OLED_Refresh();
  HAL_Delay(20);  // 主循环 20ms 刷新一次
}
```

## 构建和烧录

### 使用 CMake + VS Code

1. **安装依赖**：
   - [CMake](https://cmake.org/) 3.22+
   - [ARM GCC Toolchain](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm)
   - STM32CubeProgrammer（用于烧录）

2. **构建项目**：
   ```bash
   cd A04_OLED_Show
   mkdir build && cd build
   cmake ..
   make -j4
   ```

3. **烧录到设备**：
   ```bash
   st-flash write build/A04_OLED_Show.bin 0x8000000
   # 或使用 STM32CubeProgrammer GUI
   ```

### 使用 STM32CubeIDE

1. 打开 `A04_OLED_Show.ioc` 配置文件
2. 生成代码或直接打开项目
3. 编译 → 烧录

## 工作原理

### 时序图

```
TIM2 1ms 中断周期
│
├─ count = 0, 1, 2, ..., 1000
│  ├─ 当 count % 5 == 0   → 执行 5ms 任务
│  ├─ 当 count % 20 == 0  → progress++ （进度条增加）
│  └─ 当 count % 1000 == 0 → count 重置为 0
│
└─ 主循环每 20ms 读取 progress 显示进度条
```

### 显示效果

```
Progress Bar
[███████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░]  // 进度条
Count: 1250
Progress: 50%
```

## 进度条特性

- **宽度**：100 像素
- **高度**：12 像素
- **步进**：1%（每 20ms 增加 1%）
- **速度**：约 2 秒完成 0-100%（100 步 × 20ms）
- **显示方式**：
  - 填充部分：黑色实心矩形
  - 空白部分：仅显示边框

## 自定义修改

### 改变进度条速度

在中断函数中修改步进间隔：
```c
if(count % 10 == 0)  // 改为 10ms 更新，速度加倍
{
  progress++;
}
```

### 添加自定义任务

```c
if(count % 333 == 0)  // 每约 333ms 执行一次
{
  LED_Toggle();       // 切换 LED 状态
}
```

### 修改进度条宽度

```c
OLED_ShowProgressBar(2, 1, 80, progress);  // 改为 80 像素宽度
```

## 故障排除

| 问题 | 解决方案 |
|------|--------|
| 屏幕无显示 | 检查 I2C 连接（PB8, PB9）和电源 |
| 进度条不动 | 检查 TIM2 中断是否启用（`HAL_TIM_Base_Start_IT`） |
| 字符显示乱码 | 确保 OLED 初始化正确，检查字库数据 |
| 计数不增加 | 确认 `volatile` 关键字已添加到全局变量 |

## 版本历史

### v1.0（当前版本）
- ✅ 完整 OLED 驱动实现
- ✅ 中断驱动进度条显示
- ✅ 计数器和百分比显示
- ✅ CMake 构建系统
- ✅ 多级中断任务支持

## 许可证

MIT License - 开源免费使用

## 相关资源

- [STM32F103 数据手册](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf)
- [STM32 HAL 库文档](https://www.st.com/en/embedded-software/stsw-stm32l1xx-cat.html)
- [OLED 显示屏驱动原理](https://github.com/topics/oled-display)

## 联系方式

如有问题或建议，欢迎提交 Issue 或 Pull Request！

---

**最后更新**：2026年4月14日
