# 简介

这是一套经过长时间实盘验证，有超过100亿美金交易量，包含币安合约的数据录入，风控，交易的架构实现，并提供一个简单的交易实践演示数据读取，开仓，平仓，以及止盈止损，风控，还有前端数据展示和分析（前端板块的开源还在准备中，因为现在还有维护一个左侧策略，打算直接把这个左侧策略的数据释放出来示范，应该在7月25号前发布上来）

你可以利用它简单，低成本的实现你的交易逻辑，其大量运用阿里云服务器进行分布式架构，多进程处理，以及飞书进行异常报错，交易信息...

如果你愿意详细阅读该readme的所有信息，尤其是 [模块详细解析](#模块详细解析) ，那么他同时也会是一部关于币安合约的交易风控，设计架构的经验理解历史，总结了几乎本人所有成功和失败的经验，希望能让后人少踩些坑

# 优势

低成本，高效率，简单实现是这套系统的三个优势

不到1000人民币一个月的成本，实现每分钟扫描1500万次交易对是否满足交易条件

除了撮合服务器（C++）外都采用python编写，简单易懂

大量的分布式架构实现更快的速度，并且可以根据个人需求，自由伸缩的调控服务器数量来实现成本和性能的平衡

通过多重接口读取行情/账户信息，并根据更新时间戳进行整合，最大程度的降低数据风险

# 架构

该系统通过一个C++服务器作为主撮合服务器，大量的可伸缩调整的分布式python服务器作为数据采集服务器。

将采集的数据，包括Kline ，trades，tick等，喂给C++服务器。

交易服务器再从C++服务器统一读取数据，并且在本地端维护一个K线账本，来避开交易所的频率限制，实现高效率，低成本的数据读取。

又比如，账户的余额，持仓数据，在币安上有三种方式获取，a是position risk，b是account，c是ws，那么会有三台服务器分别采用这三种方式读取，然后汇入C++服务器进行校对，通过对比更新时间截取最新的数据，然后服务给交易服务器

# 作者自述

2021年，我从一家top量化公司辞职后做起量化交易，主要战场在币安，这两年间，我从做市商->趋势->套利等类型均有涉及，最高峰的时候，在币安一个月有接近20亿美金的交易额。

至2023年7月，因为各种原因，大方向上失败了，只遗留下一个朋友的资金在继续运行一个比较稳定盈利的左侧交易策略。

这是在这两年时间里面探索出来的一套高效率，低成本的数据读取，录入框架，同时包含了一套风控系统，他更像一个架构，而不是一个实现，你同样可以简单的运用到okex，bybit等等上

目前本人在艰难的寻求资金合作或者工作机会中，如能提供帮助请联系 c.binance.quant@gmail.com

# 模块详细解析

我方设计的模块包括 [通用部分](#通用部分) ，[数据处理部分](#数据处理部分) ，[关键操作部分](#关键操作部分) ，以及没有开源的具体交易逻辑部分

该项目的最小化开启需求为 一台web服务器，一台ws服务器，一台tick数据读取服务器，一台one min kline数据读取服务器，以及一台交易服务器

你可以根据自己的需求，扩展相关的服务器，例如需要 15 m 的kline，需要trades等等，只需要在这个基础上增加即可。

我方实盘的项目运行环境为Ubuntu 22.04 64位，Python 3.10.6，由于项目使用的全部库和包都是官方或者热门的，在google可以查找到安装方式，此处不再累述如何安装环境，如果需要最简启动方案，可邮件 c.binance.quant@gmail.com 联系我方，直接共享系统镜像到你方阿里云账号，收取100 USDT的技术费用

我方具体服务器配置为

一台 wsPosition 服务器，用于通过ws的方式读取币安的仓位和余额数据，汇入ws数据撮合服务器

一台positionRisk服务器，用于读取/fapi/v2/positionRisk接口，通过该接口获取仓位信息并实时判断损失，以便于及时止损，该服务器和wsPosition，makerStopLoss，getBinancePosition具备类似功能，之所以设计多种交叉相同功能的服务器，是为了最大限度的防止风险，在币安的某一接口出现延迟的情况下，系统依然可以健壮的运行，以下不再重述

一台makerStopLoss服务器，用于从getBinancePosition服务器读取仓位信息后，读取单独的挂单信息，然后预设止损单，之所以与getBinancePosition服务器进行拆分，是因为读取挂单需要的权重较高，拆分成两个ip可以更高频率的进行操作

一台getBinancePosition服务器，用于读取/fapi/v2/account接口，获得仓位信息和余额后汇入ws数据撮合服务器

一台commission服务器，用于记录流水信息

一台checkTimeoutOrders服务器，用于读取/fapi/v1/openOrders接口，查询全部挂单，然后取消超过一定时间的挂单，或者进行一些额外的操作，例如挂单三秒没被吃，则转换成take订单等等

一台cancelServer服务器，本质上也是web server服务器，运行webServer文件，用于取消订单

一台webServer服务器，本质上也是web server服务器，运行webServer文件，用于大部分程序一开始读取交易对等等

两台oneMinKlineToWs服务器，用于低频率读取一分钟线的kline

两台volAndRate服务器，用于读取交易量数据做分析并提供给其他服务器

五台specialOneMinKlineToWs 服务器，用于高频率读取一分钟线的kline

五台tickToWs服务器，用于读取 tick群信息

一台ws服务器，用于C++的信息戳合服务器

以及五台交易服务器

合计28台服务器，实际上由于只需要选用最低配置的抢占式服务器，成本一个月不足2000元人民币，但是基本可以满足大部分量化的风控和延迟需求


在该项目里，一个单独的模块，需要一台服务器一个IP单独运行，目前基本已经将单模块的https读取频率调教到币安允许的最大值。

此处需要说明的是，这里都是以阿里云日本为例子，币安的服务器在亚马逊云。

而本文所叙述的延迟，其实包含两种延迟，一是读取频率的延迟，二是网络的延迟，这两种延迟综合计算后，才是真实环境下的最终延迟。

之所以使用阿里云是因为阿里云抢占式服务器具备成本优势，从而具备读取频率延迟的优势。

阿里云网络延迟约为10ms，而亚马逊在不申请内网权限的情况下预计为1~3ms，申请内网则面临锁定ip的问题，即无法通过铺开更多ip的手段去降低延迟。

虽然亚马逊云具备更低的延迟，但是由于

1. ws类型的数据读取通常被币安锁定100ms以上的延迟，且ws类型的读取数据具备某些不可确认的风险因素，所以该方式被排除

2. 如果采取https读取的方式，某些数据的读取权重高达20，甚至是30，由此推演出需要多IP，进行分布式读取才能具备更高频率，而这个时候，单个IP的成本价格即成了需要考虑的因素，阿里云抢占式服务器的成本一个月不足20人民币，在综合的性价比考虑后，我方选择了日本阿里云。

## 通用部分

### binance_f文件夹

币安涉及到api key的接口的处理包，是从官网下载后，进行了二次改造的版本

### config.py

通用配置，需要自行申请飞书api key等，亦可简单的替换成其他提醒工具

### commonFunction.py

通用方法

### updateSymbol/trade_symbol.sql

在数据库中生成trade_symbol表格，该表格将控制系统可执行交易的交易对信息，这也是该系统给唯一需要进行数据库储存和处理的地方，你也可以改造为录入文件中并从文件中读取

### updateSymbol/updateTradeSymbol.sql

向trade_symbol表格录入交易对信息，此处进行了一些特殊处理，主要是适应我方的情况，包括只录入usdt的交易对，不录入指数类型的交易对（如btcdom，football这类型），此处部分字段为配合专业手操工具而设计，用于量化的数据字段实际上只有symbol，status

### simpleTrade

一个最基础的交易演示程序，当某个交易对，持仓价值为0且一分钟涨幅>1%的时候开多，当他持仓价值>0且一分钟跌幅小于-0.5%平多

## 数据处理部分

### wsServer.cpp

撮合服务器

所有的数据都会汇入这里，并简单的根据更新时间戳进行筛选，使用以下命令行可编译成可执行文件

g++ wsServer.cpp -o wsServer.out -lboost_system

### dataPy/uploadDataPy

将程序从一个阿里云的主控服务器，上传到各个对应的阿里云服务器，运行，然后销毁。

这里只展示tick，oneMinKlineToWs，specialOneMinKlineToWs三个数据录入程序的使用，其他程序亦同理

该程序可以简单快速的发布分布式运行的程序，到所有符合命名规则的服务器上云运行和销毁。

使用前需要将阿里云服务器进行统一命名，如tickToWs_1,tickToWs_2...

然后调用get_aliyun_private_ip_arr_by_name函数搜索对应字符段的阿里云服务器的私网地址，然后上传，执行，并且在三秒后判断有没有在正常运行

其后会进行覆盖销毁，防止机密数据泄露，只保留程序在内存运行

由于是私网地址的操作，需要在同地域的阿里云服务器上执行该程序

### dataPy/oneMinKlineToWs.py

该程序属于分布式运行架构，只需要标准化命名即可无限扩展服务器降低延迟
![image](https://github.com/Melelery/c-binance-future-quant/assets/139823868/801409a3-25b7-41c8-b795-d7aa0efd0fe6)

缓更新的1分钟的K线数据读取程序

每次程序运行的时候，会像ws服务器发送目前交易对的总数，每次读取kline数据前，会从ws服务器拿到一个交易对编号，拿取的同时，ws服务器会对编号执行 +1的操作，确保分布式架构的时候，每台扩展oneMinKlineToWs都可以按照顺序读取交易对的kline数据。

由于币安的k线数据更新延迟略低于tick数据，所以每次更新前还会再读取一次交易对单独的tick数据去校正最后一条k线数据

kline数据会向ws服务器发送两次数据，一次是所有读取到的数据，比如说读取kline的时候，设置limit=45，即读取了最近45条kline的数据，但很显然，前面43条的数据是一直不变的，所以交易服务器只需要长时间（30秒/1分钟...）去校正一次即可，只有最新的两条数据的变化概率是大的。

所以我拆分成了两条信息，一个是前两段kline，一个是全部kline，前两段kline用于交易服务器实时读取，实时更新，后面两端则用于一定时间间隔后的校对。

其他时间间隔的k线，如5分钟 ，15分钟，1小时等等读取和打入ws服务器的过程亦同理，只需要简单的替换文件中的参数即可实现，所以此处不再列出

### dataPy/specialOneMinKlineToWs.py

该程序属于分布式运行架构，只需要标准化命名即可无限扩展服务器降低延迟

急速更新的1分钟的K线数据读取程序

与上面不同的是，此处的读取数据有一个前置的交易量条件，你可以理解为我的量化系统只有满足某个交易量条件的要求时候才会开仓，所以针对这一部分可能开仓的交易对，铺设了专门读取数据的服务器。

假设本来有200个交易对轮流读取，限制条件后，变成了20个交易对，那么等于你单个机器的数据读取数据提高了10倍

这个只是一个展示程序，实际上你应该根据你的交易条件去自己编写相应的条件，限制数据读取的交易对。

其他时间间隔的k线，如5分钟 ，15分钟，1小时等等读取和打入ws服务器的过程亦同理，只需要简单的替换文件中的参数即可实现，所以此处不再列出

### dataPy/tickToWs.py

该程序属于分布式运行架构，只需要标准化命名即可无限扩展服务器降低延迟

tick数据读取程序会读取阿里云上，所有的tick服务器数量，然后自动锁定该服务器在一秒内的某个时间段内去进行数据读取

打个比方，现在我们开通了五台tick服务器，那么tick 1服务器会在每一秒的>=0 <200毫秒的时间段内去读取数据，tick 2会在每一秒的>=200 <400毫秒的时间段内去读取数据...以此类推

### dataPy/useData.py

展示了如何从ws服务器拿取one min kline数据和tick数据后，在本地端自行拼合维护一个k线数据，此处还可以扩展到加入trade vol等数据，原理相同所以不再展示

## 关键操作部分

### binanceOrdersRecord.py

记录orders信息

### binanceTradesRecord.py

记录trades信息

### checkTimeoutOrders.py

检查是否有超时订单并取消，同时可以附加一些交易操作，如maker挂单超过多少秒没有成交则按比例转化成take订单

### commission.py

记录所有资金流水

### getBinancePosition.py

通过/fapi/v2/account接口获取币安仓位和余额信息，并且上传到本服务器的80端口，以及ws服务，上传到ws服务器的信息，会与positionRisk和wsPosition拿到信息的更新时间戳进行对比，选择最新的信息发送给交易服务器

### positionRisk.py

通过/fapi/v2/positionRisk接口获取币安仓位和余额信息，其他同上

### getBinancePosition.py

通过websocket接口获取币安仓位和余额信息，其他同上

### makerStopLoss

读取仓位信息和挂单信息后，在开仓的同时挂出最大止损单






