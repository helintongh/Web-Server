# socket简介(Python版)

## 要弄明白的几个问题

- 什么是socket
- 为什么是socket
- 如何使用socket

## socket是什么

- Socket是电脑网络中进程间数据流的端点
- Socket是操作系统的通信机制
- 应用程序通过Socket进行网络数据的传输

## socket是什么更加直观的解释

## socket通信方式

- socket分为UDP和TCP两种不同的通信方式

## 为什么使用Socket

- socket能够适用多种网络协议
- socket是基础应用，了解socket可以举一反三
- 服务器传输大量涉及到网络协议，离不开socket应用

## socket API：
### socket参数



## 从短链接到持久化连接

### 只进行一次通信

```
#导入模块,服务器端
import socket

#创建实例
sk = socket.socket()
#定义绑定ip和port
ip_port = ("127.0.0.1",8888)
#绑定监听
sk.bind(ip_port)
#设置最大连接数
sk.listen(5)
#提示信息
print("正在进行等待接受....")
#接受数据，conn是连接对象，address是客户端地址，处于阻塞状态
conn, address = sk.accept()
#定义信息
msg = "Hello World!"
#返回信息
#python3.x以上，网络数据的发送接受都是byte类型
#如果发送的数据是str型则需进行编码
conn.send(msg.encode())
#主动关闭连接
conn.close()
```

```
#导入模块,客户端
import socket

#实例初始化，使用协议需要一致
client = socket.socket()
#访问的服务器端的ip和端口
ip_port = ("127.0.0.1",8888)
#连接主机
client.connect(ip_port)
#接收主机信息
data = client.recv(1024)
#打印接收的数据
#python3.x以上，网络数据的发送接受都是byte类型
print(data.decode())#decode解码的意思

```

###