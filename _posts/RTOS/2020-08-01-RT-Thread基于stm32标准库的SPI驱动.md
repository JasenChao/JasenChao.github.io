---
layout: post
title: RT-Thread基于STM32标准库的SPI驱动
tags: [RT-Thread, STM32]
categories: 文章
---

* TOC
{:toc}

发现rt-thread在某个版本更新中，stm32 BSP下的库函数从标准库切换到了HAL库，HAL库应该是stm32日后发展的主流，但是个人感觉标准库更简洁，易于理解，因此在旧版的RTT上改写了一版SPI的驱动，便于加深对SPI的理解。

# 一、内核中的SPI device

SPI包含以下几个结构体：

```c
struct rt_spi_device
{
    struct rt_device parent;
    struct rt_spi_bus *bus;

    struct rt_spi_configuration config;
    void   *user_data;
};

struct rt_spi_bus
{
    struct rt_device parent;
    rt_uint8_t mode;
    const struct rt_spi_ops *ops;

    struct rt_mutex lock;
    struct rt_spi_device *owner;
};

struct rt_spi_ops
{
    rt_err_t (*configure)(struct rt_spi_device *device, struct rt_spi_configuration *configuration);
    rt_uint32_t (*xfer)(struct rt_spi_device *device, struct rt_spi_message *message);
};
```

因此，我们在驱动中需要实现的部分，就是将stm32的SPI总线注册到内核中，并且实现底层的ops，以及SPI设备的挂载。

# 二、驱动中的结构体

```c
struct stm32_hw_spi_cs
{
    GPIO_TypeDef* GPIOx;
    uint16_t GPIO_Pin;
};

struct stm32_spi
{
    SPI_TypeDef *instance;
    char *bus_name;
    SPI_InitTypeDef init;
    struct rt_spi_bus spi_bus;
};
```

stm32_hw_spi_cs是片选引脚的结构体，用于设备挂载到总线。

通过查询手册，SPI1引脚使用如下：
- NSS----PA4
- SCK----PA5
- MISO----PA6
- MOSI----PA7

# 三、驱动中的函数

首先是总线的挂载，此处仅以SPI1为例。

```c
static int stm32_hw_spi_bus_init(void)
{
    struct stm32_spi *spi_bus;

    RCC_Configuration();
    GPIO_Configuration();

    spi_bus = &spi1;
    spi_bus->instance = SPI1;
    spi_bus->bus_name = "spi1";
    spi_bus->spi_bus.parent.user_data = &spi1;

    rt_spi_bus_register(&spi_bus->spi_bus, spi_bus->bus_name, &stm_spi_ops);
    LOG_D("%s bus init done", spi_bus->bus_name);

    return 0;
}
INIT_BOARD_EXPORT(stm32_hw_spi_bus_init);
```

逐个来看以下这一函数做了哪些事情。首先是RCC_Configuration。

```c
static void RCC_Configuration(void)
{
    RCC_AHB1PeriphClockCmd(RCC_AHB1Periph_GPIOA, ENABLE);
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_SPI1, ENABLE);
}
```

RCC_Configuration中使能了SPI1和GPIOA的时钟。

然后是GPIO_Configuration。

```c
static void GPIO_Configuration(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF;
    GPIO_InitStructure.GPIO_OType = GPIO_OType_PP;
    GPIO_InitStructure.GPIO_PuPd  = GPIO_PuPd_UP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_2MHz;

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_6 | GPIO_Pin_7;
     /* Connect alternate function */
    GPIO_PinAFConfig(GPIOA, GPIO_PinSource4, GPIO_AF_SPI1);
    GPIO_PinAFConfig(GPIOA, GPIO_PinSource5, GPIO_AF_SPI1);
    GPIO_PinAFConfig(GPIOA, GPIO_PinSource6, GPIO_AF_SPI1);
    GPIO_PinAFConfig(GPIOA, GPIO_PinSource7, GPIO_AF_SPI1);
    
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}
```

GPIO_Configuration中配置了SPI各个引脚的功能。

之后开始配置spi1，spi1由之前的stm32_spi结构体定义，然后调用rt_spi_bus_register，将spi1注册到内核。

注册时需要传入ops参数，ops是内核中SPI device的操作，因此在驱动中定义ops如下：

