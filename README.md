# Quad SPI

> 希望能通过这个基于强大功能和丰富的生态的MCU进而任意和Flash交互
>
> This project is based on STM32WB55 Nucleo Pack.
>
> Flash:W25Q64 WinBond

##### Import Structure

###### QSPI_CommandTypeDef

```c
typedef struct
{
  uint32_t Instruction;        /* Specifies the Instruction to be sent
                                  This parameter can be a value (8-bit) between 0x00 and 0xFF */
  uint32_t Address;            /* Specifies the Address to be sent (Size from 1 to 4 bytes according AddressSize)
                                  This parameter can be a value (32-bits) between 0x0 and 0xFFFFFFFF */
  uint32_t AlternateBytes;     /* Specifies the Alternate Bytes to be sent (Size from 1 to 4 bytes according AlternateBytesSize)
                                  This parameter can be a value (32-bits) between 0x0 and 0xFFFFFFFF */
  uint32_t AddressSize;        /* Specifies the Address Size
                                  This parameter can be a value of @ref QSPI_AddressSize */
  uint32_t AlternateBytesSize; /* Specifies the Alternate Bytes Size
                                  This parameter can be a value of @ref QSPI_AlternateBytesSize */
  uint32_t DummyCycles;        /* Specifies the Number of Dummy Cycles.
                                  This parameter can be a number between 0 and 31 */
  uint32_t InstructionMode;    /* Specifies the Instruction Mode
                                  This parameter can be a value of @ref QSPI_InstructionMode */
  uint32_t AddressMode;        /* Specifies the Address Mode
                                  This parameter can be a value of @ref QSPI_AddressMode */
  uint32_t AlternateByteMode;  /* Specifies the Alternate Bytes Mode
                                  This parameter can be a value of @ref QSPI_AlternateBytesMode */
  uint32_t DataMode;           /* Specifies the Data Mode (used for dummy cycles and data phases)
                                  This parameter can be a value of @ref QSPI_DataMode */
  uint32_t NbData;             /* Specifies the number of data to transfer. (This is the number of bytes)
                                  This parameter can be any value between 0 and 0xFFFFFFFF (0 means undefined length
                                  until end of memory)*/
  uint32_t DdrMode;            /* Specifies the double data rate mode for address, alternate byte and data phase
                                  This parameter can be a value of @ref QSPI_DdrMode */
  uint32_t SIOOMode;           /* Specifies the send instruction only once mode
                                  This parameter can be a value of @ref QSPI_SIOOMode */
}QSPI_CommandTypeDef;

/* @defgroup QSPI_InstructionMode QSPI Instruction Mode@ */
#define QSPI_INSTRUCTION_NONE          0x00000000U                     /*!<No instruction*/
#define QSPI_INSTRUCTION_1_LINE        ((uint32_t)QUADSPI_CCR_IMODE_0) /*!<Instruction on a single line*/
#define QSPI_INSTRUCTION_2_LINES       ((uint32_t)QUADSPI_CCR_IMODE_1) /*!<Instruction on two lines*/
#define QSPI_INSTRUCTION_4_LINES       ((uint32_t)QUADSPI_CCR_IMODE)   /*!<Instruction on four lines*/

/* @defgroup QSPI_AddressSize QSPI Address Size@ */
#define QSPI_ADDRESS_8_BITS            0x00000000U                      /*!<8-bit address*/
#define QSPI_ADDRESS_16_BITS           ((uint32_t)QUADSPI_CCR_ADSIZE_0) /*!<16-bit address*/
#define QSPI_ADDRESS_24_BITS           ((uint32_t)QUADSPI_CCR_ADSIZE_1) /*!<24-bit address*/
#define QSPI_ADDRESS_32_BITS           ((uint32_t)QUADSPI_CCR_ADSIZE)   /*!<32-bit address*/

/* @defgroup QSPI_SIOOMode QSPI Send Instruction Mode@ */
#define QSPI_SIOO_INST_EVERY_CMD       0x00000000U                  /*!<Send instruction on every transaction*/
#define QSPI_SIOO_INST_ONLY_FIRST_CMD  ((uint32_t)QUADSPI_CCR_SIOO) /*!<Send instruction only for the first command*/
```

