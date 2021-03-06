# Cullinan项目简介
## 特性
* 360度全方位扫描
* 15cm~6m范围内扫描测距
* 1800Hz扫描频率
* 加速度陀螺仪六轴运动数据输出
## 概述
Cullian是由石头科技自主研发的LDS镭射扫描测距系统。该系统由LDS测距系统和IMU惯性测量单元两大部分组成。LDS测距系统可以对360° 15cm~6m范围内环境进行扫描测距。扫描一圈采样360个点，每个点对应约1°，扫描频率为5Hz。IMU惯性测量单元可以实现对加速度和角速度等运行数据的实时获取。测量数据通过串口与外部应用程序进行交互。
## LDS测距系统
### 规格需求

参数|数值|单位
----|----|----
Distance Rang (D)|0.15~6|Meter (m)
Angular Range (A)|0~360|Degree
Angular Resolution (Ar)|约1|Degree
Sample Duration|0.5|Millisecond (ms)
Sample Frequency|1800|Hz
Scan Rate|5|Hz

### LDS串口数据
#### 串口通信格式
LDS测距数据通过串口发送给外部应用程序，波特率为115200，8个数据位，1个停止位，无奇偶校验位，无硬件流控，且只发送无接收。
#### 串口数据格式
LDS测距系统向外部接口发送两种数据，一种是属性信息，另一种是测距信息。属性信息仅在LDS旋转之前发送，且以间隔500ms循环发送。当LDS开始旋转之后，停止发送属性信息，转而发送测距信息。
#### 属性信息格式

