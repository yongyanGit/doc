#### Java NIO DatagramChannel

Java NIO中的DatagramChannel是一个能收发UDP包的通道。因为UDP是无连接的网络协议，所以不能像其它通道那样读取和写入。它发送个接收的是数据包。

**打开DatagramChannel**

```java
DatagramChannel channel =  DatagramChannel.open();   
channel.socket().bind(new InetSocketAddress(9999));//可以在9999端口上接收数据包
```

**发送数据**

通过send()方法从DatagramChannel发送数据，客户端不能感知服务端是否成功接收了消息。

```java
DatagramChannel channel =  DatagramChannel.open();

      channel.socket().bind(new InetSocketAddress(9999));
      
      String newData = "NEW STRING";
      
      ByteBuffer byteBuffer = ByteBuffer.allocate(48);
      
      byteBuffer.clear();
      
      byteBuffer.put(newData.getBytes());
      
      int byteSent = channel.send(byteBuffer,new InetSocketAddress("127.0.0.1",9999));
```

**发送到特定的地址**

可以将DatagramChannel连接到网络中特定的地址。由于UDP是无连接的，连接到特定的地址不会像tcp那样创建一个真正的连接。而是锁住DatagramChannel，让其只能从特定地址收发数据。

```
 DatagramChannel channel =  DatagramChannel.open();

 channel.connect(new InetSocketAddress("127.0.0.1",80));
```

当连接后，也可以使用read()和write()方法。