###### QSPI_AutoPollingTypeDef

```c
typedef struct
{
  uint32_t Match;              /* Specifies the value to be compared with the masked status register to get a match.
                                  This parameter can be any value between 0 and 0xFFFFFFFF */
  uint32_t Mask;               /* Specifies the mask to be applied to the status bytes received.
                                  This parameter can be any value between 0 and 0xFFFFFFFF */
  uint32_t Interval;           /* Specifies the number of clock cycles between two read during automatic polling phases.
                                  This parameter can be any value between 0 and 0xFFFF */
  uint32_t StatusBytesSize;    /* Specifies the size of the status bytes received.
                                  This parameter can be any value between 1 and 4 */
  uint32_t MatchMode;          /* Specifies the method used for determining a match.
                                  This parameter can be a value of @ref QSPI_MatchMode */
  uint32_t AutomaticStop;      /* Specifies if automatic polling is stopped after a match.
                                  This parameter can be a value of @ref QSPI_AutomaticStop */
} QSPI_AutoPollingTypeDef;
```

##### Order

Mode first, data second.Let's see the APIs.

###### HAL_QSPI_Command

```c
/**
  * @brief Set the command configuration.
  * @param hqspi QSPI handle
  * @param cmd : structure that contains the command configuration information
  * @param Timeout Timeout duration
  * @note   This function is used only in Indirect Read or Write Modes
  * @retval HAL status
  */
HAL_StatusTypeDef HAL_QSPI_Command(QSPI_HandleTypeDef *hqspi, QSPI_CommandTypeDef *cmd, uint32_t Timeout)
{
    QSPI_Config(hqspi, cmd, QSPI_FUNCTIONAL_MODE_INDIRECT_WRITE);
    
    if (cmd->DataMode == QSPI_DATA_NONE)
    {
        /* When there is no data phase, the transfer start as soon as the configuration is done
            so wait until TC flag is set to go back in idle state */
        status = QSPI_WaitFlagStateUntilTimeout(hqspi, QSPI_FLAG_TC, SET, tickstart, Timeout);
        if (status == HAL_OK)
        {
          __HAL_QSPI_CLEAR_FLAG(hqspi, QSPI_FLAG_TC);
          /* Update QSPI state */
          hqspi->State = HAL_QSPI_STATE_READY;
        }
    }
    else
    {
      /* Update QSPI state */
      hqspi->State = HAL_QSPI_STATE_READY;
    }
}
```

###### QSPI_Config

```c
/**
  * @brief  Configure the communication registers.
  * @param  hqspi QSPI handle
  * @param  cmd structure that contains the command configuration information
  * @param  FunctionalMode functional mode to configured
  *           This parameter can be one of the following values:
  *            @arg QSPI_FUNCTIONAL_MODE_INDIRECT_WRITE: Indirect write mode
  *            @arg QSPI_FUNCTIONAL_MODE_INDIRECT_READ: Indirect read mode
  *            @arg QSPI_FUNCTIONAL_MODE_AUTO_POLLING: Automatic polling mode
  *            @arg QSPI_FUNCTIONAL_MODE_MEMORY_MAPPED: Memory-mapped mode
  * @retval None
  */
static void QSPI_Config(QSPI_HandleTypeDef *hqspi, QSPI_CommandTypeDef *cmd, uint32_t FunctionalMode)
{
  if (cmd->InstructionMode != QSPI_INSTRUCTION_NONE)
  {
    if (cmd->AlternateByteMode != QSPI_ALTERNATE_BYTES_NONE)
    {
      /* Configure QSPI: ABR register with alternate bytes value */
      WRITE_REG(hqspi->Instance->ABR, cmd->AlternateBytes);

      if (cmd->AddressMode != QSPI_ADDRESS_NONE)
      {
        /*---- Command with instruction, address and alternate bytes ----*/
        /* Configure QSPI: CCR register with all communications parameters */
        if (FunctionalMode != QSPI_FUNCTIONAL_MODE_MEMORY_MAPPED)
        {
          /* Configure QSPI: AR register with address value */
        }
      }
      else
      {
        /*---- Command with instruction and alternate bytes ----*/
        /* Configure QSPI: CCR register with all communications parameters */
      }
    }
    else
    {
      if (cmd->AddressMode != QSPI_ADDRESS_NONE)
      {
        /*---- Command with instruction and address ----*/
        /* Configure QSPI: CCR register with all communications parameters */W
        if (FunctionalMode != QSPI_FUNCTIONAL_MODE_MEMORY_MAPPED)
        {
          /* Configure QSPI: AR register with address value */
          WRITE_REG(hqspi->Instance->AR, cmd->Address);
        }
      }
      else
      {
        /*---- Command with only instruction ----*/
        /* Configure QSPI: CCR register with all communications parameters */
      }
    }
  }
}
```

