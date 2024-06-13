# Android app+ESP32-CAM实现远程监控app

### 文章目录

#### 目录

#### 前言

#### 一、ESP32-CAM设备准备

#### 二、设备接线

#### 三、 Arduino获取视频IP地址以及端口

#### 四、在Android studio上代码实现

#### 总结

## 前言

#### 最近打比赛创新点需要在app里设计添加监控模块，看了CSDN其他文章，借鉴了各位大佬的思路，最近成功解决，由于花费也少，所以在这里总结一下。

## 一、ESP32-CAM设备准备

#### ESP32-CAM开发板（ 30 块左右），USB转TTL设备、ESP32-CAM烧录座（可以不买，后面发现被坑了几块钱）、杜邦线五根

#### 本人于淘宝和pdd上购买，截图如下： 

![image](https://github.com/caixukun-jinitaimei/ESP32-CAM/assets/127214278/fca02237-dbb0-463a-98bd-1d64e5fa61e4)   ![image](https://github.com/caixukun-jinitaimei/ESP32-CAM/assets/127214278/7b63859b-1bf7-4f4b-b492-7096e70f7f60)



#### 这里ESP32-CAM开发板到货后，注意摄像头插槽的使用，可以看下图，将开发板黑色小插槽上翻，刚好压住摄像头元件，正常情况下摄像头是不会脱落的。

![alt text](image-2.png)


## 二、设备接线

#### 用杜邦线将ESP32-CAM的5V、GND、U0T、U0R分别连上USB转TTL设备的5V、GND、RXD、TXD，其中输入电源一定要至少5V 2A，否则视频会出现水纹。另外再用杜邦线将ESP32-CAM的GND与IO0短接，否则后期代码烧录会失败。

![alt text](image-3.png)


## 三、 Ar duino获取视频IP地址以及端口

#### 1.下载Arduino IDE 2.0.4 下载地址：https://www.arduino.cc/en/software

#### 注意下载win10 and newer，64bits

#### 2.设备连接

#### USB转TTL连上电脑，将开发板IO0与GND短接，打开电脑的设备管理器，端口中有CH340，则说明设备连接成功。

![alt text](image-4.png)

#### 3.Arduino代码示例

#### 打开Arduino的exe文件，File—Examples—ESP32—Camera—CameraWebServer，Tools工具栏的Board选择AI Thinker ESP32—CAM，Port选择COM3，在sketch中敲入示例代码：

```
1 #include <Arduino.h>
2 #include <WiFi.h>
3 #include "esp_camera.h"
4 #include <vector>
5
6 #define maxcache 1024 // 图像数据包的大小
7
8 const char* ssid = "*****"; // 输入 wifi 名称
9 const char* password = " "; // 输入电脑连上的 wifi 的密码
10
11 const int LED = 4 ; // 闪光灯
12 const int ZHESHI_LED = 33 ; // 指示灯
13 bool cam_state = true; // 是否开启摄像头传输
14 const int port = 8080 ; 15 String frame_begin = "FrameBegin"; _//_ 图像传输包头
16 String frame_over = "FrameOverr"; _//_ 图像传输包尾
17 String msg_begin = "Esp32Msg"; _//_ 消息传输头
18 _//_ 创建服务器端
19 WiFiServer server;
20 _//_ 创建客户端
21 WiFiClient client;
22
23 _//CAMERA_MODEL_AI_THINKER_ 类型摄像头的引脚定义
24 #define PWDN_GPIO_NUM 32
25 #define RESET_GPIO_NUM -
26 #define XCLK_GPIO_NUM 0
27 #define SIOD_GPIO_NUM 26
28 #define SIOC_GPIO_NUM 27
29
30 #define Y9_GPIO_NUM 35
31 #define Y8_GPIO_NUM 34
32 #define Y7_GPIO_NUM 39
33 #define Y6_GPIO_NUM 36
34 #define Y5_GPIO_NUM 21
35 #define Y4_GPIO_NUM 19
36 #define Y3_GPIO_NUM 18
37 #define Y2_GPIO_NUM 5
38 #define VSYNC_GPIO_NUM 25
39 #define HREF_GPIO_NUM 23
40 #define PCLK_GPIO_NUM 22
41
42 static camera_config_t camera_config = {
43 .pin_pwdn = PWDN_GPIO_NUM,
44 .pin_reset = RESET_GPIO_NUM,
45 .pin_xclk = XCLK_GPIO_NUM,
46 .pin_sscb_sda = SIOD_GPIO_NUM,
47 .pin_sscb_scl = SIOC_GPIO_NUM,
48
49 .pin_d7 = Y9_GPIO_NUM,
50 .pin_d6 = Y8_GPIO_NUM,
51 .pin_d5 = Y7_GPIO_NUM,
52 .pin_d4 = Y6_GPIO_NUM,
53 .pin_d3 = Y5_GPIO_NUM,
54 .pin_d2 = Y4_GPIO_NUM,
55 .pin_d1 = Y3_GPIO_NUM,
56 .pin_d0 = Y2_GPIO_NUM,
57 .pin_vsync = VSYNC_GPIO_NUM,
58 .pin_href = HREF_GPIO_NUM,
59 .pin_pclk = PCLK_GPIO_NUM,
60
61 .xclk_freq_hz = 20000000 ,
62 .ledc_timer = LEDC_TIMER_0,
63 .ledc_channel = LEDC_CHANNEL_0,
64
65 .pixel_format = PIXFORMAT_JPEG,
66 .frame_size = FRAMESIZE_VGA,
67 .jpeg_quality = 31 , _//_ 图像质量 _0-63_ 数字越小质量越高
68 .fb_count = 1 ,
69 };
70 _//_ 初始化摄像头
71 esp_err_t camera_init() {
72 _//initialize the camera_
73 esp_err_t err = esp_camera_init(&camera_config);
74 if (err != ESP_OK) {
75 Serial.println("Camera Init Failed!");
76 return err;
77 }
78 sensor_t * s = esp_camera_sensor_get();
79 _//initial sensors are flipped vertically and colors are a bit saturated_
80 if (s->id.PID == OV2640_PID) {
81 _// s->set_vflip(s, 1);//flip it back_
82 _// s->set_brightness(s, 1);//up the blightness just a bit_
83 _// s->set_contrast(s, 1);_
84 }
85 Serial.println("Camera Init OK!");
86 return ESP_OK;
87 }
88
89 bool wifi_init(const char* ssid,const char* password ){
90 WiFi.mode(WIFI_STA);
91 WiFi.setSleep(false); _//_ 关闭 _STA_ 模式下 _wifi_ 休眠，提高响应速度
92 #ifdef staticIP
93 WiFi.config(staticIP, gateway, subnet); 94 #endif
95 WiFi.begin(ssid, password);
96 uint8_t i = 0 ;
97 Serial.println();
98 while (WiFi.status() != WL_CONNECTED && i++ < 20 ) {
99 delay( 500 );
100 Serial.print(".");
101 }
102 if (i == 21 ) {
103 Serial.println();
104 Serial.print("Could not connect to");
105 Serial.println(ssid);
106 digitalWrite(ZHESHI_LED,HIGH); _//_ 网络连接失败 熄灭指示灯
107 return false;
108 }
109 Serial.print("Connecting to wifi ");
110 Serial.print(ssid);
111 Serial.println(" success!");
112 digitalWrite(ZHESHI_LED,LOW); _//_ 网络连接成功 点亮指示灯
113 return true;
114 }
115
116 void TCPServerInit(){
117 _//_ 启动 _server_
118 server.begin(port);
119 _//_ 关闭小包合并包功能，不会延时发送数据
120 server.setNoDelay(true);
121 Serial.print("Ready! TCP Server");
122 Serial.print(WiFi.localIP());
123 Serial.println(":8080 Running!");
124 }
125 void cssp(){
126 camera_fb_t * fb = esp_camera_fb_get();
127 uint8_t * temp = fb->buf; _//_ 这个是为了保存一个地址，在摄像头数据发送完毕后需要返回，否则会出现板子发送一段时间后自动重启，不断重复
128 if (!fb)
129 {
130 Serial.println("Camera Capture Failed");
131 }
132 else
133 {
134 _//_ 先发送 _Frame Begin_ 表示开始发送图片 然后将图片数据分包发送 每次发送 _1430_ 余数最后发送
135 _//_ 完毕后发送结束标志 _Frame Over_ 表示一张图片发送完毕
136 client.print(frame_begin); _//_ 一张图片的起始标志
137 _//_ 将图片数据分段发送
138 int leng = fb->len;
139 int timess = leng/maxcache;
140 int extra = leng%maxcache;
141 for(int j = 0 ;j< timess;j++)
142 {
143 client.write(fb->buf, maxcache);
144 for(int i = 0 ;i< maxcache;i++)
145 {
146 fb->buf++;
147 }
148 }
149 client.write(fb->buf, extra);
150 client.print(frame_over); _//_ 一张图片的结束标志
151 _//Serial.print("This Frame Length:");_
152 _//Serial.print(fb->len);_
153 _//Serial.println(".Succes To Send Image For TCP!");_
154 _//return the frame buffer back to the driver for reuse_
155 fb->buf = temp; _//_ 将当时保存的指针重新返还
156 esp_camera_fb_return(fb); _//_ 这一步在发送完毕后要执行，具体作用还未可知。
157 }
158 _//delay(20);//_ 短暂延时 增加数据传输可靠性
159 }
160 void TCPServerMonitor(){
161 if (server.hasClient()) {
162 if ( client && client.connected()) {
163 WiFiClient serverClient = server.available();
164 serverClient.stop();
165 Serial.println("Connection rejected!");
166 }else{
167 _//_ 分配最新的 _client_
168 client = server.available();
169 client.println(msg_begin + "Client is Connect!");
170 Serial.println("Client is Connect!");
171 }
172 } 
173 _//_ 检测 _client_ 发过来的数据
174 if (client && client.connected()) {
175 if (client.available()) {
176 String line = client.readStringUntil('\n'); _//_ 读取数据到换行符
177 if (line == "CamOFF"){
178 cam_state = false;
179 client.println(msg_begin + "Camera OFF!");
180 }
181 if (line == "CamON"){
182 cam_state = true;
183 client.println(msg_begin + "Camera ON!");
184 }
185 if (line == "LedOFF"){
186 digitalWrite(LED, LOW);
187 client.println(msg_begin + "Led OFF!");
188 }
189 if (line == "LedON"){
190 digitalWrite(LED, HIGH);
191 client.println(msg_begin + "Led ON!");
192 }
193 Serial.println(line);
194 }
195 }
196
197 _//_ 视频传输
198 if(cam_state)
199 {
200 if (client && client.connected()) {
201 cssp();
202 }
203 }
204 }
205
206 void setup() {
207 Serial.begin( 115200 );
208 pinMode(ZHESHI_LED, OUTPUT);
209 digitalWrite(ZHESHI_LED, HIGH);
210 pinMode(LED, OUTPUT);
```


#### 4.编译烧录（按右向箭头）

#### 下载代码时注意将GND与IO0一直相连，出现Leaving... 和 Hard resetting via RTS pin...，即下载成功。

#### 5.测试结果

#### 将GND与IO0断开，打开右上角的Serial Monitor，选择115200baud波段，按一下ESP32—CAM开发板上的RST复位键，显示IP地址和相应端口，如下图，TCP Server后面就是IP地址，将其记住。

![alt text](image-5.png)

## 四、在Android studio上代码实现

#### 编写Android studio代码，将每一帧视频的图片导入监控页面上，输入在arduino中成功连接的IP地址和端口号。连接以后结果如下，即可将摄像头采集到的视频导入到你想做的app上了。

![alt text](image-6.png)

#### 代码链接：https://github.com/caixukun-jinitaimei/ESP32-CAM

## 总结

#### 以上就是今天要讲的内容，本文介绍了ESP32—CAM结合android app简单的开发使用，本人第一次发文章，还请多多包涵。


