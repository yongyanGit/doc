#### SocketChannel

Java NIO中SocketChannel是一个连接到TCP网络套接字的通道。可以通过以下两种方式创建SocketChannel：

1. 打开一个SocketChannel并连接到互联网上的某台服务器。
2. 一个连接到ServerSocketChannel时，会创建一个SocketChannel。

**打开SocketChannel**

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("http://127.0.0.1",8000));
```

**关闭通道**

```
socketChannel.close();
```

**从SocketChannel读取数据**

调用read()方法从SocketChannel中读取数据，首先创建一个ByteBuffer，然后将数据写入到缓存中，read()方法返回的是读了多少字节到buffer里。

```java
ByteBuffer byteBuffer = ByteBuffer.allocate(48);
int bytesRead = socketChannel.read(byteBuffer);
```

**写入SocketChannel**

写数据到SocketChannel用的是write()方法，该方法以一个Buffer作为参数：

```java
 SocketChannel socketChannel = SocketChannel.open();
       socketChannel.connect(new InetSocketAddress("http://127.0.0.1",8000));

       ByteBuffer byteBuffer = ByteBuffer.allocate(48);

       byteBuffer.clear();

       byteBuffer.put("testa".getBytes());

       byteBuffer.flip();

       while (byteBuffer.hasRemaining()){
           socketChannel.write(byteBuffer);
       }
```

**非阻塞模式**

可以设置SocketChannel为非阻塞模式，设置之后，就可以在异步模式下调用connect()、read()、write()方法。

* connect()方法：如果SocketChannel在非阻塞的模式下，此时调用connect()方法，该方法可能在建立连接之前就返回了。为了确定连接是否建立，可以使用finishConnect()方法:

```java
 SocketChannel channel = SocketChannel.open();
       channel.configureBlocking(false);
       channel.connect(new InetSocketAddress("",8000));

       while (!channel.finishConnect()){
           //wait
       }
```

* write()：可能在尚未写出任何内容时就可能返回了。
* read()：可能在没有读取到数据时就返回了。所以需要根据它的int返回值来判断读取了多少字节。

**阻塞模式与选择器**

非阻塞模式与选择器搭配会工作的更好，通过将一或者多个SocketChannel注册到Selector，可以询问选择器哪个通道已经准备好了读取、写入等。