```c
static const struct rt_spi_ops stm_spi_ops =
{
    spi_configure,
    spixfer,
};
```

configure函数实现如下，设置指定的配置。

```c
static rt_err_t spi_configure(struct rt_spi_device *device,
                              struct rt_spi_configuration *configuration)
{
    RT_ASSERT(device != RT_NULL);
    RT_ASSERT(configuration != RT_NULL);

    struct stm32_spi *spi_drv =  rt_container_of(device->bus, struct stm32_spi, spi_bus);

    SPI_InitTypeDef *spi_init = &spi_drv->init;

    if (cfg->mode & RT_SPI_SLAVE)
    {
        spi_init->SPI_Mode = SPI_Mode_Slave;
    }
    else
    {
        spi_init->SPI_Mode = SPI_Mode_Master;
    }

    if (cfg->mode & RT_SPI_3WIRE)
    {
        spi_init->SPI_Direction = SPI_Direction_1Line_Rx;
    }
    else
    {
        spi_init->SPI_Direction = SPI_Direction_2Lines_FullDuplex;
    }

    if (cfg->data_width == 8)
    {
        spi_init->SPI_DataSize = SPI_DataSize_8b;
    }
    else if (cfg->data_width == 16)
    {
        spi_init->SPI_DataSize = SPI_DataSize_16b;
    }
    else
    {
        return RT_EIO;
    }

    if (cfg->mode & RT_SPI_CPHA)
    {
        spi_init->SPI_CPHA = SPI_CPHA_2Edge;
    }
    else
    {
        spi_init->SPI_CPHA = SPI_CPHA_1Edge;
    }

    if (cfg->mode & RT_SPI_CPOL)
    {
        spi_init->SPI_CPOL = SPI_CPOL_High;
    }
    else
    {
        spi_init->SPI_CPOL = SPI_CPOL_Low;
    }

    if (cfg->mode & RT_SPI_NO_CS)
    {
        spi_init->SPI_NSS = SPI_NSS_Soft;
    }
    else
    {
        spi_init->SPI_NSS = SPI_NSS_Soft;
    }

    uint32_t SPI_APB_CLOCK;

#if defined(SOC_SERIES_STM32F0) || defined(SOC_SERIES_STM32G0)
    SPI_APB_CLOCK = SystemCoreClock>>APBPrescTable[(RCC->CFGR & RCC_CFGR_PPRE1)>> 10U]
#else
    SPI_APB_CLOCK = SystemCoreClock>>APBPrescTable[(RCC->CFGR & RCC_CFGR_PPRE2)>> 13U];
#endif

    if (cfg->max_hz >= SPI_APB_CLOCK / 2)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_2;
    }
    else if (cfg->max_hz >= SPI_APB_CLOCK / 4)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_4;
    }
    else if (cfg->max_hz >= SPI_APB_CLOCK / 8)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_8;
    }
    else if (cfg->max_hz >= SPI_APB_CLOCK / 16)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_16;
    }
    else if (cfg->max_hz >= SPI_APB_CLOCK / 32)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_32;
    }
    else if (cfg->max_hz >= SPI_APB_CLOCK / 64)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_64;
    }
    else if (cfg->max_hz >= SPI_APB_CLOCK / 128)
    {
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_128;
    }
    else
    {
        /*  min prescaler 256 */
        spi_init->SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_256;
    }
    RCC_ClocksTypeDef RCC_ClocksStatus;
    RCC_GetClocksFreq(&RCC_ClocksStatus);

    LOG_D("sys freq: %d, pclk2 freq: %d, SPI limiting freq: %d, SPI_BaudRatePrescaler: %d",
          RCC_ClocksStatus.SYSCLK_Frequency,
          SPI_APB_CLOCK,
          cfg->max_hz,
          spi_init->SPI_BaudRatePrescaler);

    if (cfg->mode & RT_SPI_MSB)
    {
        spi_init->SPI_FirstBit = SPI_FirstBit_MSB;
    }
    else
    {
        spi_init->SPI_FirstBit = SPI_FirstBit_LSB;
    }

    spi_init->SPI_CRCPolynomial = 7;

#if defined(SOC_SERIES_STM32L4) || defined(SOC_SERIES_STM32G0)
    spi_init->NSSPMode          = SPI_NSS_PULSE_DISABLE;
#endif

    SPI_Init(spi_drv->instance, spi_init);

    SPI_Cmd(spi_drv->instance, ENABLE);

#if defined(SOC_SERIES_STM32L4) || defined(SOC_SERIES_STM32F0) \
        || defined(SOC_SERIES_STM32F7) || defined(SOC_SERIES_STM32G0)
    SET_BIT(spi_init->instance->CR2, SPI_RXFIFO_THRESHOLD_HF);
#endif

    spi_drv->instance->CR1 |= SPI_CR1_SPE;

    LOG_D("%s init done", spi_drv->bus_name);
    return RT_EOK;
}
```