内容|描述|Size / byte|说明
----|----|-----------|----
LDS_INFO_START|同步字符|1|固定为0xAA，发送的时候需要对属性包体里的0xAA进行转义，0xAA用0xA901代替，0xA9用0xA900代替。注：先计算checksum，之后再进行转义操作！
PACKET_SIZE|属性包体大小|2|除去LDS_INFO_START的部分，84 bytes
SOFTWARE_VERSION|LDS固件版本号|2|<主版本号4位 15:12 bit><次版本号12位 11:0 bit> 例：0x1001表示 V1.1
UID_SIZE|UID大小|2|值为18
UID|UID内容|18|
RSVD|系统保留|58|
CHECK_SUM|属性包校验和|2|计算方法详见[属性信息校验和计算方法](#属性信息校验和计算方法)

##### 属性信息校验和计算方法
```` c
uint16_t checksum = 0;
 for(i = 0; i < size; i++){ // size为属性包不包含checksum部分的word大小，为41
  checksum += *(lds_info_data + i);  // lds_info_data 为属性包的uint16_t*指针
 }
````
#### 测距信息格式
测距数据包不进行转义操作。

内容|描述|Size / byte|说明
----|----|----|----
Start|同步字符|1|固定为 0xFA
Index|数据包索引|1|表示在90个数据包里的index，从0xA0到0xF9，每个数据包携带4个采样点测距数据，每圈共360个采样点
Speed|转速（转/分钟）的64倍|2|小端序，前10 bits表示整数部分，后6 bits表示小数部分
Data 0|第1组测距数据|4|数据格式详见[采样点数据格式](#采样点数据格式)
Data 1|第2组测距数据|4|数据格式详见[采样点数据格式](#采样点数据格式)
Data 2|第3组测距数据|4|数据格式详见[采样点数据格式](#采样点数据格式)
Data 3|第4组测距数据|4|数据格式详见[采样点数据格式](#采样点数据格式)
Checksum|校验和|2|计算方法详见[测距信息校验和计算方法](#测距信息校验和计算方法)

##### 采样点数据格式
字节偏移量|描述
----|----
0|距离[7:0]
1|bit7 = 数据无效flag位，bit6 = 强度告警flag位，bit[5:0] = 距离[13:8]
2|强度[7:0]
3|强度[15:8]

##### 测距信息校验和计算方法
```` c
uint32_t i, checksum = 0;

for (i = 0; i < 20; i += 2)
{
    checksum = (checksum << 1) + *(uint16_t *)&lds_buf[i];
}
checksum = (checksum + (checksum >> 15)) & 0x7FFF;
````
## IMU惯性测量单元
### 简介
IMU惯性测量单元采用Bosch Sensortec公司最新推出的BMI160惯性测量单元，该芯片将最顶尖的16位3轴超低重力加速度计和超低功耗3轴陀螺仪集成于单一封装。BMI160采用14管脚LGA封装，尺寸为2.5×3.0×0.8mm<sup>3</sup>。当加速度计和陀螺仪在全速模式下运行时，耗电典型值低至950µA，仅为市场上同类产品耗电量的50%或者更低。

## BMI160 API
### 寄存器操作API
#### BMI160_ReadReg
原型：````int32_t BMI160_ReadReg(uint8_t RegAddr, uint8_t *RegData)````  
功能：读寄存器  
参数：````RegAddr````- 寄存器地址  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````RegData````- 寄存器数据指针  
返回：````STATE_OK````- 正常结束  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````STATE_PENDING````- 正在执行  
#### BMI160_ReadRegSeq
原型：````int32_t BMI160_ReadRegSeq(uint8_t RegAddr, uint8_t *RegData, uint8_t len)````  
功能：读多个寄存器  
参数：````RegAddr````- 寄存器首地址  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````RegData````- 寄存器数据缓冲区指针  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````len````- 读寄存器个数  
返回：````STATE_OK````- 正常结束  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````STATE_PENDING````- 正在执行  
#### BMI160_WriteReg
原型：````int32_t BMI160_WriteReg(uint8_t RegAddr, uint8_t RegData)````  
功能：写寄存器  
参数：````RegAddr````- 寄存器地址  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````RegData````- 寄存器数据  
返回：````STATE_OK````- 正常结束  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````STATE_PENDING````- 正在执行  
#### BMI160_WriteRegSeq
原型：````int32_t BMI160_WriteRegSeq(uint8_t RegAddr, uint8_t *RegData, uint8_t len)````  
功能：写多个寄存器  
参数：````RegAddr````- 寄存器首地址  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````RegData````- 寄存器数据缓冲区指针  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````len````- 写寄存器个数  
返回：````STATE_OK````- 正常结束  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````STATE_PENDING````- 正在执行  
### BMI160配置API
#### BMI160_SetAccelNormal
原型：````int32_t BMI160_SetAccelNormal(void)````  
功能：配置加速度传感器为正常供电模式  
参数：无  
返回：````STATE_OK````- 正常结束  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````STATE_PENDING````- 正在执行  
#### BMI160_SetGyroNormal
原型：````int32_t BMI160_SetGyroNormal(void)````  
功能：配置陀螺仪传感器为正常供电模式  
参数：无  
返回：````STATE_OK````- 正常结束  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;````STATE_PENDING````- 正在执行  
### BMI160 API示例
```` c
    /* Select the Output data rate, range of accelerometer sensor */
    BMI160_REG.ACC_CONF.acc_odr = BMI160_ACCEL_ODR_100HZ;
    BMI160_REG.ACC_CONF.acc_bwp = BMI160_ACCEL_BW_NORMAL_AVG4;
    BMI160_REG.ACC_CONF.acc_us = 0;
    BMI160_REG.ACC_RANGE.acc_range = BMI160_ACCEL_RANGE_2G;
    while (BMI160_WriteRegSeq(REG(ACC_CONF), (uint8_t *)&BMI160_REG.ACC_CONF, 2) == STATE_PENDING) ;
    while (BMI160_SetAccelNormal() == STATE_PENDING) ; // set normal power mode
    
    /* Select the Output data rate, range of Gyroscope sensor */
    BMI160_REG.GYR_CONF.gyr_odr = BMI160_GYRO_ODR_100HZ;
    BMI160_REG.GYR_CONF.gyr_bwp = BMI160_GYRO_BW_NORMAL_MODE;
    BMI160_REG.GYR_RANGE.gyr_range = BMI160_GYRO_RANGE_2000_DPS;
    while (BMI160_WriteRegSeq(REG(GYR_CONF), (uint8_t *)&BMI160_REG.GYR_CONF, 2) == STATE_PENDING) ;
    while (BMI160_SetGyroNormal() == STATE_PENDING) ; // set normal power mode
````
### 惯性测量数据接口
#### 惯性测量数据接口串口通信格式
惯性测量单元通过USB虚拟串口将测量原始数据发送给外部应用程序，波特率为115200，8个数据位，1个停止位，无奇偶校验位，无硬件流控，且只发送无接收。
#### 串口数据格式
偏移量|描述|Size / byte|说明
------|----|-----------|----
0|GYR_X[7:0]|1|陀螺仪X轴数据低字节
1|GYR_X[15:8]|1|陀螺仪X轴数据高字节
2|GYR_Y[7:0]|1|陀螺仪Y轴数据低字节
3|GYR_Y[15:8]|1|陀螺仪Y轴数据高字节
4|GYR_Z[7:0]|1|陀螺仪Z轴数据低字节
5|GYR_Z[15:8]|1|陀螺仪Z轴数据高字节
6|ACC_X[7:0]|1|加速度X轴数据低字节
7|ACC_X[15:8]|1|加速度X轴数据高字节
8|ACC_Y[7:0]|1|加速度Y轴数据低字节
9|ACC_Y[15:8]|1|加速度Y轴数据高字节
10|ACC_Z[7:0]|1|加速度Z轴数据低字节
11|ACC_Z[15:8]|1|加速度Z轴数据高字节
12|SENSORTIME[7:0]|1|时间戳[7:0]
13|SENSORTIME[15:8]|1|时间戳[15:8]
14|SENSORTIME[23:16]|1|时间戳[23:16]

## 调试命令接口
### 调试命令串口通信格式
调试命令接口通过串口接收用户输入的命令。串口波特率为115200，8个数据位，1个停止位，无奇偶校验位，无硬件流控。
### BMI160操作命令
#### rd160
格式：  
````rd160 reg_addr````  
描述：从寄存器地址````reg_addr````读取一个寄存器并显示。  
示例：
````
rd160 40
40=28
````
#### wr160
格式：  
````wr160 reg_addr reg_data````  
描述：向寄存器地址````reg_addr````写入数据````reg_data````并回读。  
示例：
````
wr160 40 29
read back 40=29
````
#### dump160
格式：  
````dump160````  
描述：将128个寄存器全部读出并显示。  
示例：
````
dump160
     0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
[00]:D1 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[10]:00 00 00 00 00 00 00 00 6F 77 2C 10 00 00 00 00
[20]:00 80 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[30]:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[40]:29 03 28 00 0B 88 80 10 00 00 00 20 80 42 4C 00
[50]:00 00 00 00 00 00 00 00 00 00 07 30 81 0B C0 00
[60]:14 14 24 04 0A 18 48 08 11 00 00 00 00 00 00 00
[70]:00 00 00 00 51 96 09 00 00 00 15 03 00 00 00 00
````
#### set160
格式：  
````set160````  
描述：按下表配置传感器。

参数 | 数值
---- |-----
中断管脚INT1有效电平|1(高电平)
中断管脚INT1 I/O输出模式|0(push-pull)
中断管脚INT1输出使能|1(使能)
数据准备好中断映射到中断管脚INT1|1(是)
加速度传感器数据输出速率|BMI160_ACCEL_ODR_100HZ
加速度传感器带宽|BMI160_ACCEL_BW_NORMAL_AVG4
加速度传感器欠采样|0(关闭)
加速度传感器量程|BMI160_ACCEL_RANGE_2G
加速度传感器电源模式|BMI160_ACCEL_NORMAL_MODE
陀螺仪传感器数据输出速率|BMI160_GYRO_ODR_100HZ
陀螺仪传感器带宽|BMI160_GYRO_BW_NORMAL_MODE
陀螺仪传感器量程|BMI160_GYRO_RANGE_2000_DPS
陀螺仪传感器电源模式|BMI160_GYRO_NORMAL_MODE

示例：  
````set160````
#### data160
格式：  
````data160````  
描述：循环查询````STATUS````寄存器中的````drdy_acc````，当发现置位时，读出加速度传感器三轴数据、陀螺仪传感器三轴数据和时间戳数据并显示。  
示例：
````
data160
ACC.x= -0.056152 ACC.y=  0.035828 ACC.z=  1.008545 GYR.x= -0.121951 GYR.y= -0.426829 GYR.z= -0.182927 @  7277825
ACC.x= -0.055176 ACC.y=  0.036072 ACC.z=  1.007935 GYR.x= -0.182927 GYR.y= -0.426829 GYR.z= -0.182927 @  7278081
ACC.x= -0.056335 ACC.y=  0.032349 ACC.z=  1.007690 GYR.x= -0.121951 GYR.y= -0.426829 GYR.z= -0.182927 @  7278337
......
````
#### temp160
格式：  
````temp160````  
描述：循环读出温度数据并显示。  
示例：
````
temp160
T= 33.335938
T= 33.314453
T= 33.373047
......
````
## Copyright © 北京石头世纪科技有限公司
