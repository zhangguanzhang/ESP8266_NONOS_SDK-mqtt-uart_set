# ESP8266_NONOS_SDK-mqtt-uart_set
从乐鑫官方的non_os的mqtt的demo修改,我用的是esp-12系列，12F和12S测试了都可以用

官方的自带的demo好像串口编译不通过,改了一些引用文件和增加了几个定义就能过了

之前是来玩外网控制的,所以打算是用单片机的串口发送特定的格式帧给esp-8266,在8266的串口中断里检查帧格式正确就通过mqtt协议推送,大部分逻辑都是串口中断的回调函数里

增加了串口发送命令设置mqtt的配置,以回车换行结束

定义了串口缓冲区＜/br＞
uint8 uart_rx_buff[rev_len]={0};＜/br＞
#define rev_len 1024＜/br＞
extern uint8 uart_rx_buff[rev_len];＜/br＞
extern MQTT_Client mqttClient;＜/br＞
测试是一次mqtt可以推送1000个字节

实际打印可以得知串口的数据寄存器好像最大100还是108字节来着来着,也就是串口回调函数里下面这句＜/br＞
uint8 fifo_len = (READ_PERI_REG(UART_STATUS(UART0))>>UART_RXFIFO_CNT_S)&UART_RXFIFO_CNT;＜/br＞
其中的fifo_len在获取后每次打印都是一样的,刚开始发送超过100个字节的话就重复进入串口中断的回调函数＜/br＞
然后我以回车换行判断接收完成,没收到回车在下一次进来的时候接着上次的地方写入到串口缓冲区里,注释里写得很详细,这里就不截取代码说了

<hr>

### 先说下使用方法

一．	通讯说明和注意事项
    1.通信格式: (115200, 8, none, 1)波特率: 115200, 数据位:8, 无校验, 停止位:1＜/br＞
    2.实现功能：查看或者设置8266的参数,上传数据＜/br＞
    3.数据格式＜/br＞
    
二．	每次发送都要回车换行结束，代码逻辑回车换行结束接收和缓冲区的写入
指令有:
	show #查看信息
	restart #软重启8266

查看设定show
发送show查看设置的信息(不重启8266)

![Markdown](http://i2.kiimg.com/596163/53641c346618bd84.png)

设定协议格式为:
信息分为三部分: WIFI host mqtt
指令为set WIFI ssid passwd
      set host hostip port
	 set mqtt id username passwd 
假设路由器的wifi的ssid和密码为test  testpasswd，需要修改8266的连接信息，向8266的串口发送set WIFI test testpasswd即可

倘若只修改某一部分，即用户路由器修改了密码，但是ssid没改，可发送set WIFI null testpasswd
字符‘null’表示那部分不修改
例如只修改id为abcdefg,发送set mqtt abcdefg null null即可

设定部分三种情况下无论哪种发送修改帧过去均会从串口打印出各自部分的信息，设置后建议发送restart重启加载信息，如下

![Markdown](http://i2.kiimg.com/596163/ec63620e4d2a0775.png)

上传协议格式为:
除了设定发啥都是上传
上传字符最大限制是997个字符
上传会在发送的数据后面加上一个空格然后加上自身id(即mqtt的id)
!!!上传建议速度不要太快,否则mqtt的推送缓冲区满了会失败

![Markdown](http://i2.kiimg.com/596163/9c88fca2bc00a4a7.png)
 
id验证是在8266的内部进行的，上传会自动把内容末尾附加上自身的id，服务器端会根据id处理，与下位机无关

开发部分:
云端控制内容传来的时候8266是先提取消息里面的id对比自身的id，一致才串口打印出消息

![Markdown](http://i2.kiimg.com/596163/cbce7bb3040c0e39.png)

上图所示是两帧数据，第一帧是id与本身一致，即打印出消息内容,第二帧不符合自身id，即不打印内容,8266内部代码逻辑下图所示

![Markdown](http://i2.kiimg.com/596163/c2a975511ff6198b.png)

	单片机或者你的树莓派或者啥mcu的串口接8266的串口
处理云端的数据思路是以下这样:
举例
	1假如连接不上wifi就是STATION_IDLE多次串口打印
	2心跳包(Send keepalive packet to字样)的打印
所有动作和状态都会从串口输出,内容里包含多个回车换行符，所以建议串口中断不要以回车换行判定接收完成
建议开启一个定时器
串口中断里使能定时器并且接受每一个字节的时候定时器数值清零，溢出了直接标记flag标志位为接收完成并失能定时器，听不懂我这段话的话看下面图
参考原子的代码

![Markdown](http://i2.kiimg.com/596163/944a71f0e16df163.png)

8266每种个过程无论失败还是成功都会从串口打印消息的，可根据关键字判断来检查8266的状态