###### PIN Description

| CONN（第几组排针） | PIN  | MEAN | GPIO | COLOR（杜邦线的颜色） |
| ------------------ | ---- | ---- | ---- | --------------------- |
| CN7                | 16   | 3.3V | NULL | 红                    |
| CN7                | 20   | GND  | NULL | 棕                    |
| CN10               | 35   | CS   | PA2  | 绿                    |
| CN10               | 37   | CLK  | PA3  | 白                    |
| CN10               | 5    | IO0  | PB9  | 紫                    |
| CN10               | 3    | IO1  | PB8  | 灰                    |
| CN10               | 15   | IO2  | PA7  | 蓝                    |
| CN10               | 13   | IO3  | PA6  | 橙                    |

### 测试步骤

#### 第一步：读取设备ID

首先我们应该确定一下我们的接线是否准确无误，查看数据手册获取如何读取：

![image-20220413100611425](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220413100611425.png)

我们可以非常清楚的看到设备上电后默认是SPI总线(accessed through an SPI compatiable)，此时HOLD(IO2)和WP(IO3)上暂时无法传输数据，如果启用引脚上拉，此时应该是低电平，我们尝试读取一下：

![image-20220413101003646](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220413101003646.png)

分析一下代码：

```c
/* 整体看下来，把数据封装在 QSPI_CommandTypeDef 就可以完成整个读写过程 */
QSPI_CommandTypeDef     sCommand;
/* Standard SPI:1 Dual SPI:2 Quad SPI:4 */
sCommand.InstructionMode   = QSPI_INSTRUCTION_1_LINE;
/* 发出的命令 */
sCommand.Instruction       = 0x90;
/* 24位地址正好是3个没用的波形 */
sCommand.AddressMode = QSPI_ADDRESS_1_LINE;
sCommand.AddressSize = QSPI_ADDRESS_24_BITS;
sCommand.Address = 0x0U;
/* 这个暂时不知道是什么意思 */
sCommand.AlternateByteMode = QSPI_ALTERNATE_BYTES_NONE;
sCommand.AlternateBytes = QSPI_ALTERNATE_BYTES_NONE;
sCommand.AlternateBytesSize = QSPI_ALTERNATE_BYTES_NONE;
/* DummyCycles为零，实际上dummy有3个字节的地址代替了，数据手册上让这么做的，dummy 波形多产生在准备读之前 */
/* The dummy clocks allow the devices internal circuits additional time for setting up the initial address. During the dummy clocks the data value on the DO pin is a “don’t care”. */
sCommand.DummyCycles = 0;
/* 标准SPI MISO接收数据，设置接收的数据长度为2，也就是发送完地址以后，在产生16个时钟信号(接收数据)接收从机数据 */
sCommand.DataMode = QSPI_DATA_1_LINE;
sCommand.NbData = 2;
/* Double Date Rate, 不采用上升沿、下降沿双倍速率采样 */
sCommand.DdrMode           = QSPI_DDR_MODE_DISABLE;
/* 
每次交互都要重新发送命令
因为在连续读的时候，SOC可以每次发送地址而忽略（不发）命令
*/
sCommand.SIOOMode          = QSPI_SIOO_INST_EVERY_CMD;

if (HAL_QSPI_Command(&hqspi, &sCommand, HAL_QSPI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
{
    Error_Handler();
}

if (HAL_QSPI_Receive(&hqspi, buf, HAL_QSPI_TIMEOUT_DEFAULT_VALUE) != HAL_OK) {
    Error_Handler();
}
```

