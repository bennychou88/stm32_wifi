>根据OneNet平台给的代码改的，纯新手，慢慢学。。

使用的器件：

 - STM32F103CBT6 单片机
 - ESP8266-01S wifi模块
 - SHT20温湿度

使用的服务端：

 - Node.js
 - net模块

#一、单片机相关代码(只提供网络相关代码)
##1.esp8266相关配置代码
###1）esp8266.h(宏定义一些AT指令)
```
#define AT          "AT\r\n"	//测试esp8266是否工作正常
#define CWMODE      "AT+CWMODE=3\r\n"		//设置wifi模块的模式，3为STA+AP模式
#define RST         "AT+RST\r\n"        //重启
#define CIFSR       "AT+CIFSR\r\n"      //查看ip
#define CWJAP       "AT+CWJAP=\"nb\",\"123456789\"\r\n"	// 设置要连接的无线路由器的 ssid 和 password
#define CIPSTART    "AT+CIPSTART=\"TCP\",\"119.29.201.31\",4001\r\n"		//设置服务器IP和端口
#define CIPMODE0    "AT+CIPMODE=0\r\n"		//非透传模式
#define CIPMODE1    "AT+CIPMODE=1\r\n"		//透传模式
#define CIPSEND     "AT+CIPSEND\r\n"        //发送
#define CIPSTATUS   "AT+CIPSTATUS\r\n"		//网络状态查询

extern void ESP8266_Rst(void);  
extern void ESP8266_Init(void);

```

###2）esp8266.c(两个配置函数)
```
/**
  * @brief  ESP8266硬件复位
**/
void ESP8266_Rst(void)
{
	GPIO_InitTypeDef GPIO_InitStructure;
    //Pb5--对应ESP8266的reset引脚;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOB, &GPIO_InitStructure);
	
	GPIO_ResetBits(GPIOB,GPIO_Pin_5);
	mDelay(100);
	GPIO_SetBits(GPIOB,GPIO_Pin_5);
	mDelay(100);
}


/*
 *  @brief ESP8266模块初始化
 */
void ESP8266_Init(void)
{
	ESP8266_Rst();
#if 1
    SendCmd(AT, "OK", 1000);		  //模块有效性检查
    SendCmd(CWMODE, "OK", 1000);	//模块工作模式
    SendCmd(RST, "OK", 2000);		  //模块重置
    SendCmd(CIFSR, "OK", 1000);		//查询网络信息
    SendCmd(CWJAP, "OK", 2000);		//配置需要连接的WIFI热点SSID和密码
    SendCmd(CIPSTART, "OK", 2000);	//TCP连接
    SendCmd(CIPMODE1, "OK", 1000);	//配置透传模式
	SendCmd(CIPSEND, ">", 1000);	//进入透传模式
	USART2_Clear();
#endif
}
```

###3)usart2.c中的SendCmd函数（USART2串口发送AT命令使用）
```
void SendCmd(char* cmd, char* result, int timeOut)
{
    while(1)
    {
        USART2_Clear();
        USART2_Write(USART2, (unsigned char *)cmd, strlen((const char *)cmd));
        mDelay(timeOut);
        printf("%s %d cmd:%s,rsp:%s\n", __func__, __LINE__, cmd, usart2_rcv_buf);
        if((NULL != strstr((const char *)usart2_rcv_buf, result)))	//判断是否有预期的结果
        {
            break;
        }
        else
        {
            mDelay(100);
        }
    }
}
```

##2.拼接发送字符串代码
###1）获取温湿度(sht20代码就不放出来了)
```
        /* 获取温湿度 */
        SHT2x_MeasureHM(SHT20_Measurement_T_HM, &temp);
        mDelay(500);
        SHT2x_MeasureHM(SHT20_Measurement_RH_HM, &humi);

        /* 转化为字符串形式 */
        sprintf(tempStr, "%d", temp);
        sprintf(humiStr, "%d", humi);
        
```

###2）拼接代码（拼接成json格式）
```
/**
  * @brief   拼接
  * @param   pkt   缓存指针
  *	@param 	 dsid1  数据流1ID
  *	@param 	 val1   字符串形式的数据点1的值
  *	@param 	 dsid2  数据流2ID
  *	@param 	 val2   字符串形式的数据点2的值
  * @retval  整个包的长度
  */
uint32_t HTTP_NewPostPacket(char *pkt, char *dsid1, char *val1, char *dsid2, char *val2)
{
    char dataBuf[50] = {0};
    char lenBuf[10] = {0};
    *pkt = 0;
    sprintf(dataBuf, "{\"%s\":\"%s\",\"%s\":\"%s\"}", dsid1, val1, dsid2, val2);
    sprintf(lenBuf, "%d", strlen(dataBuf));

    strcat(pkt, dataBuf);

    return strlen(pkt);
}

```

#二、Node.js相关代码
##1.TCP服务端代码
```
var net = require('net')
net.createServer(function(socket){
	socket.on('data',function(data){//接受处理
		console.log('got:',data.toString());//打印接收到的数据
		socket.write('I have received') //接受后回复
	});
	socket.write('Ready to receive your message!')//客户端连接成功后回复
}).listen(4001);
```

#三、结果截图
##1.服务端打印的信息截图
![服务端打印的信息截图](https://leanote.com/api/file/getImage?fileId=58cb384aab64417ac3007b02)

##2.客户端串口输出的信息截图
![客户端串口输出的信息截图](https://leanote.com/api/file/getImage?fileId=58cb384aab64417ac3007b01)


@治电小白菜 20170317