---
layout:     post
title:      "stm8s003中timer2配置初始化中引起中断更新"
subtitle:   " \"Hello World, Hello Blog\""
date:       2018-11-28 22:00:00
author:     "LaDD"
header-img: "img/winston_churchill.jpg"
tags:
    - stm8
    - timer
    - 中断
    - interrupt
---

## 概述

本文简要记述关于stm8s003中初始化timer2立即进入中断的解决方法：
在调试stm8 tim2作为100ms定时器的时候发现，在enable timer2后，不久远远小于100ms大概800us左右就会立即进入中断（更新事件触发），无论是怎样设置先后顺序，以及在enable中断之前清除中断状态位都无法解决进入中断的问题。此处澄清真的不是st的bug，不过这种设计不是我等小白能够领悟到的，哈哈！

## 搜索
在度娘中搜索到的结果一般解决方法都是等待第一次触发后清除事件再打开中断。出于对知（领）识（导）的好（压）奇（迫），便寻找解决方法，最终找到了问题的根源（google大法好：https://community.st.com/s/question/0D50X00009XkWotSAF/premature-tim2-interrupt-happening-immediately-on-timer-start）
 
## 原因
究其原因是因为在初始化中对预分频器（ARRPreload）进行了更新。实际上在触发更新时间后，该寄存器的配置才会生效（spec中有说明），故进入中断的原因是因为预分频器数值默认为0（写文章时并没有考究是不是0，反正远远小于我设置的数值），才在使能后短时间内触发中断，实际上是真的溢出触发了中断。

附上大家喜爱的代码，亲测可用，基于2M HSI CLK

这里没有列出中断函数，清中断神马的就不是问题的根源，不在赘述（代码不在此电脑中，懒得考了）
```
static void TIM2_Start()
{
	GPIO_WriteReverse(GPIOA, GPIO_PIN_1);//测试用
	GPIO_WriteReverse(GPIOA, GPIO_PIN_2);//测试用
	TIM2_Cmd(ENABLE);
}
static void TIM2_Stop()
{
	TIM2_Cmd(DISABLE);
	GPIO_WriteReverse(GPIOA, GPIO_PIN_2);
	TIM2_UpdateDisableConfig(ENABLE);
	TIM2_GenerateEvent(TIM2_EVENTSOURCE_UPDATE);
  TIM2_UpdateDisableConfig(DISABLE);
}
static void TIM2_Config(void)
{
	TIM2_DeInit();

	/* Time base configuration */
	TIM2_TimeBaseInit(TIM2_PRESCALER_128, 0x061b);
	TIM2_ARRPreloadConfig(ENABLE);
	TIM2_UpdateRequestConfig(TIM2_UPDATESOURCE_REGULAR);//中断源选择为只有溢出才能触发
	TIM2_GenerateEvent(TIM2_EVENTSOURCE_UPDATE);//产生更新事件，不触发中断（这就是我的解决方法，此处即更新了预分频器）
	
	TIM2->SR1 &= 0xFE;//清除中断，按常理应该没用
	TIM2->IER |= 0X01;//使能TIMER


}
```