### Java网络编程

#### 1. Java Socket

下面是一个Socket编程，服务端启动并且监听8000端口，Socket 客户端连接服务端后，向服务端发送消息。

* Socker Server 

```java
public class MultiThreadEchoServer {


    private static ExecutorService executorService = Executors.newCachedThreadPool();

    static class HandleMsg implements Runnable{

        Socket socket ;

        public HandleMsg(Socket socket){
            this.socket = socket;
        }

        @Override
        public void run() {

            BufferedReader bufferedReader = null;

            PrintWriter printWriter = null;
            try {
                //接收客户端发送过来的消息
                bufferedReader = new BufferedReader(new InputStreamReader(socket.getInputStream()));

                printWriter = new PrintWriter(socket.getOutputStream(),true);

                String inputLine = null;

                long b = System.currentTimeMillis();

                while ((inputLine = bufferedReader.readLine())!= null){
                    // //将服务端的消息发送到客户端
                    printWriter.println(inputLine);
                }

                long e = System.currentTimeMillis();

                System.out.println("spend："+(e -b)+"ms");

            }catch (Exception e){
                e.printStackTrace();
            }finally {
                try {
                    if (bufferedReader != null) {
                        bufferedReader.close();
                    }
                    if (printWriter != null){
                        printWriter.close();
                    }

                    socket.close();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }

        }
    }

    public static void main(String[] args){

        ServerSocket serverSocket =  null;

        Socket clientSocket = null;
        try {
            //定义一个socket服务端，监听8000端口
            serverSocket = new ServerSocket(8000);
        }catch (Exception e){
            e.printStackTrace();
        }

        while (true){
            try {

                clientSocket = serverSocket.accept();

                System.out.println(clientSocket.getRemoteSocketAddress()+"connect");
                //新开线程处理网络请求
                executorService.execute(new HandleMsg(clientSocket));


            }catch (Exception e){
                System.out.println(e);
            }

        }

    }


}
```

* Socket Client  ,客户端没向服务端发送一个消息时，就会阻塞一段时间，在这期间，服务端也会进行等待（cpu进行io等待）。

```java
public class SocketClient {

    private static ExecutorService executorService = Executors.newCachedThreadPool();

    private static final int sleep_time = 1000*1000*1000;


    public static class EchoClient implements Runnable{

        @Override
        public void run() {

            Socket socket = null;

            PrintWriter printWriter = null;

            BufferedReader reader = null;
            try {
                socket = new Socket();

                socket.connect(new InetSocketAddress("127.0.0.1", 8000));

                printWriter = new PrintWriter(socket.getOutputStream(),true);

                printWriter.print("h");
                LockSupport.parkNanos(sleep_time);

                printWriter.print("e");
                LockSupport.parkNanos(sleep_time);

                printWriter.print("l");
                LockSupport.parkNanos(sleep_time);

                printWriter.print("l");
                LockSupport.parkNanos(sleep_time);


                printWriter.print("o");
                LockSupport.parkNanos(sleep_time);

                printWriter.println();

                printWriter.flush();

                reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));

                System.out.println("from server："+reader.readLine());


            }catch (Exception e){
                e.printStackTrace();
            }finally {
                try {
                    if (printWriter != null) {
                        printWriter.close();
                    }
                    if (reader != null) {
                        reader.close();
                    }

                    if (socket != null) {
                        socket.close();
                    }
                }catch (Exception e){

                }
            }

        }
    }

    public static void main(String[] args){

       EchoClient echoClient = new EchoClient();

       for (int i = 0; i < 10; i++){

           executorService.execute(echoClient);

       }

    }

}
```

运行结果：

```java
/127.0.0.1:35568connect
/127.0.0.1:35570connect
/127.0.0.1:35572connect
/127.0.0.1:35574connect
/127.0.0.1:35576connect
/127.0.0.1:35578connect
/127.0.0.1:35580connect
/127.0.0.1:35584connect
/127.0.0.1:35582connect
/127.0.0.1:35586connect
spend：5004ms
spend：5003ms
spend：5004ms
spend：5004ms
spend：5003ms
spend：5005ms
spend：5003ms
spend：5006ms
spend：5004ms
spend：5006ms
```