xfer函数如下:

```c
static rt_uint32_t spixfer(struct rt_spi_device *device, struct rt_spi_message *message)
{
    int state;
    rt_size_t message_length, already_send_length;
    rt_uint16_t send_length;
    rt_uint8_t *recv_buf;
    const rt_uint8_t *send_buf;

    RT_ASSERT(device != RT_NULL);
    RT_ASSERT(device->bus != RT_NULL);
    RT_ASSERT(device->bus->parent.user_data != RT_NULL);
    RT_ASSERT(message != RT_NULL);

    struct stm32_spi *spi_drv =  rt_container_of(device->bus, struct stm32_spi, spi_bus);
    SPI_TypeDef *spi_instance = spi_drv->instance;
    SPI_InitTypeDef *spi_init = &spi_drv->init;
    struct stm32_hw_spi_cs *cs = device->parent.user_data;

    if (message->cs_take)
    {
        GPIO_WriteBit(cs->GPIOx, cs->GPIO_Pin, Bit_RESET);
    }

    LOG_D("%s transfer prepare and start", spi_drv->bus_name);
    LOG_D("%s sendbuf: %X, recvbuf: %X, length: %d",
          spi_drv->bus_name,
          (uint32_t)message->send_buf,
          (uint32_t)message->recv_buf, message->length);

    message_length = message->length;
    recv_buf = message->recv_buf;
    send_buf = message->send_buf;
    while (message_length)
    {
        if (message_length > 65535)
        {
            send_length = 65535;
            message_length = message_length - 65535;
        }
        else
        {
            send_length = message_length;
            message_length = 0;
        }

        /* calculate the start address */
        already_send_length = message->length - send_length - message_length;
        send_buf = (rt_uint8_t *)message->send_buf + already_send_length;
        recv_buf = (rt_uint8_t *)message->recv_buf + already_send_length;
        
        if (message->send_buf && message->recv_buf)
        {
        	state = spi_transmitreceive(*spi_init, spi_instance, (uint8_t *)send_buf, (uint8_t *)recv_buf, send_length, 1000);
        }
        else if (message->send_buf)
        {
            state = spi_transmit(*spi_init, spi_instance, (uint8_t *)send_buf, send_length, 1000);
        }
        else
        {
            memset((uint8_t *)recv_buf, 0xff, send_length);
            state = spi_receive(*spi_init, spi_instance, (uint8_t *)recv_buf, send_length, 1000);
        }

        if (state != 0)
        {
            LOG_I("spi transfer error : %d", state);
            message->length = 0;
        }
        else
        {
            LOG_D("%s transfer done", spi_drv->bus_name);
        }
    }

    if (message->cs_release)
    {
        GPIO_WriteBit(cs->GPIOx, cs->GPIO_Pin, Bit_SET);
    }

    return message->length;
}
```

函数根据内核调用时的不同传输情况，需要实现三个传输函数，如下:

