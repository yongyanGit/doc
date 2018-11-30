#### ServerSocketChannel

Java NIO中的ServerSocketChannel是一个可以监听新进来的TCP连接的通道，就像标准ServerSocket一样。

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();    
serverSocketChannel.socket().bind(new InetSocketAddress(8000));   
while (true){        
      SocketChannel socketChannel = serverSocketChannel.accept();
       //do something      
 }
```

**监听新进来的连接**

通过ServerSocketChannel的accept()方法监听新进来的连接。当accept()方法返回的时候，它包含一个新进来的SocketChannel。因此accept()方法会一直阻塞到有新的连接到达。

**非阻塞模式**

在非阻塞模式下，accept()方法会立即返回，如果还没有新进的连接，返回的将是null。因此，需要检查返回的SocketChannel是否是null.

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
       serverSocketChannel.configureBlocking(false);
       while (true){

           SocketChannel channel = serverSocketChannel.accept();
           if (channel != null){
               //do something
           }

       }
```

