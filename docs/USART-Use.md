串口USART通信可以支持我们使用一些扩展设备，如陀螺仪、遥控器、裁判系统等等

# usart.h和usart.cpp文件的使用

## usart.h

```c++
#pragma once
#include "stm32f4xx_hal.h"
#include "stm32f4xx_hal_uart.h"

#define UART_MAX_LEN 100

class UART
{
    public:
    UART& Init(USART_TypeDef* Instance, uint32_t BaudRate);
    UART& DMATxInit(void);
    UART& DMARxInit(const uint8_t *buffer = nullptr,const uint32_t size = UART_MAX_LEN);

    void DMATransmit(uint8_t* pData, uint32_t Size) const;
    void OnUARTITHandler(uint16_t Size);
    void OnDMAITHandler(void) const;
    void RxIdleItCallback(uint16_t rxSize) const;
    void UARTTransmit(uint8_t* pData, uint32_t Size);

    static void DMAClearAllFlags(DMA_HandleTypeDef *dmax)
    {
        const auto ele = reinterpret_cast<uint32_t>(dmax->Instance);

        (ele == reinterpret_cast<uint32_t>DMA1_Stream0) ? (DMA1->LIFCR = 0x0000003D) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream1) ? (DMA1->LIFCR = 0x00000F40) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream2) ? (DMA1->LIFCR = 0x003D0000) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream3) ? (DMA1->LIFCR = 0x0F400000) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream4) ? (DMA1->HIFCR = 0x0000003D) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream5) ? (DMA1->HIFCR = 0x00000F40) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream6) ? (DMA1->HIFCR = 0x003D0000) :\
            (ele == reinterpret_cast<uint32_t>DMA1_Stream7) ? (DMA1->HIFCR = 0x0F400000) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream0) ? (DMA2->LIFCR = 0x0000003D) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream1) ? (DMA2->LIFCR = 0x00000F40) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream2) ? (DMA2->LIFCR = 0x003D0000) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream3) ? (DMA2->LIFCR = 0x0F400000) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream4) ? (DMA2->HIFCR = 0x0000003D) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream5) ? (DMA2->HIFCR = 0x00000F40) :\
            (ele == reinterpret_cast<uint32_t>DMA2_Stream6) ? (DMA2->HIFCR = 0x003D0000) :\
            (DMA2->HIFCR = 0x0F400000);
    }

    UART_HandleTypeDef huart;
    uint8_t m_uartrx[UART_MAX_LEN];
};

extern UART uart1,uart2, uart4, uart6;// , uart6, uart7;
```

## usart.cpp

