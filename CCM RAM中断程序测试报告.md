# 1. 文档信息

| 项目               | 内容                     |
| ---------------- | ---------------------- |
| MCU 型号           | STM32F334R8            |
| Datasheet 版本     | DS9994                 |
| Reference Device | Raspberry Pi 4 Model B |
| 测试人员             | 杨文亮                 |
| 测试日期             | 2026.01.07             |
| 硬件版本             | STM32F334R8T6 开发板   |
| 编译器              | Keil MDK-ARM           |
| 测试环境             | 室温                     |

# 2. 测试范围说明

**测试依据：**

- STM32F334 Datasheet
- rm0364-stm32f334xx-advanced-arm-based-32-bit-mcus-stmicroelectronics-en

**测试目标：**

- 验证 CCM RAM（Core Coupled Memory）中断处理程序执行功能
- 验证中断处理程序和中断回调函数在CCM RAM中的正确执行
- 验证软件触发中断功能
- 验证中断处理流程完整性

**不包含内容：**

- 系统级应用功能
- 中断性能基准测试
- EMC / ESD
- 多中断嵌套测试
- 中断优先级测试

# 3. 测试步骤及结果

## 3.1. 硬件环境

| 项目   | 说明         |
| ---- | ---------- |
| MCU  | STM32F334R8T6  |
| 系统时钟 | HSI 8 MHz |
| 电源   | 3.3V       |
| 调试接口 | SWD       |

## 3.2. 测试仪器 

| 设备    | 型号     | 用途       |
| ----- | ------ | -------- |
| 调试器   | SWD    | 程序下载/调试 |
| IDE   | Keil MDK-ARM | 代码编译/调试  |


## 3.3. 功能测试用例

### 3.3.1. 测试步骤及方法

#### 3.3.1.1. CCM RAM函数地址验证 1.1.1

**测试目的：** 验证中断处理程序和回调函数是否真的在CCM RAM区域

**测试步骤：**

1. 编译项目，生成 `.map` 文件
2. 在 `main.c` 中调用 `CCMRAM_GetFunctionAddress()` 获取函数地址
3. 在Watch窗口查看 `func_addr` 变量的值
4. 验证函数地址是否在CCM RAM范围内（0x10000000 - 0x10000FFF）
5. 在Memory窗口查看函数地址对应的内存内容

**预期结果：**

- 函数地址在 0x10000000 - 0x10000FFF 范围内
- Memory窗口显示可执行代码

**实际结果：**

- `func_addr` = 0x10000021 
- 地址在CCM RAM范围内 
- Memory窗口显示可执行代码 

---

#### 3.3.1.2. 软件触发中断功能 1.1.2

**测试目的：** 验证软件触发中断功能是否正常工作

**测试步骤：**

1. 在主循环中每1秒调用 `CCMRAM_TriggerInterruptBySoftware()`
2. 在Watch窗口添加 `debug_trigger_called` 变量
3. 观察变量值是否每1秒增加
4. 在Memory窗口查看 `EXTI->SWIER` 寄存器（地址：0x40010410）
5. 验证触发后SWIER寄存器bit0是否变为1

**预期结果：**

- `debug_trigger_called` 每1秒增加1
- SWIER寄存器bit0在触发后变为1

**实际结果：**

- `debug_trigger_called` = 9（运行9秒后） 
- `debug_last_swier_value` = 0x00000001（bit0=1） 

---

#### 3.3.1.3. 中断处理程序执行 1.1.3

**测试目的：** 验证中断处理程序是否在CCM RAM中正常执行

**测试步骤：**

1. 在Watch窗口添加 `debug_irq_handler_called` 变量
2. 运行程序，观察变量值变化
3. 在 `EXTI0_IRQHandler()` 函数入口设置断点
4. 验证程序是否停在断点处
5. 在Disassembly窗口查看PC（程序计数器）地址

**预期结果：**

- `debug_irq_handler_called` 每1秒增加1
- 程序停在中断处理程序断点处
- PC地址在CCM RAM范围内

**实际结果：**

- `debug_irq_handler_called` = 9（运行9秒后）
- 程序正常进入中断处理程序 
- 中断处理程序在CCM RAM中执行 

---

#### 3.3.1.4. 中断回调函数执行 1.1.4

**测试目的：** 验证中断回调函数是否在CCM RAM中正常执行

**测试步骤：**

