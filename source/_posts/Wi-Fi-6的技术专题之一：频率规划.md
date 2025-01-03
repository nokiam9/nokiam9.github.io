---
title: Wi-Fi 6的技术专题之一：频率规划
date: 2022-03-27 17:01:15
tags:
---

Wi-Fi联盟（英语：Wi-Fi Alliance，缩写WFA），是一个商业联盟，拥有Wi-Fi的商标。它负责Wi-Fi认证，与商标授权的工作，也负责制定Wi-Fi的标准，总部位于美国德州首府Austin。

Wi-Fi背后的技术标准，是由美国的电气电子工程师协会（IEEE）制定的802.11系列协议。在最初的很多年里，Wi-Fi虽然一代代向前发展，但世界上并没有Wi-Fi几代这样的说法，直接就用802.11后面加几个字母这样的协议编号，对普通用户非常不友好。

2018年，Wi-Fi联盟决定把下一代技术标准802.11ax用更为简单易懂的Wi-Fi 6来宣传，上一代的802.11ac和802.11n就被追溯成了Wi-Fi 5和Wi-Fi 4，至于Wi-Fi 1/2/3，并没有得到大规模的推广应用，技术也已淘汰，就没啥好讨论的了。

{% asset_img wifi.png %}

本文主要讨论Wi-Fi的工作频段，也就是2.4GHz和5GHz，这两个频段均属于ISM频段（Industrial Scientific Medical Band）,详细分析请参见附录一。
根据工信部的规定，国内无线接入系统（Wi-Fi）频率的使用范围是：

|系统名称 |频率范围|
|:-:|:-:|
|2400MHz 频段无线接入系统|2400-2483.5MHz|
|5100MHz 频段无线接入系统|5150-5350MHz（仅限于室内使用），其中 5250-5350MHz 应具备DFS功能|
|5800MHz 频段无线接入系统|5725-5850MHz|

## 一、2.4GHz频段

使用2.4GHz频段的无线技术标准较多且发展较早，不仅是Wi-Fi，蓝牙和Zigbee等也普遍采用该频段，造成该频段早已拥堵不堪。相对于后来的5GHz频段，2.4G频段的波长更短，传输衰减更小，穿透力更强，但在增强了信号的同时也加剧了2.4GHz频段的干扰。

ISM中的2.4GHz频段有2.4-2.5GHz，总计100MHz的可用频率。在中国大陆，Wi-Fi被允许使用2.40-2.4835GHz，共计83.5MHz的频率来放置各个信道。

- 这些频率被划分为13个20MHz的信道，信道与信道的中心频点之间的频率差为5MHz，共使用2.412-2.472GHz这60MHz频率来放置中心频点
- 为了防止信道之间干扰，每个20MHz宽的信道，实际占用22MHz的频率，且信道两端各使用了1MHz保护频率来防止干扰。
- 为了不与其他非2.4GHz频段的无线电设备干扰，Wi-Fi还在其起始频率处增加了1MHz的保护频率，来避免与其他频段互相干扰。

{% asset_img 2.4GHz.png %}

> 信道14的定义不规律，与13信道的间隔为12MHz，位于该ISM频段的上限，在美国和世界大部分地区都被禁止，但在日本却被允许
> 尽管在中国允许使用12-13信道（美国要求限制功率），但由于存在较严重的邻频干扰，许多路由器厂商建议不使用

- 最初的802.11b，采用22MHz的信道带宽，最多只能容纳3组没有重叠的信道，即：1、6、11。
- 后续802.11g/n，采用20MHz的信道带宽，可以容纳4组没有重叠的信道，即：1、5、9、13。
- 802.11n首次支持40MHz的信道带宽（不支持80MHz带宽），但只有2组没有重叠的信道，即：3、11。

{% asset_img 24.png %}

## 二、5GHz频段

5GHz频段的开放较晚，可用频谱充足（4.910GHz-5.875GHz，有900多M的带宽，是2.4G的10倍还多）、干扰小、速率高，是目前Wi-Fi的首选，但也存在信号穿透力差、空气中衰减较大等问题。常见设备有Wi-Fi、气象雷达、无线图传等。

