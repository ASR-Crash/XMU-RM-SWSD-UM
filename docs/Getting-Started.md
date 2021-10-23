## 基础入门 点亮第一盏led灯


	#首先创建一个新的工程文档，然后将会得到一下工程模板
	
	
	#include "stm32f4xx_hal.h"
	#include "can.h"
	#ifdef __cplusplus
	extern "C"
	#endif
	void SysTick_Handler(void)
	{
		HAL_IncTick();
		HAL_SYSTICK_IRQHandler();
	}                                    //以上为头文件加载与系统计时器、中断服务函数初始化
	
	int main(void)                 //主函数，点亮LED灯的重点
	{
		HAL_Init();               //HAL库函数的初始化
	__GPIOD_CLK_ENABLE();     //初始化GPIO对应组的时钟，需要查看原理图找到想要使能的GPIO口的组别（组别从A-G）
	GPIO_InitTypeDef GPIO_InitStructure; //定义一个GPIO初始化结构体，用于配置相关寄存器
	
	GPIO_InitStructure.Pin = GPIO_PIN_12;    //pin脚的设置，需要查看原理图得到对应pin脚编号
	
	GPIO_InitStructure.Mode = GPIO_MODE_OUTPUT_PP;  //设置GPIO的模式
	GPIO_InitStructure.Speed = GPIO_SPEED_HIGH;    //设置GPIO的速度
	GPIO_InitStructure.Pull = GPIO_NOPULL;           //设置GPIO的上下拉状态
	HAL_GPIO_Init(GPIOD, &GPIO_InitStructure);     //GPIO的初始化，一样需要改要使能的GPIO的组别
	
	for (;;)
	{
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_SET);
	     //调用库函数更改使能的GPIO口的状态，入口参数的组别与引脚编号均需要改变
		HAL_Delay(500);                   //调用HAL delay库函数，延迟一定时间，单位ms
		HAL_GPIO_WritePin(GPIOD, GPIO_PIN_12, GPIO_PIN_RESET); //同上
		HAL_Delay(500);
	}
	
	}
