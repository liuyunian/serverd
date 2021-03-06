# serverd
C++实现的TCP服务器框架  
采用master-worker + Reactor + ThreadPool实现高并发

## 目录结构
* app: main.cpp所在的目录
* net: 存放与网络通信相关的代码
* proc: 存放与进程处理相关的代码
* signal: 存放与信号处理相关的代码

## 使用
### 依赖
1. 测试OS: Ubuntu 18.04 desktop
2. 编译器: gcc 7.4.0
3. 构建工具: cmake 3.10.2
4. 依赖库: [tools-cxx](https://github.com/liuyunian/tools-cxx)

### 编译
1. 构建: make
2. 安装: make install
3. 卸载: make uninstall
4. 清除: make clean

### 测试
* 编译完成未安装情况下: ./_build/serverd -h
* 已安装情况下: serverd -h

## 修改日志
**2019-11-16 目录结构和构建脚本**  
目录结构如上所示  
项目的构建工具为cmake

**2019-11-20 配置文件**  
采用tools-cxx的[config库](https://github.com/liuyunian/tools-cxx/blob/master/tools/config/README.md)来解析项目中的配置文件[serverd.conf](/serverd.conf)

**2019-11-22 日志**  
采用tools-cxx的[多线程日志库](https://github.com/liuyunian/tools-cxx/blob/master/tools/log/README.md)  
日志信息写入日志文件中，日志文件的路径可在配置文件中指定，默认存储在项目根目录下的logs目录中

**2019-11-24 master-worker**  
master主进程创建worker子进程，worker进程数可以通过配置文件设置    
每个进程独占一个日志文件，日志文件名中有进程ID  
提供了修改进程标题(ps命令CMD栏显示的内容)的接口

**2019-11-27 信号处理**  
封装了信号集SignalSet、进程信号屏蔽字SignalMask、信号处理注册类SignalHandler  
master进程注册了SIGCHLD信号处理函数，避免了worker进程终止之后变为僵尸进程

**2019-11-28 守护进程**  
参考nginx源码实现了程序以守护进程方式运行  
为了代码的统一性取消了是否以守护进程方式运行的选项，只能以守护进程方式运行 

**2019-12-30 接收数据**  
采用了tools-cxx封装的[poller](https://github.com/liuyunian/tools-cxx/blob/master/tools/poller/README.md)、[Socket](https://github.com/liuyunian/tools-cxx/blob/master/tools/socket/README.md)实现了每个worker进程以Reactor模式运行并监听socket事件  
参考muduo网络库封装了TCPConnection和TCPServer，以注册回调函数方式处理业务逻辑

**2020-01-01 应用层缓冲区**  
参考muduo网络库增加了应用层缓冲区类Buffer  
每个TCPConnection对象都有一个输入缓冲区和一个输出缓冲区，输入和输出缓冲区都是必须的：
* 输入缓冲区应对所接收数据包不完整的情况
* 输出缓冲区应对发送数据时一次系统调用（write(2)）发送不完的情况

**2020-01-02 发送数据**  
TCPConnection增加了send()函数  
TCPConnection增加了shutdown()函数，服务器端可以主动关闭连接

**2020-01-03 增加高/低水位回调函数**  
参考muduo网络库给TCPConnection增加了如下两个回调函数，作用是：让上层应用调节发送数据速率，避免数据在本地内存中堆积
* WriteCommpleteCallback（低水位回调函数）：当直接调用write(2)发送的数据或者输出缓冲区的数据都写入内核发送缓冲区之后则执行该回调
* HighWaterMaskCallback（高水位回调函数）：当输出缓冲区的数据加上目前要发送的数据大于一个高水位阈值时则执行该回调  

**2020-01-04 网络库模式**  
之前是按照应用的方式实现，现在改成了网络库架构，具体的改变如下：
* 将master-worker模式放置在TCPServer类中，只对上层应用暴露TCPServer类
* 增加Acceptor类，接替原TCPServer功能
* 弃用配置文件
* 打破原有的目录结构，将网络库代码统一放置在serverd目录下
* 修改构建脚本，编译生成的libserverd.a静态文件

使用signalfd重构了SignalHandler类  
在examples目录下使用libserverd.a实现常见的TCPServer

**2020-01-13 重构TCPServer类**  
之前TCPServer类设计不够优雅，堆积了很多不该属于TCPServer类的属性和方法，因此做了如下重构：
* 重构了Process类，使其能作为父类存在
* 新设计了MasterProcess和WorkerProcess类继承自Process类，并承担了诸多原来存在在TCPServer类中的属性和方法
* 重新启用配置文件，减少参数传递数量，同时也方便实现热更新功能

**2020-01-16 增加HttpServer示例**  
参考muduo网络库实现了HttpServer、HttpRequest、HttpResponse等类  
TCPConnection类中增加了一个std::any成员，根据上层应用需要存放任意类型的数据