```c
int spi_transmit(SPI_InitTypeDef spi_init, SPI_TypeDef *spi_instance, uint8_t *pData, uint16_t Size, uint32_t Timeout)
{
  uint32_t tickstart = 0U;
  int errorcode = 0;
  __IO uint16_t              TxXferCount;
  __IO uint32_t SPITimeout;

  /* Init tickstart for timeout management*/
  tickstart = rt_tick_get() * 1000 / RT_TICK_PER_SECOND;

  if((pData == NULL ) || (Size == 0))
  {
    errorcode = 1;
    goto error;
  }

  TxXferCount = Size;

  /* Configure communication direction : 1Line */
  if(spi_init.SPI_Direction == SPI_Direction_1Line_Rx)
  {
    spi_instance->CR1 |= SPI_CR1_BIDIOE;
  }

  /* Check if the SPI is already enabled */
  if((spi_instance->CR1 & SPI_CR1_SPE) != SPI_CR1_SPE)
  {
    /* Enable SPI peripheral */
    spi_instance->CR1 |=  SPI_CR1_SPE;
  }

  /* Transmit data in 16 Bit mode */
  if(spi_init.SPI_DataSize == SPI_DataSize_16b)
  {
    if((spi_init.SPI_Mode == SPI_Mode_Slave) || (TxXferCount == 0x01))
    {
      spi_instance->DR = *((uint16_t *)pData);

      pData += sizeof(uint16_t);
      TxXferCount--;
    }
    /* Transmit data in 16 Bit mode */
    while (TxXferCount > 0U)
    {
      SPITimeout = 0x1000;
      /* Wait until TXE flag is set to send data */
      while (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_TXE) == RESET)
      {
        if((SPITimeout--) == 0)
        {
          rt_kprintf("spi超时\n");
          errorcode = 3;
          goto error;
        }
      }
      /* 写入数据寄存器，把要写入的数据写入发送缓冲区 */
      spi_instance->DR = *((uint16_t *)pData);

      pData += sizeof(uint16_t);
      TxXferCount--;
    }
  }
  /* Transmit data in 8 Bit mode */
  else
  {
    if((spi_init.SPI_Mode == SPI_Mode_Slave) || (TxXferCount == 0x01U))
    {
      *((__IO uint8_t*)&spi_instance->DR) = (*pData);

      pData += sizeof(uint8_t);
      TxXferCount--;
    }

    while(TxXferCount > 0)
    {
      SPITimeout = 0x1000;

      /* 等待发送缓冲区为空，TXE事件 */
      while (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_TXE) == RESET)
      {
        if((SPITimeout--) == 0)
        {
          rt_kprintf("spi超时\n");
          errorcode = 3;
          goto error;
        }
      }

      /* 写入数据寄存器，把要写入的数据写入发送缓冲区 */
      *((__IO uint8_t*)&spi_instance->DR) = (*pData);

      pData += sizeof(uint8_t);
      TxXferCount--;
    }
}

  /* Wait until TXE flag */
  if(spi_wait_until_timeout(spi_init, spi_instance, SPI_FLAG_TXE, SET, Timeout, tickstart) != 0)
  {
    errorcode = 3;
    goto error;
  }
  
  /* Check Busy flag */
  if(spi_wait_until_timeout(spi_init, spi_instance, SPI_FLAG_TXE, SET, Timeout, tickstart) != 0)
  {
    errorcode = 1;
    goto error;
  }

error:
  return errorcode;
}

int spi_transmitreceive(SPI_InitTypeDef spi_init, SPI_TypeDef *spi_instance, uint8_t *pTxData, uint8_t *pRxData, uint16_t Size, uint32_t Timeout)
{
  uint32_t tmp = 0U;

  uint32_t tickstart = 0U;
  /* Variable used to alternate Rx and Tx during transfer */
  uint32_t txallowed = 1U;
  int errorcode = 0;
  __IO uint16_t              TxXferCount;
  __IO uint16_t              RxXferCount;

  /* Init tickstart for timeout management*/
  tickstart = rt_tick_get() * 1000 / RT_TICK_PER_SECOND;
  
  tmp  = spi_init.SPI_Mode;
  
  if(!((tmp == SPI_Mode_Master) && (spi_init.SPI_Direction == SPI_Direction_2Lines_FullDuplex)))
  {
    errorcode = 2;
    goto error;
  }

  if((pTxData == NULL) || (pRxData == NULL) || (Size == 0))
  {
    errorcode = 1;
    goto error;
  }

  TxXferCount = Size;
  RxXferCount = Size;

  /* Check if the SPI is already enabled */
  if((spi_instance->CR1 &SPI_CR1_SPE) != SPI_CR1_SPE)
  {
    /* Enable SPI peripheral */
    spi_instance->CR1 |=  SPI_CR1_SPE;
  }

  /* Transmit and Receive data in 16 Bit mode */
  if(spi_init.SPI_DataSize == SPI_DataSize_16b)
  {
    if((spi_init.SPI_Mode == SPI_Mode_Slave) || (TxXferCount == 0x01U))
    {
      SPI_I2S_ReceiveData(spi_instance);
      spi_instance->DR = *((uint16_t *)pTxData);

      pTxData += sizeof(uint16_t);
      TxXferCount--;
    }
    while ((TxXferCount > 0U) || (RxXferCount > 0U))
    {
      /* Check TXE flag */
      if(txallowed && (TxXferCount > 0U) && (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_TXE) == SET))
      {
        SPI_I2S_ReceiveData(spi_instance);
        spi_instance->DR = *((uint16_t *)pTxData);
        pTxData += sizeof(uint16_t);
        TxXferCount--;
        /* Next Data is a reception (Rx). Tx not allowed */ 
        txallowed = 0U;
      }

      /* Check RXNE flag */
      if((RxXferCount > 0U) && (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_RXNE) == SET))
      {
        *((uint16_t *)pRxData) = spi_instance->DR;
        pRxData += sizeof(uint16_t);
        RxXferCount--;
        /* Next Data is a Transmission (Tx). Tx is allowed */ 
        txallowed = 1U;
      }
      if((Timeout != 0xFFFFFFFFU) && ((rt_tick_get() * 1000 / RT_TICK_PER_SECOND-tickstart) >=  Timeout))
      {
        errorcode = 3;
        goto error;
      }
    }
  }
  /* Transmit and Receive data in 8 Bit mode */
  else
  {
    if((spi_init.SPI_Mode == SPI_Mode_Slave) || (TxXferCount == 0x01U))
    {
      SPI_I2S_ReceiveData(spi_instance);
      *((__IO uint8_t*)&spi_instance->DR) = (*pTxData);

      pTxData += sizeof(uint8_t);
      TxXferCount--;
    }
    while((TxXferCount > 0U) || (RxXferCount > 0U))
    {
      /* 等待发送缓冲区为空，TXE事件 */
      if (txallowed && (TxXferCount > 0U) && (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_TXE) == SET))
      {
        /* 写入数据寄存器，把要写入的数据写入发送缓冲区 */
        SPI_I2S_ReceiveData(spi_instance);
        *((__IO uint8_t*)&spi_instance->DR) = (*pTxData);

        pTxData += sizeof(uint8_t);
        TxXferCount--;
        txallowed = 0U;
      }

      if ((RxXferCount > 0U) && SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_RXNE) == SET)
      {
        *(uint8_t *)pRxData = spi_instance->DR;

        pRxData += sizeof(uint8_t);
        RxXferCount--;
        txallowed = 1U;
      }
      if((Timeout != 0xFFFFFFFFU) && ((rt_tick_get() * 1000 / RT_TICK_PER_SECOND-tickstart) >=  Timeout))
      {
        errorcode = 3;
        goto error;
      }
    }
  }

    /* Wait until TXE flag */
    if(spi_wait_until_timeout(spi_init, spi_instance, SPI_FLAG_TXE, SET, Timeout, tickstart) != 0)
    {
        errorcode = 3;
        goto error;
    }

    /* Check Busy flag */
    if(spi_wait_until_timeout(spi_init, spi_instance, SPI_FLAG_BSY, RESET, Timeout, tickstart) != 0)
    {
        errorcode = 1;
        goto error;
    }
  
error :
    return errorcode;
}

int spi_receive(SPI_InitTypeDef spi_init, SPI_TypeDef *spi_instance, uint8_t *pData, uint16_t Size, uint32_t Timeout)
{
  uint32_t tickstart = 0U;
  int errorcode = 0;
  __IO uint16_t              RxXferCount;
  __IO uint32_t             SPITimeout;

  if((spi_init.SPI_Mode == SPI_Mode_Master) && (spi_init.SPI_Direction == SPI_Direction_2Lines_FullDuplex))
  {
     /* Call transmit-receive function to send Dummy data on Tx line and generate clock on CLK line */
    return spi_transmitreceive(spi_init, spi_instance,pData,pData,Size,Timeout);
  }

  /* Init tickstart for timeout management*/
  tickstart = rt_tick_get() * 1000 / RT_TICK_PER_SECOND;

  if((pData == NULL ) || (Size == 0))
  {
    errorcode = 1;
    goto error;
  }

  RxXferCount = Size;

  /* Configure communication direction: 1Line */
  if(spi_init.SPI_Direction == SPI_Direction_1Line_Rx)
  {
    spi_instance->CR1 &= (~SPI_CR1_BIDIOE);
  }

  /* Check if the SPI is already enabled */
  if((spi_instance->CR1 & SPI_CR1_SPE) != SPI_CR1_SPE)
  {
    /* Enable SPI peripheral */
    spi_instance->CR1 |=  SPI_CR1_SPE;
  }

    /* Receive data in 8 Bit mode */
  if(spi_init.SPI_DataSize == SPI_DataSize_8b)
  {
    /* Transfer loop */
    while(RxXferCount > 0U)
    {
        SPITimeout = 0x1000;

        /* 等待发送缓冲区为空，TXE事件 */
        while (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_RXNE) == RESET)
        {
          if((SPITimeout--) == 0)
          {
            rt_kprintf("spi超时\n");
            errorcode = 3;
            goto error;
          }
        }

        *(uint8_t *)pData = spi_instance->DR;

        pData += sizeof(uint8_t);
        RxXferCount--;
    }
  }
  else
  {
    /* Transfer loop */
    while(RxXferCount > 0U)
    {
      SPITimeout = 0x1000;
      /* Check the RXNE flag */
      while (SPI_I2S_GetFlagStatus(spi_instance, SPI_I2S_FLAG_RXNE) == RESET)
      {
        if((SPITimeout--) == 0)
          {
            rt_kprintf("spi超时\n");
            errorcode = 3;
            goto error;
          }
      }
      *((uint16_t*)pData) = spi_instance->DR;
      pData += sizeof(uint16_t);
      RxXferCount--;
    }
  }

  /* Check the end of the transaction */
  if((spi_init.SPI_Mode == SPI_Mode_Master)&&((spi_init.SPI_Direction == SPI_Direction_1Line_Rx)||(spi_init.SPI_Direction == SPI_Direction_2Lines_RxOnly)))
  {
    /* Disable SPI peripheral */
    spi_instance->CR1 &= (~SPI_CR1_SPE);
  }

error :
  return errorcode;
}
```