##### 波形

- 这里记录一下波形，当我们读不出来的时候记录一下

手册上的波形

![image-20220414095948343](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414095948343.png)

由于现在是标准SPI模式，IO2和IO3是没有数据交互的，所以我们可以使用SPI解析

![image-20220414101907407](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414101907407.png)

![image-20220414102930640](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414102930640.png)

#### 第二步：切换到QuadSPI模式

 Quad SPI and QPI instructions require the non-volatile Quad Enable bit (**QE**) in Status Register-2 

to be set. 

![image-20220414104803868](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414104803868.png)

所以使用Quad SPI提高传输速度仅需要两步：

1. 使能QE位
2. 使用支持Quad SPI特征的指令

##### QE Bit Enable

非易失性存储写（掉电数据不丢，甚至状态寄存器也是这样，我们在MCU上的寄存器是一段特殊的内存，掉电后都是需要初始化的）之前必须要先发送写使能指令。

> The Write Status Register instruction allows the Status Register to be written. Only non-volatile Status Register bits SRP0, SEC, TB, BP2, BP1, BP0 (bits 7 thru 2 of Status Register-1) and CMP, LB3, LB2, LB1, **QE**, SRP1 (bits 14 thru 8 of Status Register-2) can be written to. To write non-volatile Status Register bits, a standard Write Enable (06h) instruction must previously have been executed for the device to accept the Write Status Register instruction (Status Register bit WEL must equal 1). 

![image-20220414134654959](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414134654959.png)

![image-20220414140817507](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414140817507.png)

```c
int W25QXXWriteEnable()
{
	QSPI_CommandTypeDef sCommand = {0};

	/* Enable write operations ------------------------------------------ */
	sCommand.InstructionMode   = QSPI_INSTRUCTION_1_LINE;
	sCommand.Instruction       = 0x06;

	sCommand.AddressMode = QSPI_ADDRESS_NONE;
	sCommand.AddressSize = QSPI_ADDRESS_NONE;
	sCommand.Address = 0x0U;

	sCommand.AlternateByteMode = QSPI_ALTERNATE_BYTES_NONE;
	sCommand.AlternateBytes = QSPI_ALTERNATE_BYTES_NONE;
	sCommand.AlternateBytesSize = QSPI_ALTERNATE_BYTES_NONE;
	/* 注意这里一定要设置DataMode为0 */
	sCommand.DummyCycles = 0;
	sCommand.DataMode = QSPI_DATA_NONE;
	sCommand.NbData = 0;

	sCommand.DdrMode = QSPI_DDR_MODE_DISABLE;
	sCommand.SIOOMode = QSPI_SIOO_INST_EVERY_CMD;

	if (HAL_QSPI_Command(&hqspi, &sCommand, HAL_QSPI_TIMEOUT_DEFAULT_VALUE) != HAL_OK)
	{
	  Error_Handler();
	}

	return 0;
}
```

现在终于可以置为 **QE Bit Enable** 了

![image-20220414141443634](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414141443634.png)

顺便记录一下示波器的

![image-20220414141746348](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414141746348.png)

Quad SPI Read

**Mode:Fast Read Quad Output**

到目前位置前边的都是铺垫，现在终于可以看看Quad SPI传输了

![image-20220414150710148](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414150710148.png)

##### Write And Read

上边的读不太严谨，但是也没有明显的读错误，接下来我们试试随机数读写。

![image-20220414151057356](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414151057356.png)

这个接口真的是为了读而设计的，写的话还得老老实实的用标准SPI.

![image-20220414154455103](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414154455103.png)

看一下完整的波形

![image-20220414154650573](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414154650573.png)

![image-20220414155252801](C:\Users\34776\AppData\Roaming\Typora\typora-user-images\image-20220414155252801.png)

##### Erase 、Write And Read

```c
while(1)
{
    ...
}
```

