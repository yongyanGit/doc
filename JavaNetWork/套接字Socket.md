### 套接字Socket

#### 基于TCP协议的Socket程序函数调用过程

TCP的服务端要先监听一个端口，一般先调用bind函数，给这个Socket赋予一个IP地址和端口。端口的作用是当一个网络包来的时候，内核要通过TCP头里面的这个端口来找到这个应用程序，把包给你。有时候一台机器有多个网卡，也就会有多个IP地址，你可以选择监听所有的网卡，也可以选择监听一个网卡，这样，只有发给这个网卡的包，才会给你。

当服务端有了IP地址和端口号，就可以调用listen函数进行监听。在TCP的状态图里面，有一个listen状态，当调用这个函数之后，服务端就进入了这个状态，这个时候客户端就可以发起连接。

如下是Java中SocketServer的实现：

```java
 try {
 	SecurityManager security = System.getSecurityManager();
    if (security != null)
    	security.checkListen(epoint.getPort());
    	getImpl().bind(epoint.getAddress(), epoint.getPort());//绑定ip、端口
        getImpl().listen(backlog);//监听
        bound = true;
    } catch(SecurityException e) {
        bound = false;
        throw e;
    } 
```

在内核中，为每个Socket维护两个队列，一个是已经建立了连接的队列，这时候连接三次握手已经完成，处于established状态；一个是还没有完成建立连接的队列，这个时候三次握手还没完成，处于syn_rcvd的状态。接下来，服务端调用accept()函数，拿出一个已经完成的连接进行处理，如果还没有完成，就要等着。

在服务端等待的时候，客户端可以通过connect函数发起连接。先在参数中指明要连接的IP地址和端口号，然后开始发起三次握手。内核会给客户端分配一个临时的端口。一旦握手成功，服务端的accept就会返回另一个Socket(监听的Socket和真正用来传数据的Socket是两个，一个叫做监听Socket，一个叫做已连接Socket)。

![tcp](../images/net/tcpsocket.jpg)

示例代码：

* 服务端

```java
public void run() throws Exception{

        //创建服务端的Socket，绑定端口，监听此端口
        ServerSocket serverSocket = new ServerSocket(8081);
        //开始监听客户端的连接
        Socket socket = serverSocket.accept();
        //获取输入流，读取客户端传过来的信息
        InputStream inputStream = socket.getInputStream();
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));

        String info = null;

        while ((info = bufferedReader.readLine()) != null){

            System.out.println("服务端："+info);
        }

        socket.shutdownInput();
        OutputStream outputStream = socket.getOutputStream();
        PrintWriter printWriter = new PrintWriter(outputStream);
        printWriter.write("欢迎你");

        printWriter.flush();
        printWriter.close();
        bufferedReader.close();
        serverSocket.close();
    }
```

* 客户端

```java
public void run() throws Exception{

        Socket socket = new Socket("localhost",8081);

        OutputStream outputStream = socket.getOutputStream();

        PrintWriter printWriter = new PrintWriter(outputStream);
        printWriter.write("hello world");
        printWriter.flush();
        socket.shutdownOutput();
        InputStream inputStream = socket.getInputStream();

        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
        String info = null;
        while ((info = bufferedReader.readLine()) != null){
            System.out.println("客户端："+info);
        }

        bufferedReader.close();
        printWriter.close();
        socket.close();

    }
```

在Linux内核中，Socket是一个文件，那对应就有文件秒送符。写入和写出就是通过文件秒送符。每一个进程都有一个数据结构，task_struct，里面指向一个文件描述符数组，用来列出这个进程打开的所有文件的文件描述符。文件描述符是一个整数，是这个数组的下标。这个数组的内容是一个指针，指向内核中所有打开的文件列表。既然是一个文件，就会有一个inode，只不过Socket对应的inode不像真正的文件系统一样，保存在硬盘上，而是在内存中。在这个inode中，指向了Socket的内核中的Socket结构。

![socket](../images/net/SocketStructure.jpg)

Socket结构主要是两个队列，一个是发送队列，一个是接收队列。在这两个队列里面保存的是一个缓存sk_buff,这个缓存里面能够看到完整的包结构。

#### 基于UDP协议的Socket程序函数调用过程

UDP是没有连接的，所以不需要三次握手，也就不需要调用listen和connect，但是，UDP的交互仍然需要ip和端口号，因而也需要bind。UDP是没有维护连接状态的，因此不需要每对连接建立一组Socket，因而只要一个Socket，就能够和多个客户端通信。

![udp](../images/net/udpsocket.jpg)

示例：

```java
 public void server()throws Exception{

        //接收数据
        DatagramSocket datagramSocket = new DatagramSocket(8081);
        byte[] data = new byte[1024];
        DatagramPacket datagramPacket = new DatagramPacket(data,data.length);
        datagramSocket.receive(datagramPacket);
        String info = new String(data,0,data.length);
        System.out.println("服务端接收："+info);

        //发送数据
        InetAddress inetAddress = datagramPacket.getAddress();
        int port = datagramPacket.getPort();
        byte[] sendData = "欢迎".getBytes();
        DatagramPacket packet = new DatagramPacket(sendData,sendData.length,inetAddress,port);

        datagramSocket.send(packet);
        datagramSocket.close();
    }
```

```java
 public void client() throws Exception{
        //向服务端发送数据
        DatagramSocket datagramSocket = new DatagramSocket();

        int port = 8081;

        InetAddress inetAddress = InetAddress.getByName("localhost");
        String data = "helo world";
        DatagramPacket datagramPacket = new DatagramPacket(data.getBytes(),data.length(),inetAddress,port);
        datagramSocket.send(datagramPacket);


        //接收服务器端响应的数据
        byte[] recData = new byte[1024];
        DatagramPacket packet = new DatagramPacket(recData,recData.length);
        datagramSocket.receive(packet);
        String s = new String(recData,0,recData.length);
        System.out.println("客户端收到:"+s);
        datagramSocket.close();
    }
```