至此SPI驱动需要实现的功能基本已经完成，SPI总线在内核启动过程中会注册到内核。

当然还需要挂载SPI设备，如下:

```c
rt_err_t rt_hw_spi_device_attach(const char *bus_name, const char *device_name, GPIO_TypeDef *cs_gpiox, uint16_t cs_gpio_pin)
{
    RT_ASSERT(bus_name != RT_NULL);
    RT_ASSERT(device_name != RT_NULL);

    rt_err_t result;
    struct rt_spi_device *spi_device;
    struct stm32_hw_spi_cs *cs_pin;

    /* initialize the cs pin && select the slave*/
    GPIO_InitTypeDef GPIO_Initure;
    GPIO_Initure.GPIO_Pin = cs_gpio_pin;
    GPIO_Initure.GPIO_Mode = GPIO_Mode_OUT;
    GPIO_Initure.GPIO_PuPd = GPIO_PuPd_UP;
    GPIO_Initure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(cs_gpiox, &GPIO_Initure);
    GPIO_WriteBit(cs_gpiox, cs_gpio_pin, Bit_SET);

    /* attach the device to spi bus*/
    spi_device = (struct rt_spi_device *)rt_malloc(sizeof(struct rt_spi_device));
    RT_ASSERT(spi_device != RT_NULL);
    cs_pin = (struct stm32_hw_spi_cs *)rt_malloc(sizeof(struct stm32_hw_spi_cs));
    RT_ASSERT(cs_pin != RT_NULL);
    cs_pin->GPIOx = cs_gpiox;
    cs_pin->GPIO_Pin = cs_gpio_pin;
    result = rt_spi_bus_attach_device(spi_device, device_name, bus_name, (void *)cs_pin);

    if (result != RT_EOK)
    {
        LOG_E("%s attach to %s faild, %d\n", device_name, bus_name, result);
    }

    RT_ASSERT(result == RT_EOK);

    LOG_D("%s attach to %s done", device_name, bus_name);

    return result;
}
```

现在SPI设备就可以使用了。