5GHz Wi-Fi仍然延续了2.4GHz Wi-Fi信道的命名标准（但为减少干扰，不再允许信道之间频率重叠），从5.0GHz开始称为信道1，每个信道宽度为20Mhz，之后每隔5MHz中心频率，信道数增加1。例如，38信道就是指中心频率为5190Mhz，宽度为20MHz的信道。

2002年，工信部发布了[5.8GHz的频率管理规定](https://www.miit.gov.cn/jgsj/wgj/wjfb/art/2020/art_bfd0fd64e0a1427aaf8aff18a01c0fd0.html)，开放了5.8GHz频段（也称为5GHz高频段），范围为5.745–5.825GHz，共计100MHz，划分为149-165信道。

> 当时，该频段也被移动运营商用于无线局域网的CPE接入，但工信部要求须取得相应的基础电信业务经营许可，且收取频率占用费。

2012年，中国工信部正式开放5.2GHz频段（也称为5GHz低频段），范围为5150-5350MHz，共计200MHz频率，划分为高低部分，各100Mhz。其中：

- 5170-5250MHz，共80MHz，划分为36-48信道，可以直接使用，
- 5250-5330MHz，共80MHz，划分为52-64信道，为DFS信道；
- 信道前后各空余的20Mhz频率作为保护带使用

{% asset_img 5GHz.jpg %}

根据IEEE 802.11系列标准，Wi-Fi可以采用20MHz、40MHz、80MHz和160MHz等4种方式，信道越宽，传输速率也越高。

- 对于20MHz带宽，5GHz频段可以容纳13个信道，包括5.2GHz的8个信道(其中4个需支持DFS)，和5.8GHz的5个信道
- 对于40MHz带宽，5GHz频段可以容纳6个信道，包括5.2GHz的4个信道（其中2个需支持DFS），和5.8GHz的2个信道
- 对于80MHz带宽，5GHz频段可以容纳3个信道，包括5.2GHz的2个信道（其中1个需支持DFS），和5.8GHz的1个信道
- 中国目前不允许160MHz带宽，因为5.8Ghz频段的带宽不足，而5.2GHz频段由于需要支持DFS无法使用

## 三、DFS & TPC

由于许多军用、气象用雷达也都使用5GHz的频段，当中有些频段与Wi-Fi有所重叠，例如欧洲军方的雷达系统广泛运用这一频率(其中探测隐型飞机的雷达就使用这一频率)，如果民用的无线产品也使用这一频率，很可能会对军事雷达和通讯产生干扰。为此，IEEE组织制定了**802.11h**规范并要求Wi-Fi厂商遵循，也就是：`Auto DFS + Auto TPC = 802.11h`。

简单来说，DFS就是要求无线产品主动探测军方使用的频率，并主动选择其他频率以避免干扰。
DFS（Dynamic Frequency Selection）动态频率选择，是通过动态将5GHz无线电的工作频率切换到不干扰雷达的频率以避免干扰雷达信号的过程。
TPC（Transmit Power Control）发射功率控制，是指根据法规要求和范围信息来调整无线电的发射功率。

802.11h的技术认证是属于强制性的，不符合标准的产品将不会获得欧盟及有此项规范要求的国家的无线产品上市许可。目前世界各国对Auto DFS & Auto TPC的使用频率范围规定不一，但要求具备这两项机能的趋势，却是确定的。其核心要求是：

- DFS频道时不得在户外环境、只能在室内使用
- 当Wi-Fi路由器准备启用DFS信道之前，必须先**静默等待60秒**，以确认没有军事与气象雷达正在使用
- 具备实时侦测军事与气象雷达信号的能力，一旦发现必须自动回避并切换到监管允许的其他频道，以免干扰到军事雷达与气象雷达运作

{% asset_img dfs.jpeg %}

> 在中国，政府规定52-64信道需满足DFS要求，为避免DFS功能造成数据传输中断，路由器厂商通常不使用这些信道用于MESH回传中继

## 四、关于频率干扰的讨论

### 其他通讯协议

由于ISM频段无需授权，除了Wi-Fi以外，许多通信协议也在同时使用这些频段。

- 蓝牙同样采用2.4GHz频段（2.400-2.4835MHz），规划了40个物理信道，每个信道带宽2MHz，包含3个广播信道和37个数据信道
  {% asset_img 蓝牙和Wi-Fi的信道对比 bluetooth.png %}
- 大疆公司开发的全高清数字图像传输系统Lightbridge，同样工作在2.4GHz和5GHz频段，但自主开发了基于广播模式的数据协议，由于省略了握手的协议开销，延迟更短，速率更高，传输距离也高达1.7公里。
- 基于IEEE 802.15.4标准的ZigBee协议、LoRa协议也使用2.4GHz频段
- 高速公路ETC收费系统使用基于5.8GHz的有源电子标签

### USB 3.0

2012年，Intel发布了技术白皮书[《USB 3.0 Radio Frequency Interference Impact on 2.4 GHz Wireless Devices》](https://www.usb.org/sites/default/files/327216.pdf)，明确指出USB3.0在使用时，会在2.4G频段增加约20dB的噪声，造成对ISM频段的射频干扰。这种干扰会降低无线接收的灵敏度，进而缩减收讯范围，足以影响干扰无线设备（无线网卡、无线鼠标及无线耳机等）的正常使用。

实际上，USB 3.0的信号速率是５Gbit/s，但由于USB 3.0芯片需要支持数据加密，为此在时钟上应用了扩频技术，导致其频谱从0Hz一直延伸到5GHz。经Intel测量，干扰功率随频率下降，在2.4G频段约有-60dBm，到5G频段只有-90dBm。很可惜的是，这个由USB 3.0高频通讯所产生的噪讯是一种宽频噪讯，因此无法被过滤消除，而且刚好落在常用的2.4-2.5GHz的频段范围。

{% asset_img usb.jpeg %}

Intel建议的解决方式是对USB 3.0连接器及周边装置进行遮蔽设计，做得愈彻底，效果愈好。此外，无线天线放得离USB 3.0连接器及装置也要愈远愈好。

> 路由器厂商通常建议用户关闭USB 3.0接口，改用USB 2.0模式。

### 微波炉

此外，家用微波炉也可能对2.4GHz产生严重干扰，因为其辐射频段是2.450GHz，而且功率超大，但前提是微波炉的密封不好造成辐射泄露。有兴趣的，可以看看[1998年澳大利亚Parkes射电天文望远镜的乌龙事件](https://news.sciencenet.cn/htmlnews/2015/5/318644.shtm)。

## 五、其他频段

在美国，Wi-Fi可以采用3.9GHz频段，具体为3.655-3.695GHz之间的40M带宽，但必须获得政府许可执照。
此外，美国Wi-Fi还采用了4.6GHz频段，具体为4.940-4.990GMHz的50M带宽，但仅限美国公共安全机构使用。

当前，Wi-Fi的下阶段演进目标是**WiFi 6E**，多了一个E，代表着“Extended”，扩展的意思，主要改进点在于支持 6GHz频段，这就意味着WiFi 6E设备能够支持2.4GHz、5GHz和6GHz三个频段，并由此带来了很多优势。

与已经普遍使用的2.4GHz、5GHz频段相比，6GHz频段在很多地区还未开放，所以使用6GHz频段的设备比较少，自然在这个频段上的拥堵、干扰就比较少。同时6GHz频段处于5925-7125MHz之间，包含7个160MHz信道、14个80MHz信道、29个40MHz信道、60个20MHz信道，总计110个信道。信道数量比5GHz频段和2.4GHz频段有了大幅的增长，带来更为强悍的吞吐能力，特别是160MHz信道从5GHz的1个提升到了7个，能实现7条互不干扰的连接了。

---

## 附录一：ISM（Industrial Scientific Medical）频段

ISM频段（Industrial Scientific Medical Band）主要是开放给工业、科学和医用3个主要机构使用的公共频段资源。
无需事前获得政府机构的授权，任何人都可以使用ISM频段进行数据传输，但必须遵循管制要求限制发射功率，目的是控制辐射范围以避免干扰他人使用。当然，如果ISM频段内的无线电设备之间产生了干扰，原则上也不受政府保护，使用者应自行解决或协商解决。

在美国，ISM频段是由美国联邦通信委员会（FCC）负责定义，其他大多数国家和地区的政府也都安排了ISM频段，但具体规定并不统一。

在我国，根据《中华人民共和国无线电频率划分规定》及其中脚注5.138和5.150，规定：

- 6765-6795kHz（中心频率6780kHz）、61-61.5GHz（中心频率61.25GHz）、122-123GHz（中心频率122.5GHz）、244-246GHz（中心频率245GHz）频段用于ISM应用，但其使用须经无线电主管部门给予特别批准；
- 13553-13567kHz（中心频率13560kHz）、26957-27283kHz（中心频率27120kHz）、40.66-40.7MHz（中心频率40.68MHz）、2400-2500MHz（中心频率2450MHz）、5725-5875MHz（中心频率5800MHz）、24-24.25GHz（中心频率24.125GHz）频段也用于ISM应用，在这些频段内工作的无线电业务必须承受由于这些ISM应用产生的有害干扰
- 国际上，433.05-434.79MHz（中心频率433.92MHz）频段是国际电联第一区部分欧洲国家指定用于ISM应用的频段，902-928MHz（中心频率915MHz）频段是美国等国际电联划分的第二区国家指定用于ISM应用的频段，上述指定用于ISM应用的频段也划分给了无线电业务使用，不论是ISM应用还是符合划分的无线电业务，都可以使用上述频段，但均要符合我国无线电管理有关规定

Wi-Fi所使用的2.4G频段与5.8G频段就落在全球各国共有的ISM频段上，此外常见的ISM频段还有：

### 13.56MHz

频率范围为13.553-13.567MHz，处于短波频段。
这是最典型的RFID / NFC高频工作频率，是实际应用中使用量最大的电子标签，国际标准有ISO14443、ISO15693和ISO18000-3等。

> 传统ID卡的工作片一般是125KHz或135kHz，属于低频卡，常见国际标准有ISO11784/11785、ISO18000-2等。
> 但是，30KHz～300KHz的低频卡通常并不纳入ISM频段范围。

### 27.125MHz

频率范围为26.957-27.283MHz。
除了电感耦合RFID系统外，这个频率范围的ISM应用还有医疗用电热治疗仪、工业用高频焊接装置和传呼机等，国内很多遥控玩具工作频率在27MHz.

### 315MHz

北美地区很早就将315MHz纳入ISM频段，广泛地运用在车辆监控、遥控遥测、无线抄表、门禁系统、工业数据采集、安全防火系统、生物信号采集、水文气象监控、机器人控制等领域。
这也是国内早期无线遥控产品的主要频段，但由于这个频段的产品庞杂造成干扰严重，后期逐步转向使用433MHz频段。

### 433.920MHz

频率范围为430.050～434.790MHz，属于UHF频段，电磁波遇到建筑物或其他障碍物时，将出现明显的衰减和反射。。
在世界范围内分配给业余无线电服务使用，该频段可用于反向散射RFID系统，除此之外，还可用于小型电话机、遥测发射器、无线耳机、近距离小功率无线对讲机、汽车无线中央闭锁装置等。

### 民用对讲机

民用对讲机也采用ISM频段，但各国的规定并不统一，我国开放的频点数20个，频率范围409.750-409.9875MHZ，间隔125KHz。
美国则是14个频点，分462MHz和467MHz两组，每组7个频点，日本、韩国也采用美国标准。

## 附录二：关于发射功率的法律限制

无线路由器的功率很低，而且严格受到国家的管控，只要是符合国家标准的产品，都是安全的。
中国工信部对于[Wi-Fi各频段的发射功率有严格要求](https://wap.miit.gov.cn/cms_files/filemanager/1226211233/attach/20219/d125301b13454551b698ff5afa49ca28.pdf)，对路由器无线辐射的担心是不必要的。
无线功率的常用单位有：分贝毫瓦（dBm），毫瓦（mW）

```txt
EIRP（等效全向辐射功率） = 有效功率 + 天线增益 - 天线馈线线路损耗
```

### 2.4GHz频段

天线增益＜10dBi时：发射功率≤100mW 或 ≤20dBm，一般是家用路由器和室内AP。
天线增益≥10dBi时：发射功率≤500mW 或 ≤27dBm，一般是室外AP。

### 5.2Ghz频段（36~64信道）

中国：发射功率≤200mW 或 23dBm

### 5.8Ghz频段（149~165信道）

发射功率≤2000+mW或33dBm，射频口发射功率 <=500mW 或 27dBm（功放组合功率）

> 美国、澳大利亚等国家对发射功率的要求较为宽松，一般为4000mW，或36dBm，因此许多路由器可以通过修改国家属性提高覆盖范围

## 附录三：中国移动通信的无线频率分布

{% asset_img YD.png %}
{% asset_img DX.png %}
{% asset_img LT.png %}
{% asset_img GD.png %}

## 附录四：车联网的常用频率

{% asset_img auto.png %}

## 附录五：昙花一现的无线城市

2008开始中国进入3G移动通信时代，受到通信技术标准的困扰，中国移动大力推广“无线城市”，试图采用基于802.11n的Wi-Fi技术分流移动网络流量，通过大功率CPE实现大范围的室外覆盖，曾经宣布已在全国部署300万个Wi-Fi热点并提供商用，但始终举步维艰，重要原因是：

- 频率资源限制：根据通信蜂窝网的基本原则，有效覆盖至少需要3个互不干扰的40MHz信道，也就是120MHz频率。但2.4G频段只有83.5MHz，先期开发的5.8GHz只有100MHz，5.2GHz有干净的80MHz，但另外80MHz需要满足DFS的要求，始终无法提供足够的频率资源。
- 商业模式缺失：由于IEEE 802.11是IT技术而非通信技术，其WAP2认证协议的功能非常简单，当时提出要攻克Wi-Fi认证难题，实际是希望实现基于时长或流量的收费模式，但最终无疾而终

随着2013年4G牌照的发放，运营商放弃无线城市转而全力发展移动网络，而工信部也全面放开5GHz频段，用户也不再为那些室外CPE的频率干扰闹心了。

---

## 参考文献

### 官方网站

- [工信部-关于加强和规范2400MHz、5100MHz和5800MHz频段无线电管理有关事宜的通知](https://wap.miit.gov.cn/zwgk/zcwj/wjfb/tz/art/2021/art_e4ae71252eab42928daf0ea620976e4e.html)
- [Wi-Fi Alliance的官方网站](https://www.wi-fi.org/zh-hans)

### 技术评论

- [Wi-Fi的那些事：无线电、速率与组网](https://yuanze.wang/posts/things-about-wifi/)
- [无线路由器及Wi-Fi组网指南（史上最全）](https://www.eet-china.com/mp/a77283.html)
- [无线局域网信道列表 - Wiki](https://zh.wikipedia.org/wiki/%E6%97%A0%E7%BA%BF%E5%B1%80%E5%9F%9F%E7%BD%91%E4%BF%A1%E9%81%93%E5%88%97%E8%A1%A8)
- [何为ISM频段？ISM频段主要频率有哪些？](https://www.sohu.com/a/432446555_814535)
- [射频识别RFID不同工作频段有何特点](http://www.iwl.iiot.com/news/345.html)
- [雷达的工作频率](http://uuspider.com/2015/01/14/01.html)

### 资料下载

- [WLAN频段介绍.pdf](WLAN频段介绍.pdf)
- [Intel关于USB 3.0频率干扰的技术白皮书](Intel关于USB3.0无线干扰的白皮书.pdf)
- [工业互联网和物联网无线电频率使用指南（2021年版）](工业互联网和物联网无线电频率使用指南-2021.pdf)
- [车联网发展、法律法规和测试研究](00016c58d7ef1d09ab3538.pdf)
