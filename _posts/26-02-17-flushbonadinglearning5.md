---
layout: post
title: STM入门学习记录5-TIM定时器
categories: STM32
description: STM入门学习记录
keywards: note,record,STM32
mermaid: true
mathjax: true
---

STM32入门学习记录5：TIM定时器

学习自江协科技 [STM32入门教程-2023版 细致讲解 中文字幕](https://www.bilibili.com/video/BV1th411z7sn/?p=4&share_source=copy_web&vd_source=67b9019751b1734c92e834bdee04be24)

# 一、TIM简介
### 1. 简介
定时器可以对输入的时钟进行计数，并在计数值达到设定值时触发中断  
16位计数器(执行计数定时的寄存器，每时钟+1)、预分频器(简称**PSC**，可对计数器时钟分频)、自动重装寄存器(简称**ARR**，计数目标值)，这几部分构成**时基单元**，在72MHz计数时钟下可以实现最大59.65s的定时 ( 计算： $\frac{72M}{2^{16} \cdot 2^{16}}$ 得到中断频率后取倒数，若不够可以再串一个)  
不仅具备基本的定时中断功能，而且还包含内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等多种功能

### 2. 分类

| 类型 | 编号 | 总线 | 功能 |
| :--- | :--- | :--- | :--- |
| **高级定时器** | TIM1、TIM8 | APB2 | 拥有通用定时器全部功能，并额外具有重复计数器、死区生成、互补输出、刹车输入等功能 |
| **通用定时器** | TIM2、TIM3、TIM4、TIM5 | APB1 | 拥有基本定时器全部功能，并额外具有内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等功能 |
| **基本定时器** | TIM6、TIM7 | APB1 | 拥有定时中断、主模式触发DAC的功能 |

- 分类依据：复杂度和应用场景
- STM32F103C8T6定时器资源：TIM1、TIM2、TIM3、TIM4

### 3. 基本定时器
根据 $f_{out} = \frac{f_{clk}}{(PSC + 1)(ARR + 1)}$ ，可算出输出频率

![](/images\posts\record\Basic-Timer.png)

向上折箭头代表触发中断，向下折箭头代表产生事件，这里对应的中断和事件分别为**更新中断**和**更新事件**  
基本时钟→PSC→计数器计数自增并与ARR比较→达到ARR设定值，产生更新中断/更新事件  
只支持向上计数  
可将更新事件映射到TRGO触发DAC  

### 4. 通用定时器

![](/images\posts\record\General-Timer.png)

- 除了向上计数外，还支持向下计数，中央对齐
- 除了系统内部72MHz时钟外还可选择外部时钟
- 定时器的编码器接口可以读取正交编码器输出波形


####  4.1 TIMx 内部触发连接与定时器级联
右上角TRGO输出通向其他定时器时，接到定时器的左侧ITR引脚上

| 从定时器 | ITR0 (TS = 000) | ITR1 (TS = 001) | ITR2 (TS = 010) | ITR3 (TS = 011) |
| :--- | :--- | :--- | :--- | :--- |
| **TIM2** | TIM1 | TIM8 | TIM3 | TIM4 |
| **TIM3** | TIM1 | TIM2 | TIM5 | TIM4 |
| **TIM4** | TIM1 | TIM2 | TIM3 | TIM8 |
| **TIM5** | TIM2 | TIM3 | TIM4 | TIM8 |

> 如果某个产品中没有相应的定时器，则对应的触发信号 ITRx 也不存在。

通过这一路就能实现定时器级联的功能

#### 4.2 外部时钟模式1的输入
用作触发输入，可触发定时器的**从模式**

触发输入作为外部时钟的情况eg：  
  
初始化`TIM3` → 使用主模式将更新事件映射到TRGO上 → 初始化`TIM2` → 选择ITR2(即`TIM3`的TRGO) → 选择时钟为外部时钟1  
如此`TIM3`的更新事件便可驱动`TIM2`的时基单元  
  
还可选择TI1F_ED，连接输入捕获单元CH1引脚 (从CH1引脚获得时钟，边沿触发)，向右看，该时钟还可通过TI1FP1，TI2FP2获得，分别连接CH1、CH2引脚时钟  

总之，外部时钟模式1的输入可以是ETR引脚、其他定时器、CH1引脚边沿，或CH1、CH2引脚，一般情况下用ETR引脚即可  

#### 4.3 外部时钟模式2
相比1比较简单，ETR引脚输入时钟通过ETRF进入触发控制器，即可选择作为时基单元时钟
若想在ETR外部引脚提供时钟，或想对ETR时钟计数，将定时器做计数器，即可如此配置

#### 4.4 定时器的主模式输出
位于右上TRGO输出，可把内部一些事件映射到TRGO引脚上，如基本定时器可将更新事件映射到TRGO触发DAC，触发输出范围比基本定时器更广(DAC、ADC、其他定时器)

#### 4.5 输出比较电路
位于右下，有四个通道，分别对应CH1~CH4引脚  
可用于输出PWM波形，驱动电机  

#### 4.6 输入捕获电路
位于左下，有四个通道，分别对应CH1~CH4引脚  
可用于测量输入方波频率等  

#### 4.7 捕获/比较寄存器
位于中下部，为输入捕获和输出比较电路共用，因为二者不能同时使用

### 5. 高级定时器
![](/images\posts\record\Advanced-Timer.png)
- 左上同通用定时器
- 申请中断处添加重复次数计数器，可以实现通过数个计数周期进行1次更新中断/事件
#### 5.1 其余部分
其余部分为输出比较模块升级
- DTG 死区生成电路 为防止开关切换时由于器件不理想导致的直通
- 右侧前三路输出变为一对互补输出，可输出互补PWM波，该电路常用于驱动三相无刷电机 (如四轴飞行器，电动车后轮，电钻等，因为三相无刷电机的电路一般需要三个桥臂，每个桥臂由两个大功率开关管控制，正好使用三路互补输出)
- 刹车输入功能，若外部引脚BKIN产生了刹车信号或内部时钟失效故障，则控制电路会自动切断电机输出

# 二、TIM定时中断
### 1. 定时中断
#### 1.1 基本结构
![](/images\posts\record\Timing-break-basic-structure.png)
#### 1.2 时基单元的时序问题
##### 1.2.1 预分频器
![](/images\posts\record\Time-Sequence-PSC.png)
- 计数器计数频率：CK_CNT = CK_PSC / (PSC + 1)
- CK_PSC 预分频器输入时钟
- CNT_EN 计数器使能，高电平有效
- CK_CNT 计数器时钟，即预分频器的输出给计数器的时钟

预分频器具有缓冲机制，即有两个预分频寄存器，一个供读写用，不直接决定分频系数，另有一个缓冲寄存器(影子寄存器)为真正起作用的寄存器  
比如在某一时刻，将预分频寄存器的数值由0改为1，变化不会立即生效，而是等到本次计数周期结束时产生更新事件才会传递到缓冲寄存器并生效

##### 1.2.2 计数器
![](/images\posts\record\Time-Sequence-Counter.png)
  

计数器溢出频率：  
CK_CNT_OV = $\frac{CK\_CNT} {ARR + 1}$ = $\frac{CK\_PSC} {(PSC + 1)(ARR + 1)}$

- 无/有预装的计数器时序
![](/images\posts\record\Time-Sequence_Counter1.png)
![](/images\posts\record\Time-Sequence_Counter2.png)
通过设置ARPE位选择是否有预装

### 2. RCC时钟树
![](/images\posts\record\RCC-Timer-Tree.png)
一般用外部晶振，若外部晶振电路出现问题则无法以72MHz运行而是8MHz

### 3. 程序示例
#### 3.1 TIM函数
##### 3.1.1 初始化与使能
`TIM_DeInit(TIM_TypeDef* TIMx)`  
恢复缺省配置  
  
`TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct)`  
初始化，参数为TIMx和结构体  
  
`void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState)`  
使能定时器`ENABLE` / `DISABLE`  
  
`void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState)`  
使能外设中断输出  
`TIM_IT_Update`则为达到ARR时触发
  
##### 3.1.2 时钟选择与ETR引脚配置
`void TIM_InternalClockConfig(TIM_TypeDef* TIMx)`  
选择内部时钟(可省略不写，默认使用内部时钟)  
  
`void TIM_ITRxExternalClockConfig(TIM_TypeDef* TIMx, uint16_t TIM_InputTriggerSource)`  
选择ITRx其他定时器的时钟  
  
`void TIM_TIxExternalClockConfig(TIM_TypeDef* TIMx, uint16_t TIM_TIxExternalCLKSource, uint16_t TIM_ICPolarity, uint16_t ICFilter)`  
选择TIx捕获通道的时钟  
  
`void TIM_ETRClockMode1Config(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, uint16_t TIM_ExtTRGPolarity, uint16_t ExtTRGFilter)`  
选择ETR通过外部时钟模式1输入时钟  
参数分别为预分频器，参数分别为TIMx，预分频器，极性和滤波器  
  
`void TIM_ETRClockMode2Config(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, uint16_t TIM_ExtTRGPolarity, uint16_t ExtTRGFilter)`  
选择ETR通过外部时钟模式2输入时钟外部触发  
  
`void TIM_ETRConfig(TIM_TypeDef* TIMx, uint16_t TIM_ExtTRGPrescaler, uint16_t TIM_ExtTRGPolarity, uint16_t ExtTRGFilter)`  
单独配置ETR引脚的预分频器，极性和滤波器参数  
- 这六个函数对应**定时中断基本结构**中的**时钟选择部分**

##### 3.1.3 参数调整
> 结构体参数在初始化后可能仍需更改(自动重装值，预分频值等)，可使用以下函数  

`void TIM_PrescalerConfig(TIM_TypeDef* TIMx, uint16_t Prescaler, uint16_t TIM_PSCReloadMode)`  
更改预分频值，参数为TIMx，预分频值，写入模式(是否立即生效)  
  
`void TIM_ARRPreloadConfig(TIM_TypeDef* TIMx, FunctionalState NewState)`  
自动重装器预装功能设置(使能/失能)  
  
`void TIM_SetCounter(TIM_TypeDef* TIMx, uint16_t Counter)`  
给计数器写入一个值  
  
`void TIM_SetAutoreload(TIM_TypeDef* TIMx, uint16_t Autoreload)`  
给自动重装器写一个值  
  
`uint16_t TIM_GetCounter(TIM_TypeDef* TIMx)`  
获取当前计数器的值  
  
`uint16_t TIM_GetPrescaler(TIM_TypeDef* TIMx)`  
获取当前分频器值  

##### 3.1.4 标志位函数
```
FlagStatus TIM_GetFlagStatus(TIM_TypeDef* TIMx, uint16_t TIM_FLAG);
void TIM_ClearFlag(TIM_TypeDef* TIMx, uint16_t TIM_FLAG);
ITStatus TIM_GetITStatus(TIM_TypeDef* TIMx, uint16_t TIM_IT);
void TIM_ClearITPendingBit(TIM_TypeDef* TIMx, uint16_t TIM_IT);
```
同NVIC的标志位函数

#### 3.2 TIM结构体
- TIM_ClockDivision
决定输入滤波器和边沿检查器电路分频(信号抖动检测时间，影响延迟)
参数可为`TIM_CKD_DIVx`($x$=1,2,4 为分频数)
- TIM_CounterMode
选择计数模式(向上/向下/三种中央对齐)
`TIM_CounterMode_x`(x=Up,Down,CenterAligned$x$ ($x$=1,2,3) )
- TIM_Period
设定ARR值(0~65535)
- TIM_Prescaler
设定PSC值(0~65535)
- TIM_RepetitionCounter
重复定时器，高级定时器才有，不需要用则写0

PSC 和 ARR值可自由设定，若调低ARR增大PSC，则是以较低频率记较少的数，反之同理

**如果需要使用其他文件的变量，则需要在文件上部用extern x 来声明变量**  
**或者直接将中断函数转移至使用的地方**

#### 3.3 代码部分
- Timer.c
```
#include "stm32f10x.h"                  // Device header

void Timer_Init(void)
{
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
	
	TIM_InternalClockConfig(TIM2);
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStructure.TIM_Period = 10000-1;
	TIM_TimeBaseInitStructure.TIM_Prescaler = 7200-1;
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);
	
	TIM_ClearFlag(TIM2, TIM_FLAG_Update);
	TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);
	
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	
	NVIC_InitTypeDef NVIC_InitStructure;
	NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
	NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 2;
	NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
	
	NVIC_Init(&NVIC_InitStructure);
	
	TIM_Cmd(TIM2, ENABLE);
}

```
- main.c
```
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Timer.h"

uint16_t Num;

int main(void)
{
	OLED_Init();
	Timer_Init();
	
	OLED_ShowString(1, 1, "Num:");
	
	while (1)
	{
		OLED_ShowNum(1, 5, Num, 5);
		OLED_ShowNum(2, 5, TIM_GetCounter(TIM2), 5);
	}
}

void TIM2_IRQHandler(void)
{
	if (TIM_GetITStatus(TIM2, TIM_IT_Update) == SET)
	{
		Num++;
		TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
	}
}

```

# 三、TIM输出比较
> 多用于含电机的项目，如智能车、机器人等 

### 1. 简介
缩写为OC(output compare)输出比较
> 另还有IC(input capture)输入捕获，CC(capture/compare)一般表示输入捕获和输出比较的单元  

输出比较可以通过比较CNT(计数器)与CCR(捕获/比较寄存器,R(register)寄存器)寄存器值的关系，来对输出电平进行置1、置0或翻转的操作，用于输出一定频率和占空比的PWM波形  
同上文：
每个高级定时器和通用定时器都拥有4个输出比较通道
高级定时器的前3个通道额外拥有死区生成和互补输出的功能

### 2. PWM波形
PWM（Pulse Width Modulation）脉冲宽度调制
在具有惯性的系统中，可以通过对一系列脉冲的宽度进行调制，来等效地获得所需要的模拟参量，常应用于电机控速等领域
PWM参数：  
- 频率 = 1 / TS 一般为几k到几十kHz          
- 占空比 = $T_{ON}$ / $T_{S}$           
- 分辨率 = 占空比变化步距  

### 3. 输出比较八大模式

| 模式 | 描述 |
| :--- | :--- |
| 冻结 | CNT=CCR时，REF保持为原状态 |
| 匹配时置有效电平 | CNT=CCR时，REF置有效电平 |
| 匹配时置无效电平 | CNT=CCR时，REF置无效电平 |
| 匹配时电平翻转 | CNT=CCR时，REF电平翻转 |
| 强制为无效电平 | CNT与CCR无效，REF强制为无效电平 |
| 强制为有效电平 | CNT与CCR无效，REF强制为有效电平 |
| PWM模式1 | **向上计数：** CNT < CCR时，REF置有效电平，CNT ≥ CCR时，REF置无效电平<br>**向下计数：** CNT > CCR时，REF置有效电平，CNT ≤ CCR时，REF置无效电平 |
| PWM模式2 | **向上计数：** CNT < CCR时，REF置无效电平，CNT ≥ CCR时，REF置有效电平<br>**向下计数：** CNT > CCR时，REF置无效电平，CNT ≤ CCR时，REF置有效电平 |

置有效/无效电平仅为一次性，不常用
电平翻转模式下输出波形频率=更新频率/2(高低电平切换两次为一周期)
最后的PWM模式1/2可输出频率和占空比都可调的PWM波形，两者只有ref极性的区别

- PWM频率：	$Freq = \frac{CK\_PSC} {(PSC + 1)(ARR + 1)}$
- PWM占空比：	$Duty = \frac{CCR}{ARR + 1}$
- PWM分辨率：	$Reso = \frac{1} {ARR + 1}$

![](/images\posts\record\PWM-Basic-Structure.png)

### 4. 相关设备
#### 4.1 舵机
舵机是一种根据输入PWM信号占空比来控制输出角度的装置
**输入PWM信号要求：周期为20ms，高电平宽度为0.5ms~2.5ms**

#### 4.2 直流电机及驱动
直流电机是一种将电能转换为机械能的装置，有两个电极，当电极正接时，电机正转，当电极反接时，电机反转  
直流电机属于大功率器件，GPIO口无法直接驱动，需要配合电机驱动电路来操作  
TB6612是一款双路H桥型的直流电机驱动芯片，可以驱动两个直流电机并且控制其转速和方向  
  
驱动芯片硬件电路:
![](/images\posts\record\Motor-Hardware-circuit.png)
- 引脚详情如图所示，右下为功能表  
- 三个GND内部联通，任选其一使用即可  
- 图中灰色色块相连表示引脚与电机对应关系，其中PWM引脚要接PWM信号输出端  
- 给一个低功率驱动信号，电机芯片即可从电机电源汲取电流，由此实现低功率控制信号控制大功率信号  
- STBY为待机模式控制，接逻辑电源VCC正常工作，GND则待机，可使用GPIO口控制  

### 5.示例程序
#### 5.1 PWM驱动LED呼吸灯
##### 5.1.1 思路
1. RCC开启时钟(TIM,GPIO)
2. 配置时基单元
3. 配置输出比较单元
4. 配置GPIO(复用推挽输出)
5. 运行控制，启用计数器，输出PWM

##### 5.1.2 TIM外设函数(续)
`void TIM_OC1Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct)`  
`void TIM_OC2Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct)`  
`void TIM_OC3Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct)`  
`void TIM_OC4Init(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct)`  
**分别配置四个输出单元**  

`void TIM_CtrlPWMOutputs(TIM_TypeDef* TIMx, FunctionalState NewState)`  
仅高级定时器使用，输出PWM时需要调用使能主输出，否则无法正常输出  

`void TIM_OCStructInit(TIM_OCInitTypeDef* TIM_OCInitStruct)`  
<font color=green>当结构体成员较多时，可能不便逐个配置，可以先使用结构体初始化函数，再根据需要进行配置</font>  

>以下为部分小功能配置，使用不多，不太要求掌握

`void TIM_ForcedOC1Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction)`  
`void TIM_ForcedOC2Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction)`  
`void TIM_ForcedOC3Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction)`  
`void TIM_ForcedOC4Config(TIM_TypeDef* TIMx, uint16_t TIM_ForcedAction)`  
配置强制输出模式，可以直接配置占空比实现  

`void TIM_OC1PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload)`  
`void TIM_OC2PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload)`  
`void TIM_OC3PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload)`  
`void TIM_OC4PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload)`  
配置CCR寄存器的预装功能(即影子寄存器)  

`void TIM_OC1FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast)`  
`void TIM_OC2FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast)`  
`void TIM_OC3FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast)`  
`void TIM_OC4FastConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCFast)`  
配置快速使能，手册中单脉冲模式有提及  

`void TIM_ClearOC1Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear)`  
`void TIM_ClearOC2Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear)`  
`void TIM_ClearOC3Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear)`  
`void TIM_ClearOC4Ref(TIM_TypeDef* TIMx, uint16_t TIM_OCClear)`  
外部事件时清除REF信号，手册该节有提及

>截止线

`void TIM_OC1PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity)`  
`void TIM_OC1NPolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCNPolarity)`  
`void TIM_OC2PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity)`  
`void TIM_OC2NPolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCNPolarity)`  
`void TIM_OC3PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity)`  
`void TIM_OC3NPolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCNPolarity)`  
`void TIM_OC4PolarityConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPolarity)`  
单独配置输出比较极性，带n的即为高级定时器的互补通道配置  

`void TIM_CCxCmd(TIM_TypeDef* TIMx, uint16_t TIM_Channel, uint16_t TIM_CCx)`  
`void TIM_CCxNCmd(TIM_TypeDef* TIMx, uint16_t TIM_Channel, uint16_t TIM_CCxN)`  
单独修改输出使能参数  

`void TIM_SelectOCxM(TIM_TypeDef* TIMx, uint16_t TIM_Channel, uint16_t TIM_OCMode)`  
选择输出比较模式  

`void TIM_SetCompare1(TIM_TypeDef* TIMx, uint16_t Compare1)`  
`void TIM_SetCompare2(TIM_TypeDef* TIMx, uint16_t Compare2)`  
`void TIM_SetCompare3(TIM_TypeDef* TIMx, uint16_t Compare3)`  
`void TIM_SetCompare4(TIM_TypeDef* TIMx, uint16_t Compare4)`  
<font color=green>单独修改CCR寄存器值，比较重要，可在运行时修改占空比</font>  

##### 5.1.3 输出比较的结构体成员
- 带n的和IdleState都是高级定时器才需要用到的
- TIM_OCMode
设置输出比较模式
`TIM_OCMode_Timing` 冻结模式  
`TIM_OCMode_Active` 相等时置有效电平 
`TIM_OCMode_Inactive` 相等时置无效电平  
`TIM_OCMode_PWM1`  
`TIM_OCMode_PWM2` PWM模式1、2  
- TIM_OCPolarity
设置输出比较极性  
`TIM_OCPolarity_High` ref电平不变；有效电平为高，ref有效输出高电平  
`TIM_OCPolarity_Low` ref电平取反；有效电平为低，ref有效输出为低电平  
- TIM_OutputState
设置输出使能  
`TIM_OutputState_Disable`  
`TIM_OutputState_Enable` 失能/使能  
- TIM_Pulse
设置CCR寄存器值  

##### 5.1.4 程序
- PWM.c
```
#include "stm32f10x.h"                  // Device header

void PWM_Init(void)
{
	
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA,ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	
	TIM_InternalClockConfig(TIM2);
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStructure;
	TIM_TimeBaseInitStructure.TIM_ClockDivision = TIM_CKD_DIV1;
	TIM_TimeBaseInitStructure.TIM_CounterMode = TIM_CounterMode_Up;
	TIM_TimeBaseInitStructure.TIM_Period = 100-1;
	TIM_TimeBaseInitStructure.TIM_Prescaler = 720-1;
	TIM_TimeBaseInitStructure.TIM_RepetitionCounter = 0;
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStructure);
	
	TIM_OCInitTypeDef TIM_OCInitStructure;
	TIM_OCStructInit(&TIM_OCInitStructure);
	TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
	TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
	TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
	TIM_OCInitStructure.TIM_Pulse = 50;
	TIM_OC1Init(TIM2, &TIM_OCInitStructure);

	
	TIM_Cmd(TIM2, ENABLE);
}	

```
由此，完成初始化后可以发现调节占空比即可调整LED亮度


##### 5.1.5 引脚重映射
引脚详情见[GPIO章节]({% post_url 26-01-22-flushbonadinglearning2 %})  
当复用占用相同引脚时，可以进行重定义，若列表中未找到对应的重映射，则无法更改位置  
如 TIM2 的 CH1 通道可以重映射至PA15引脚
可使用AFIO
`void GPIO_PinRemapConfig(uint32_t GPIO_Remap, FunctionalState NewState)`  
引脚重映射配置，具体参数配置可查看手册AFIO节
![](/images\posts\record\TIM-remapping.png)
由引脚定义可看到PA15默认复用为了调试端口JTDI (JTAG Test Data in) ，若想作为普通GPIO则需要关闭调试端口复用
>同理，JTDO即调试端口输出，JTRST为复位
```
\\GPIO.c内规定
  *     @arg GPIO_Remap_SWJ_NoJTRST      :解除JTRST引脚的复用 Full SWJ Enabled (JTAG-DP + SW-DP) but without JTRST
  *     @arg GPIO_Remap_SWJ_JTAGDisable  :解除JTAG调试端口复用，一并释放JTDI,JTDO,JTRST所复用的引脚 JTAG-DP Disabled and SW-DP Enabled
  *     @arg GPIO_Remap_SWJ_Disable      :将SWD，JTAG调试端口全部解除 Full SWJ Disabled (JTAG-DP + SW-DP)
\\SWJ即SWD和JATG两种调试模式
```  
  
**<font color=red>非特殊情况不建议将调试端口全部解除，否则将无法使用ST-Link下载程序，只能使用串口下载新的没用解除调试端口的程序</font>**  
  
![](/images\posts\record\Debug-Port.png)

如：将TIM2重映射到PA15  
```
GPIO_PinRemapConfig(GPIO_PartialRemap1_TIM2,ENABLE);
GPIO_PinRemapConfig(GPIO_Remap_SWJ_JTAGDisable, ENABLE);
```  

**<font color=red>切记顺序不可改变！</font>**
<font color=green>  
由于TIM2 的重映射位是正常的读写位（可读可写）；而SWJ_CFG 位（控制 JTAG/SWD 模式的位）在硬件手册里被定义为只写，读取时将返回000或无定义，而 000 代表的恰好是硬件默认状态：“开启完整 JTAG”  
若反过来，执行重映射 TIM2 时，C语言的标准库会做一个“读-修改-写”的操作
-  读取当前的 AFIO_MAPR 寄存器。因为 SWJ_CFG 是只写位，读出来的这部分全是 0
-  把 TIM2 对应的位改成你需要的值，其他位保持刚才读出来的样子（也就是把 SWJ_CFG 保持为 000）
-  把修改后的值写回寄存器

由此，JTAG再次被开启并占用PA15  

正确的写法中：
执行第 1 句时： 程序修改 TIM2 的位，并将 SWJ_CFG 设为 000。此时 JTAG 本来就是开启的，所以状态没变
执行第 2 句时： GPIO_Remap_SWJ_JTAGDisable 在标准库底层做了特殊处理，它会显式地向 SWJ_CFG 写入关闭 JTAG 的指令，同时保留之前已经设置好的 TIM2 重映射位
</font>  


#### 5.2 PWM驱动舵机
程序同上，更换引脚，通道即可  
由前文可知，**舵机由PWM高电平长决定角度，根据要求为0.5ms-2.5ms(0°~180°)，50Hz**  
因此可以配置占空比和周期来进行控制  
如：  
ARR=20000-1 PSC=72-1 此时满足频率50Hz  
此时调整CCR更改占空比来更改角度，500则为0°，1500对应90°，2500对应180°  
