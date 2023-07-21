Mysql客户端与服务器通信协议为协议栈中的操作系统协议（TCP/IP或本地套接字），
这一模块实现在整个服务器中使用的API，以创建、读取、解析和发送协议包。

其代码位于sql/protocal.cc、sql/protocal.h、sql/net_serv.cc中

protocal.h与protocal.cc定义实现了一个类的体系结构，**PROTOCAL为基类**，
由此派生出Protocal_simple、Protocal_prep、Protocal_cursor

+ sql/net_serv.cc中的my_net_read()

+ sql/net_serv.cc中的my_net_write()

+ sql/net_serv.cc中的net_store_data()

+ sql/net_serv.cc中的send_ok()

+ sql/net_serv.cc中的send_error()

在4.0版本中，该协议改为支持高达4Gb的数据包，在此之前，限值24MB