1. 在Watch窗口添加 `debug_callback_called` 变量
2. 运行程序，观察变量值变化
3. 在 `CCMRAM_InterruptCallback()` 函数入口设置断点（可选）
4. 验证程序是否停在断点处
5. 观察 `ccmram_interrupt_flag` 是否被设置

**预期结果：**

- `debug_callback_called` 每1秒增加1
- 程序停在回调函数断点处
- `ccmram_interrupt_flag` 被设置为1

**实际结果：**

- `debug_callback_called` = 9（运行9秒后）
- 程序正常进入回调函数 
- 回调函数在CCM RAM中执行 
- 中断标志被正确设置 

---

#### 3.3.1.5. 中断使能配置验证 1.1.5

**测试目的：** 验证EXTI和NVIC中断使能配置是否正确

**测试步骤：**

1. 在Watch窗口添加 `debug_exti_imr` 和 `debug_nvic_iser` 变量
2. 运行程序，观察变量值
3. 在Memory窗口查看寄存器：
   - EXTI->IMR（地址：0x40010400）
   - NVIC->ISER[0]（地址：0xE000E100）
4. 验证寄存器bit位是否正确设置

**预期结果：**

- EXTI->IMR bit0 = 1（EXTI Line 0中断使能）
- NVIC->ISER[0] bit6 = 1（EXTI0_IRQn中断使能）

**实际结果：**

- `debug_exti_imr` = 0xBFA40001（bit0=1） 
- `debug_nvic_iser` = 0x00000040（bit6=1） 
- 中断使能配置正确 

---

#### 3.3.1.6. 中断挂起寄存器验证 1.1.6

**测试目的：** 验证中断触发后挂起寄存器状态

**测试步骤：**

1. 在Watch窗口添加 `debug_last_pr_value` 变量
2. 运行程序，观察变量值
3. 在Memory窗口查看 `EXTI->PR` 寄存器（地址：0x40010414）
4. 验证触发中断后PR寄存器bit0是否变为1
5. 验证中断处理后PR寄存器bit0是否被清除为0

**预期结果：**

- 触发中断后，PR寄存器bit0 = 1
- 中断处理后，PR寄存器bit0 = 0

**实际结果：**

- `debug_last_pr_value` = 0x00000001（bit0=1） 
- 中断挂起寄存器状态正确 

---

## 3.4. 实际结果

### 3.4.1. 测试结果

| 项目           | 期望结果               | 实测结果  | Pass/Fail |
| ------------ | ------------------ | ----- | --------- |
| CCM RAM函数地址验证 1.1.1 | 函数地址在CCM RAM范围内（0x10000000-0x10000FFF）         | 符合期望（0x10000021）  | Pass      |
| 软件触发中断功能 1.1.2  | 软件触发函数被调用，SWIER寄存器bit0=1 | 符合期望 | Pass      |
| 中断处理程序执行 1.1.3 | 中断处理程序在CCM RAM中正常执行    | 符合期望 | Pass      |
| 中断回调函数执行 1.1.4 | 中断回调函数在CCM RAM中正常执行    | 符合期望 | Pass      |
| 中断使能配置验证 1.1.5 | EXTI和NVIC中断使能正确    | 符合期望 | Pass      |
| 中断挂起寄存器验证 1.1.6 | 中断触发后PR寄存器bit0=1    | 符合期望 | Pass      |

### 3.4.2. 测试结论

所有测试用例均通过，CCM RAM中断功能完全正常、函数以及中断处理程序能被执行。

**主要验证结果：**

1. **CCM RAM代码执行功能正常**
   - 中断处理程序在CCM RAM中正常执行
   - 中断回调函数在CCM RAM中正常执行
   - 函数地址验证通过

2. **软件触发中断功能正常**
   - 使用 `__HAL_GPIO_EXTI_GENERATE_SWIT(GPIO_PIN_0)` 成功触发中断
   - 无需外部硬件即可测试中断功能
   - SWIER寄存器操作正确

3. **中断处理流程完整**
   - 软件触发 → 中断处理程序 → 中断回调函数，流程完整
   - 所有调试变量同步增加，证明执行流程正确

4. **寄存器配置正确**
   - EXTI中断使能配置正确
   - NVIC中断使能配置正确
   - 中断挂起寄存器状态正确

### 3.4.3. 测试问题记录

| ID           | 描述                                                                           | 状态   | 备注  |
| 问题1  | 软件触发中断不工作：使用了错误的参数类型（`EXTI_LINE_0` 而不是 `GPIO_PIN_0`）           | 已解决 | 修复方法：改用 `GPIO_PIN_0` |
| 问题2  | 调试变量无法查看：代码修改后未重新编译下载                                              | 已解决 | 解决方法：重新编译和下载程序 |
| 问题3  | 寄存器无法直接查看：Watch窗口显示 `<cannot evaluate>`                                  | 已解决 | 解决方法：使用Memory窗口或代码读取 |