```c++
#include "usart.h"
#include "RC.h"
#include "judgement.h"

UART& UART::Init(USART_TypeDef* Instance, const uint32_t BaudRate)
{
    huart.Instance = Instance;
    huart.Init.BaudRate = BaudRate;
    huart.Init.WordLength = UART_WORDLENGTH_8B;
    huart.Init.StopBits = UART_STOPBITS_1;
    huart.Init.Parity = UART_PARITY_NONE;
    huart.Init.Mode = UART_MODE_TX_RX;
    huart.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart.Init.OverSampling = UART_OVERSAMPLING_16;
    HAL_UART_Init(&huart);

    return *this;
}
UART& UART::DMATxInit(void)
{
    __HAL_DMA_DISABLE(huart.hdmatx);
    huart.hdmatx->Instance->PAR = reinterpret_cast<uint32_t>(&huart.Instance->DR);
    huart.hdmatx->Instance->M0AR = 0;
    DMAClearAllFlags(huart.hdmatx);
    //开启DMA发送中断
    __HAL_DMA_ENABLE_IT(huart.hdmatx, DMA_IT_TC);
    SET_BIT(huart.Instance->CR3, USART_CR3_DMAT);
    return *this;
}
UART& UART::DMARxInit(const uint8_t *buffer, const uint32_t size)
{
    //使能串口DMA接收
    __HAL_DMA_DISABLE(huart.hdmarx);
    huart.hdmarx->Instance->PAR = reinterpret_cast<uint32_t>(&huart.Instance->DR);//PAR is DMA stream x peripheral address register
    huart.hdmarx->Instance->NDTR = size;
    if (buffer == nullptr)buffer = m_uartrx;
    huart.hdmarx->Instance->M0AR = reinterpret_cast<uint32_t>(buffer);
    DMAClearAllFlags(huart.hdmarx);
    __HAL_DMA_ENABLE(huart.hdmarx);
    SET_BIT(huart.Instance->CR3, USART_CR3_DMAR);
    //开启串口空闲中断
    __HAL_UART_CLEAR_PEFLAG(&huart);
    __HAL_UART_ENABLE_IT(&huart, UART_IT_IDLE);
    return *this;
}
void HAL_UART_MspInit(UART_HandleTypeDef* uartHandle)
{
    GPIO_InitTypeDef GPIO_InitStruct;
    if (uartHandle->Instance == USART1)
    {
        static DMA_HandleTypeDef hdma_rx;
        static DMA_HandleTypeDef hdma_tx;
        /* USART6 clock enable */
        __HAL_RCC_USART1_CLK_ENABLE();
        __HAL_RCC_GPIOA_CLK_ENABLE();

        /**USART6 GPIO Configuration
    		PG14     ------> USART6_TX
    		PG9     ------> USART6_RX
    		*/
        GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Pull = GPIO_PULLUP;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
        GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

        /* USART6 DMA Init */
        __HAL_RCC_DMA2_CLK_ENABLE();
        /* USART6_RX Init */
        hdma_rx.Instance = DMA2_Stream2;
        hdma_rx.Init.Channel = DMA_CHANNEL_4;
        hdma_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
        hdma_rx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_rx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_rx.Init.Mode = DMA_NORMAL;
        hdma_rx.Init.Priority = DMA_PRIORITY_LOW;
        hdma_rx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_rx);

        __HAL_LINKDMA(uartHandle, hdmarx, hdma_rx);

        /* USART6_TX Init */
        hdma_tx.Instance = DMA2_Stream7;
        hdma_tx.Init.Channel = DMA_CHANNEL_4;
        hdma_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
        hdma_tx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_tx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_tx.Init.Mode = DMA_NORMAL;
        hdma_tx.Init.Priority = DMA_PRIORITY_LOW;
        hdma_tx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_tx);

        __HAL_LINKDMA(uartHandle, hdmatx, hdma_tx);

        /* USART6 interrupt Init */
        HAL_NVIC_SetPriority(USART1_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(USART1_IRQn);

        /* DMA interrupt init */
        /* DMA2_Stream1_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA2_Stream2_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA2_Stream2_IRQn);
        /* DMA2_Stream6_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA2_Stream7_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA2_Stream7_IRQn);
    }
    if (uartHandle->Instance == USART2)
    {
        static DMA_HandleTypeDef hdma_rx;
        static DMA_HandleTypeDef hdma_tx;
        /* USART6 clock enable */
        __HAL_RCC_USART2_CLK_ENABLE();
        __HAL_RCC_GPIOA_CLK_ENABLE();

        /**USART6 GPIO Configuration
    		PG14     ------> USART6_TX
    		PG9     ------> USART6_RX
    		*/
        GPIO_InitStruct.Pin = GPIO_PIN_2 | GPIO_PIN_3;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Pull = GPIO_PULLUP;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
        GPIO_InitStruct.Alternate = GPIO_AF7_USART2;
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

        /* USART6 DMA Init */
        __HAL_RCC_DMA1_CLK_ENABLE();
        /* USART6_RX Init */
        hdma_rx.Instance = DMA1_Stream5;
        hdma_rx.Init.Channel = DMA_CHANNEL_4;
        hdma_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
        hdma_rx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_rx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_rx.Init.Mode = DMA_NORMAL;
        hdma_rx.Init.Priority = DMA_PRIORITY_LOW;
        hdma_rx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_rx);

        __HAL_LINKDMA(uartHandle, hdmarx, hdma_rx);

        /* USART6_TX Init */
        hdma_tx.Instance = DMA1_Stream6;
        hdma_tx.Init.Channel = DMA_CHANNEL_4;
        hdma_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
        hdma_tx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_tx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_tx.Init.Mode = DMA_NORMAL;
        hdma_tx.Init.Priority = DMA_PRIORITY_LOW;
        hdma_tx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_tx);

        __HAL_LINKDMA(uartHandle, hdmatx, hdma_tx);

        /* USART6 interrupt Init */
        HAL_NVIC_SetPriority(USART2_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(USART2_IRQn);

        /* DMA interrupt init */
        /* DMA2_Stream1_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA1_Stream5_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA1_Stream5_IRQn);
        /* DMA2_Stream6_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA1_Stream6_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA1_Stream6_IRQn);
    }
    if (uartHandle->Instance == UART4)
    {
        static DMA_HandleTypeDef hdma_rx;
        static DMA_HandleTypeDef hdma_tx;
        /* USART1 clock enable */
        __HAL_RCC_UART4_CLK_ENABLE();
        __HAL_RCC_GPIOA_CLK_ENABLE();

        /**USART1 GPIO Configuration
    		PA0     ------> UART4_TX
    		PA1     ------> UART4_RX
    		*/
        GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Pull = GPIO_PULLUP;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
        GPIO_InitStruct.Alternate = GPIO_AF8_UART4;
        HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

        /* USART1 DMA Init */
        __HAL_RCC_DMA1_CLK_ENABLE();
        /* USART1_RX Init */
        hdma_rx.Instance = DMA1_Stream2;
        hdma_rx.Init.Channel = DMA_CHANNEL_4;
        hdma_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
        hdma_rx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_rx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_rx.Init.Mode = DMA_NORMAL;
        hdma_rx.Init.Priority = DMA_PRIORITY_VERY_HIGH;
        hdma_rx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_rx);

        __HAL_LINKDMA(uartHandle, hdmarx, hdma_rx);
        /* USART1_TX Init */
        hdma_tx.Instance = DMA1_Stream4;
        hdma_tx.Init.Channel = DMA_CHANNEL_4;
        hdma_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
        hdma_tx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_tx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_tx.Init.Mode = DMA_NORMAL;
        hdma_tx.Init.Priority = DMA_PRIORITY_VERY_HIGH;
        hdma_tx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_tx);
        __HAL_LINKDMA(uartHandle, hdmatx, hdma_tx);

        /* USART1 interrupt Init */
        HAL_NVIC_SetPriority(UART4_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(UART4_IRQn);

        /* DMA interrupt init */
        /* DMA2_Stream2_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA1_Stream2_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA1_Stream2_IRQn);
        /* DMA2_Stream7_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA1_Stream4_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA1_Stream4_IRQn);
    }
    if (uartHandle->Instance == USART6)
    {
        static DMA_HandleTypeDef hdma_rx;
        static DMA_HandleTypeDef hdma_tx;
        /* USART1 clock enable */
        __HAL_RCC_USART6_CLK_ENABLE();
        __HAL_RCC_GPIOC_CLK_ENABLE();

        /**USART1 GPIO Configuration
    		PC6     ------> USART6_TX
    		PC7     ------> USART6_RX
    		*/
        GPIO_InitStruct.Pin = GPIO_PIN_6 | GPIO_PIN_7;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Pull = GPIO_PULLUP;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
        GPIO_InitStruct.Alternate = GPIO_AF8_USART6;
        HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

        /* USART6 DMA Init */
        __HAL_RCC_DMA2_CLK_ENABLE();
        /* USART6_RX Init */
        hdma_rx.Instance = DMA2_Stream1;
        hdma_rx.Init.Channel = DMA_CHANNEL_5;
        hdma_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
        hdma_rx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_rx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_rx.Init.Mode = DMA_NORMAL;
        hdma_rx.Init.Priority = DMA_PRIORITY_LOW;
        hdma_rx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_rx);

        __HAL_LINKDMA(uartHandle, hdmarx, hdma_rx);
        /* USART6_TX Init */
        hdma_tx.Instance = DMA2_Stream6;
        hdma_tx.Init.Channel = DMA_CHANNEL_5;
        hdma_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
        hdma_tx.Init.PeriphInc = DMA_PINC_DISABLE;
        hdma_tx.Init.MemInc = DMA_MINC_ENABLE;
        hdma_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
        hdma_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
        hdma_tx.Init.Mode = DMA_NORMAL;
        hdma_tx.Init.Priority = DMA_PRIORITY_LOW;
        hdma_tx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
        HAL_DMA_Init(&hdma_tx);
        __HAL_LINKDMA(uartHandle, hdmatx, hdma_tx);

        /* USART1 interrupt Init */
        HAL_NVIC_SetPriority(USART6_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(USART6_IRQn);

        /* DMA interrupt init */
        /* DMA2_Stream2_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA2_Stream1_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA2_Stream1_IRQn);
        /* DMA2_Stream7_IRQn interrupt configuration */
        HAL_NVIC_SetPriority(DMA2_Stream6_IRQn, 0, 0);
        HAL_NVIC_EnableIRQ(DMA2_Stream6_IRQn);
    }
}

void UART::RxIdleItCallback(const uint16_t rxSize) const
{
    if (huart.Instance == USART1)
    {

        if (nuc.rxData[0] == 'w'&&nuc.rxData[1] == 'h'&&nuc.rxData[2] == 'i'&&nuc.rxData[3] == 'c'&&nuc.rxData[4] == 'h')
        {
            if (judgement.ready == true)
            {
                uint8_t red[2] = "R";
                uint8_t blue[2] = "B";
                if (judgement.data.ext_game_robot_status_t.robot_id < 10)
                {
                    uart1.UARTTransmit(red, 1);
                }
                else
                {
                    uart1.UARTTransmit(blue, 1);
                }
            }
        }
        else

            nuc.OnIRQHandler(rxSize);
    }
    if (huart.Instance == USART2)
    {
        rc.OnIRQHandler(rxSize);
    }
    if (huart.Instance == UART4)
    {
        imu.OnIRQHandler(rxSize);
    }
    if (huart.Instance == USART6)
    {
        judgement.OnIRQHandler(rxSize);
    }
}
void UART::OnUARTITHandler(const uint16_t Size = UART_MAX_LEN)
{
    if (__HAL_UART_GET_FLAG(&huart, UART_FLAG_IDLE) && __HAL_UART_GET_IT_SOURCE(&huart, UART_IT_IDLE))
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart);
        __HAL_DMA_DISABLE(huart.hdmarx);

        DMAClearAllFlags(huart.hdmarx);
        RxIdleItCallback(Size - __HAL_DMA_GET_COUNTER(huart.hdmarx));

        __HAL_DMA_SET_COUNTER(huart.hdmarx, Size);
        __HAL_DMA_ENABLE(huart.hdmarx);
    }
    HAL_UART_IRQHandler(&huart);
}
void UART::OnDMAITHandler(void) const
{
    UART::DMAClearAllFlags(huart.hdmatx);
    __HAL_DMA_DISABLE(huart.hdmatx);
}


extern "C" void USART1_IRQHandler(void)
{
    uart1.OnUARTITHandler();
}
extern "C" void USART2_IRQHandler(void)
{
    uart2.OnUARTITHandler();
}
//extern "C" void USART6_IRQHandler(void)
//{
//	uart6.OnUARTITHandler();
//}
extern "C" void UART4_IRQHandler(void)
{
    uart4.OnUARTITHandler();
}

extern "C" void USART6_IRQHandler(void)//Question
{
    uart6.OnUARTITHandler();
}

extern "C" void DMA1_Stream2_IRQHandler(void)
{
}
extern "C" void DMA1_Stream4_IRQHandler(void)
{
    uart4.OnDMAITHandler();
}

extern "C" void DMA2_Stream6_IRQHandler(void)
{
    uart6.OnDMAITHandler();
}

void UART::DMATransmit(uint8_t*const pData, const uint32_t Size) const
{
    __HAL_DMA_DISABLE(huart.hdmatx);
    huart.hdmatx->Instance->NDTR = Size;
    huart.hdmatx->Instance->M0AR = reinterpret_cast<uint32_t>(pData);
    DMAClearAllFlags(huart.hdmatx);
    __HAL_DMA_ENABLE(huart.hdmatx);
}
void UART::UARTTransmit(uint8_t* pData, uint32_t Size)
{
    HAL_UART_Transmit(&huart, static_cast<uint8_t*>(pData), Size, 0xffff);
}

extern "C" void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
}

```

将我们的项目文件添加如上两个文件，文件中包括USART通信的收发方式，可以更方便我们使用USART通信来进行扩展模块代码的编写。