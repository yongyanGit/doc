#### FileChannel

Java NIO中FileChannel是一个连接文件的通道，可以通过文件通道读写文件。FileChannel无法设置为非阻塞模式，它总是运行在阻塞模式下。

**打开FileChannel**

在使用FileChannel之前，必须打开它，我们无法直接打开一个FileChannel，需要通过使用一个InputStream、OutputStream或者RandomAccessFile来获取一个FileChannel。

```java
RandomAccessFile accessFile = new RandomAccessFile("/home/worksapce/nio-data","rw");

FileChannel fileChannel = accessFile.getChannel();
```

**从FileChannel读取数据**

```
ByteBuffer byteBuffer = ByteBuffer.allocate(48);
int byteRead = fileChannel.read(byteBuffer);
```

首先创建一个Buffer，然后调用read方法将Channel中的数据读到Buffer中。read()方法返回的int值表示有多少个字节被读到了Buffer中。如果返回-1，表示到了文件末尾。

**向FileChannel写数据**

```java
 RandomAccessFile accessFile = new RandomAccessFile("/home/yanyong/worksapce/nio-data","rw");
 java.nio.channels.FileChannel fileChannel = accessFile.getChannel();
 ByteBuffer byteBuffer = ByteBuffer.allocate(48);
 byteBuffer.clear();
 byteBuffer.put("this is a test".getBytes());
 byteBuffer.flip();
 while (byteBuffer.hasRemaining()){
 	fileChannel.write(byteBuffer);
 }
```

**关闭FileChannel**

用完FileChannel后必须将其关闭：

```
fileChannel.close();
```

**FileChannel的position方法**

有时可能需要在FileChannel的某个特定位置进行数据读写操作，可以通过调用position()方法或者当前FileChannel的位置。也可以通过posttion(long pos)方法设置FileChannel的当前位置。

```java
long position = fileChannel.position();
fileChannel.position(position+1);
```

如果将position位置设置在文件结束符之后，然后试图从文件通道中读取数据，读方法将返回-1(文件结束标志)。

如果将position位置设置在文件结束符之后，然后向通道中写数据，文化将撑大到当前位置并写入数据。这可能导致“文件空洞”。

**FileChannel的size方法**

返回FileChannel所关联的文件的大小。

```
fileChannel.size();
```