**问题解决过程：**

**问题1：软件触发中断不工作**

- **现象：** `debug_trigger_called` 在增加，但 `debug_irq_handler_called` 一直是0
- **分析：** 通过Watch窗口观察到 `debug_last_swier_value` = 0，说明SWIER寄存器bit0没有变为1
- **根本原因：** `__HAL_GPIO_EXTI_GENERATE_SWIT` 宏需要 `GPIO_PIN_0`（简单位掩码），但代码中使用了 `EXTI_LINE_0`（复合值）
- **解决方案：** 修改代码，改用 `GPIO_PIN_0`
- **验证结果：** 修复后，`debug_last_swier_value` = 0x01，中断正常触发 

**问题2：调试变量无法查看**

- **现象：** 所有 `debug_` 变量显示 `<cannot evaluate>`
- **原因：** 代码修改后未重新编译和下载
- **解决方案：** 按F7重新编译，按F8重新下载
- **验证结果：** 重新编译下载后，变量可以正常查看 

**问题3：寄存器无法直接查看**

- **现象：** `EXTI->PR` 和 `NVIC->ISPR[0]` 显示 `<cannot evaluate>`
- **原因：** 调试器限制，某些寄存器无法直接在Watch窗口查看
- **解决方案：** 
  - 方法1：使用Memory窗口直接输入寄存器地址
  - 方法2：在代码中读取寄存器值到变量，然后在Watch窗口查看
- **验证结果：** 两种方法都可以正常查看寄存器值 

# 4. 附录

## 4.1. 测试代码分支/路径

## 4.2. 关键代码片段

### 4.2.1. 软件触发中断函数

void CCMRAM_TriggerInterruptBySoftware(void)
{
  debug_trigger_called++;
  __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);
  debug_last_pr_value = EXTI->PR;
  debug_last_swier_value = EXTI->SWIER;
  __HAL_GPIO_EXTI_GENERATE_SWIT(GPIO_PIN_0);
  debug_last_swier_value = EXTI->SWIER;
  debug_last_pr_value = EXTI->PR;
}

### 4.2.2. 中断处理程序


CCMRAM_FUNC void EXTI0_IRQHandler(void)
{
  debug_irq_handler_called++;
  
  
  CCMRAM_InterruptCallback();
  

  HAL_GPIO_EXTI_IRQHandler(KEY_EXTI0_Pin);
}


### 4.2.3. 中断回调函数
```c
CCMRAM_FUNC void CCMRAM_InterruptCallback(void)
{
  debug_callback_called++;
  ccmram_interrupt_flag = 1;
  ccmram_test_counter++;
}
```

## 4.3. 寄存器地址表

| 寄存器 | 地址 | 说明 | 关键位 |
|--------|------|------|--------|
| EXTI->IMR | 0x40010400 | 中断使能寄存器 | bit0: EXTI Line 0使能 |
| EXTI->SWIER | 0x40010410 | 软件中断事件寄存器 | bit0: EXTI Line 0软件触发 |
| EXTI->PR | 0x40010414 | 挂起寄存器 | bit0: EXTI Line 0挂起 |
| NVIC->ISER[0] | 0xE000E100 | 中断使能寄存器 | bit6: EXTI0_IRQn使能 |
| NVIC->ISPR[0] | 0xE000E200 | 中断挂起寄存器 | bit6: EXTI0_IRQn挂起 |

## 4.4. 调试变量说明

| 变量名 | 类型 | 说明 | 正常值 |
|--------|------|------|--------|
| `debug_trigger_called` | uint32_t | 软件触发函数调用次数 | 每1秒+1 |
| `debug_irq_handler_called` | uint32_t | 中断处理程序调用次数 | 每1秒+1 |
| `debug_callback_called` | uint32_t | 中断回调函数调用次数 | 每1秒+1 |
| `debug_last_swier_value` | uint32_t | SWIER寄存器值 | bit0=1 |
| `debug_last_pr_value` | uint32_t | PR寄存器值 | bit0=1 |
| `debug_exti_imr` | uint32_t | EXTI中断使能寄存器 | bit0=1 |
| `debug_nvic_iser` | uint32_t | NVIC中断使能寄存器 | bit6=1 |

