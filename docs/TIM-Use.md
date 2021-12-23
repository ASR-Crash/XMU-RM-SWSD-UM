定时器 (Timer) 最基本的功能就是定时了，比如定时发送 USART 数据、定时采集 AD 数据等等。 如果把定时器与 GPIO 结合起来使用的话可以实现非常丰富的功能，可以测量输入信号的脉冲宽 度，可以生产输出波形。定时器生产 PWM 控制电机状态是工业控制普遍方法，这方面知识非常 有必要深入了解。

# tim.h和tim.cpp文件的使用

## tim.h

```c++
#pragma once
#include "stm32f4xx_hal.h"
#include "functional"
#include "gpio.h"
enum{ BASE, PWM, IC };

class TIM
{
    public:
    TIM& Init(uint32_t Mode, TIM_TypeDef* TIM, uint32_t frequency);
    void MspPostInit(reuse r) const;
    void BaseInit(void);
    TIM& PWMInit(uint32_t channel, float duty, reuse r);
    void PWMDuty(uint32_t channel, float duty) const;
    void ICInit(int32_t channel, uint32_t ICPolarity, reuse r);

    bool operator==(const TIM_HandleTypeDef *htim)
    {
        return (&this->htim) == htim;
    }
    TIM_HandleTypeDef htim;
    uint32_t counter;
};

extern TIM tasks, fraction, photogate;
```

## tim.cpp

```c++
#include "tim.h"
#include "can.h"
#include "IMU.h"
#include "motor.h"
#include "control.h"
#include "RC.h"
#include "LED.h"
#include "judgement.h"
#include "powerlimit.h"

TIM_HandleTypeDef *phtim[4];

TIM& TIM::Init(const uint32_t Mode, TIM_TypeDef* TIM, const uint32_t frequency)
{
    const float x = 84000000 / frequency;
    if (x >= 5000)
    {
        htim.Init.Prescaler = x / 5000 - 1u;
        htim.Init.Period = 5000 - 1u;
    }
    else
    {
        htim.Init.Prescaler = 84u - 1u;
        htim.Init.Period = x / 84 - 1u;
    }
    htim.Instance = TIM;
    htim.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    switch (Mode)
    {
        case BASE:HAL_TIM_Base_Init(&htim); break;
        case PWM:HAL_TIM_PWM_Init(&htim); break;
        case IC:HAL_TIM_IC_Init(&htim); break;
        default:;
    }
    if (TIM == TIM2)phtim[1] = &htim;
    if (TIM == TIM3)phtim[2] = &htim;
    if (TIM == TIM4)phtim[3] = &htim;
    return *this;
}
void TIM::MspPostInit(const reuse r) const
{
    GPIO_InitTypeDef GPIO_InitStruct;
    GPIO_CLK_ENABLE(r.GPIOx);

    GPIO_InitStruct.Pin = r.Pin;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    GPIO_InitStruct.Alternate = this->htim.Instance == TIM2 ? GPIO_AF1_TIM2 : GPIO_AF2_TIM3;
    HAL_GPIO_Init(r.GPIOx, &GPIO_InitStruct);
}

extern "C" void TIM2_IRQHandler(void)
{
    if (phtim[1])HAL_TIM_IRQHandler(phtim[1]);
}
extern "C" void TIM3_IRQHandler(void)
{
    if (phtim[2])HAL_TIM_IRQHandler(phtim[2]);
}
extern "C" void TIM4_IRQHandler(void)
{
    if (phtim[3])HAL_TIM_IRQHandler(phtim[3]);
}

void TIM::BaseInit(void)
{
    TIM_ClockConfigTypeDef sClockSourceConfig;
    TIM_MasterConfigTypeDef sMasterConfig;

    sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
    HAL_TIM_ConfigClockSource(&htim, &sClockSourceConfig);

    sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
    sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
    HAL_TIMEx_MasterConfigSynchronization(&htim, &sMasterConfig);
    HAL_TIM_Base_Start_IT(&htim);
}
void HAL_TIM_Base_MspInit(TIM_HandleTypeDef* tim_baseHandle)
{
    SET_BIT(RCC->APB1ENR, 0x1U << (((reinterpret_cast<unsigned>(tim_baseHandle->Instance) - APB1PERIPH_BASE)) / 0x400U));
    /* Delay after an RCC peripheral clock enabling */
    __IO uint32_t tmpreg = READ_BIT(RCC->APB1ENR, 0x1U << (((reinterpret_cast<unsigned>(tim_baseHandle->Instance) - APB1PERIPH_BASE)) / 0x400U));
    UNUSED(tmpreg);
    HAL_NVIC_SetPriority(static_cast<IRQn_Type>((reinterpret_cast<unsigned>(tim_baseHandle->Instance) - APB1PERIPH_BASE) / 0x400U + 28), 0, 0);
    HAL_NVIC_EnableIRQ(static_cast<IRQn_Type>((reinterpret_cast<unsigned>(tim_baseHandle->Instance) - APB1PERIPH_BASE) / 0x400U + 28));
}


extern "C" void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    static uint8_t can1_data[16], can2_data[16];

    if (tasks.counter++, tasks == htim)
    {
        //在这里加入机器人需要执行的任务（update）
    }

    if (tasks.counter % 4 == 0)
    {
        for (auto &motor : can1_motor)motor.Ontimer(can1.data, can1_data);
        for (auto &motor : can2_motor)motor.Ontimer(can2.data, can2_data);
    }

    //can通讯电机输出
    switch (tasks.counter % 5)
    {
        case 0:can1.Transmit(0x200, can1_data); break;
        case 1:can1.Transmit(0x1ff, can1_data + 8); break;
        case 2:can1.Transmit(0x200, can1_data); break;//此处是为了增大底盘输出的频率
        case 3:can2.Transmit(0x200, can2_data); break;
        case 4:can2.Transmit(0x1ff, can2_data + 8); break;
        default:;
    }
}

TIM& TIM::PWMInit(uint32_t channel, float duty, reuse r)
{
    TIM_MasterConfigTypeDef sMasterConfig;
    TIM_OC_InitTypeDef sConfigOC;

    sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
    sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
    HAL_TIMEx_MasterConfigSynchronization(&htim, &sMasterConfig);

    sConfigOC.OCMode = TIM_OCMODE_PWM1;
    sConfigOC.Pulse = static_cast<unsigned>(duty * (htim.Init.Period + 1)) - 1u;
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
    sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
    HAL_TIM_PWM_ConfigChannel(&htim, &sConfigOC, channel);

    this->MspPostInit(r);
    HAL_TIM_PWM_Start(&htim, channel);
    return *this;
}
void TIM::PWMDuty(const uint32_t channel, const float duty) const
{
    *reinterpret_cast<uint32_t*>(reinterpret_cast<uint8_t*>(htim.Instance->CCR1) + (channel - TIM_CHANNEL_1)) = (htim.Init.Period + 1)*duty - 1u;
}
void HAL_TIM_PWM_MspInit(TIM_HandleTypeDef* tim_pwmHandle)
{
    __IO uint32_t tmpreg = 0x00U;
    SET_BIT(RCC->APB1ENR, 0x1U << (((reinterpret_cast<unsigned>(tim_pwmHandle->Instance) - APB1PERIPH_BASE)) / 0x400U));
    /* Delay after an RCC peripheral clock enabling */
    tmpreg = READ_BIT(RCC->APB1ENR, 0x1U << ((reinterpret_cast<unsigned>(tim_pwmHandle->Instance - APB1PERIPH_BASE)) / 0x400U));
    UNUSED(tmpreg);
}


void TIM::ICInit(int32_t channel, uint32_t ICPolarity, reuse r)
{
    TIM_MasterConfigTypeDef sMasterConfig;
    TIM_IC_InitTypeDef sConfigIC;
    this->MspPostInit(r);
    sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
    sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
    HAL_TIMEx_MasterConfigSynchronization(&htim, &sMasterConfig);

    sConfigIC.ICPolarity = ICPolarity;//TIM_INPUTCHANNELPOLARITY_RISING;
    sConfigIC.ICSelection = TIM_ICSELECTION_DIRECTTI;
    sConfigIC.ICPrescaler = TIM_ICPSC_DIV1;
    sConfigIC.ICFilter = 0;
    HAL_TIM_IC_ConfigChannel(&htim, &sConfigIC, channel);
    HAL_TIM_IC_Start_IT(&htim, channel);
}
void HAL_TIM_IC_MspInit(TIM_HandleTypeDef* tim_icHandle)
{
    SET_BIT(RCC->APB1ENR, 0x1U << ((reinterpret_cast<unsigned>(tim_icHandle->Instance - APB1PERIPH_BASE)) / 0x400U));
    /* Delay after an RCC peripheral clock enabling */
    __IO uint32_t tmpreg = READ_BIT(RCC->APB1ENR, 0x1U << ((reinterpret_cast<unsigned>(tim_icHandle->Instance - APB1PERIPH_BASE)) / 0x400U));
    UNUSED(tmpreg);
    HAL_NVIC_SetPriority(static_cast<IRQn_Type>(reinterpret_cast<unsigned>(tim_icHandle->Instance - APB1PERIPH_BASE) / 0x400U + 28), 0, 0);
    HAL_NVIC_EnableIRQ(static_cast<IRQn_Type>(reinterpret_cast<unsigned>(tim_icHandle->Instance - APB1PERIPH_BASE) / 0x400U + 28));
}
extern "C" void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM4)
    {
        if (htim->Channel == HAL_TIM_ACTIVE_CHANNEL_1)
        {
            //HAL_GPIO_TogglePin(GPIOG, GPIO_PIN_1);
            if (!(photogate.counter++ & 1))
                ;
            TIM_MasterConfigTypeDef sMasterConfig;
            TIM_IC_InitTypeDef sConfigIC;
            sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
            sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
            HAL_TIMEx_MasterConfigSynchronization(htim, &sMasterConfig);

            sConfigIC.ICPolarity = TIM_INPUTCHANNELPOLARITY_RISING;
            sConfigIC.ICSelection = TIM_ICSELECTION_DIRECTTI;
            sConfigIC.ICPrescaler = TIM_ICPSC_DIV1;
            sConfigIC.ICFilter = 0;
            HAL_TIM_IC_ConfigChannel(htim, &sConfigIC, htim->Channel);
            HAL_TIM_IC_Start_IT(htim, htim->Channel);
        }
    }
}
```

将我们的项目文件添加如上两个文件，文件中包括TIM定时器的使用和设置，可以更方便我们使用定时器功能来进行机器人复杂功能代码的编写。
