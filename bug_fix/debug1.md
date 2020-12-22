# Java Socket 循环接收数据造成线程阻塞

**问题**：socket server向client发送消息，client收到后回复一条消息给server，**接收消息采用循环导致两个线程都阻塞**，而非循环就能完成消息接收和发送。

非循环接收：

~~~java
public class test{
public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
                try{
                    ServerSocket serverSocket = new ServerSocket(8081);
                    System.out.println("server start ...");
                    while (true){
                        Socket socket = serverSocket.accept();
                        if(socket!=null){
                            System.out.println("socket connect sucess ...");
                            DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
                            System.out.println("server send message ...");
                            outputStream.writeBytes("hello client");
                            socket.shutdownOutput();
                            InputStream inputStream = socket.getInputStream();
                            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
                            bufferedReader.lines().forEach(t->System.out.println(t));
                            System.out.println("server receive message ...");
                            inputStream.close();
                            outputStream.close();
                            socket.close();
                            break;
                        }
                    }
                }catch (Exception e){
                    e.printStackTrace();
                }
        });
        Thread t2 = new Thread(()-> {
                try {
                    Socket socket = new Socket("127.0.0.1", 8081);
                    System.out.println("client start ...");
                    InputStream inputStream = socket.getInputStream();
                    BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
                    bufferedReader.lines().forEach(t->System.out.println(t));
                    System.out.println("client receive message");
                    DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
                    System.out.println("client send message ...");
                    outputStream.writeBytes("hello server");
                    outputStream.close();
                    inputStream.close();
                    socket.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
        });
        t1.start();
        t2.start();
    }
}
~~~

输出：

> server start ...
> client start ...
> socket connect sucess ...
> server send message ...
> hello client
> client receive message
> client send message ...
> hello server
> server receive message ...

发送接收成功，线程正常结束。

循环接收：

```java
public class test {
public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(()->{
                try{
                    ServerSocket serverSocket = new ServerSocket(8081);
                    System.out.println("server start ...");
                    while (true){
                        Socket socket = serverSocket.accept();
                        if(socket!=null){
                            System.out.println("socket connect sucess ...");
                            DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
                            outputStream.writeBytes("hello client");
                            System.out.println("server send message ...");
                            InputStream inputStream = socket.getInputStream();
                            List<String> message = new BufferedReader(new InputStreamReader(inputStream)).lines().collect(Collectors.toList());
                            System.out.println(message.toString());
                            System.out.println("server receive message ...");
                            inputStream.close();
                            outputStream.close();
                            socket.close();
                            break;
                        }
                    }
                }catch (Exception e){
                    e.printStackTrace();
                }
        });
        Thread t2 = new Thread(()->{
                try {
                    Socket socket = new Socket("127.0.0.1", 8081);
                    System.out.println("client start ...");
                    InputStream inputStream = socket.getInputStream();
                    List<String> message = new BufferedReader(new InputStreamReader(inputStream)).lines().collect(Collectors.toList());
                    System.out.println("client receive message");
                    System.out.println(message.toString());
                    DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
                    System.out.println("client send message ...");
                    outputStream.writeBytes("hello server");
                    outputStream.close();
                    inputStream.close();
                    socket.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
        });
        t1.start();
        t2.start();
    }
}
```

输出：

>server start ...
>client start ...
>socket connect sucess ...
>server send message ...

server发送出消息，client没有打印，之后线程阻塞。

**原因**：server发送完消息后就打开inputstream进入消息接收模式，而此时client接收到第一条消息后，进入循环等待接收第二条消息，server和client都是消息接收模式等待对方发送消息，形成死锁导致线程阻塞。因此server发送完消息后要通知client关闭接收，client才能继续发消息给server。client不能直接关闭inputstream，会导致整个socket关闭，而是应在server端调用socket的shutdownOutput方法，该方法将socket中现有的数据发送完后发送一个终止标记（-1），使接收端停止接收。

**shutdownOutput的示例**

```java
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        try{
            ServerSocket serverSocket = new ServerSocket(8081);
            System.out.println("server start ...");
            while (true){
                Socket socket = serverSocket.accept();
                if(socket!=null){
                    System.out.println("socket connect sucess ...");
                    DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
                    outputStream.writeBytes("hello client");
                    socket.shutdownOutput();
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    });
    Thread t2 = new Thread(()->{
        try {
            Socket socket = new Socket("127.0.0.1", 8081);
            System.out.println("client start ...");
            InputStream inputStream = socket.getInputStream();
            int i=0;
            while (i!=-1){
                i=inputStream.read();
                System.out.print(i+"...");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
    t1.start();
    t2.start();
}
```

输出：最后一个字节为-1 

>server start ...
>client start ...
>socket connect sucess ...
>104...101...108...108...111...32...99...108...105...101...110...116...-1...

**使用shutdownOutput后阻塞消失**

```java
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(()->{
        try{
            ServerSocket serverSocket = new ServerSocket(8081);
            System.out.println("server start ...");
            while (true){
                Socket socket = serverSocket.accept();
                if(socket!=null){
                    System.out.println("socket connect sucess ...");
                    DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
                    outputStream.writeBytes("hello client");
                    System.out.println("server send message ...");
                    socket.shutdownOutput();
                    InputStream inputStream = socket.getInputStream();
                    List<String> message = new BufferedReader(new InputStreamReader(inputStream)).lines().collect(Collectors.toList());
                    System.out.println(message.toString());
                    System.out.println("server receive message ...");
                    inputStream.close();
                    outputStream.close();
                    socket.close();
                    break;
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    });
    Thread t2 = new Thread(()->{
        try {
            Socket socket = new Socket("127.0.0.1", 8081);
            System.out.println("client start ...");
            InputStream inputStream = socket.getInputStream();
            List<String> message = new BufferedReader(new InputStreamReader(inputStream)).lines().collect(Collectors.toList());
            System.out.println("client receive message");
            System.out.println(message.toString());
            DataOutputStream outputStream = new DataOutputStream(socket.getOutputStream());
            System.out.println("client send message ...");
            outputStream.writeBytes("hello server");
            outputStream.close();
            inputStream.close();
            socket.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    });
    t1.start();
    t2.start();
}
```

输出：正常接收

~~~
client start ...
socket connect sucess ...
server send message ...
client receive message
[hello client]
client send message ...
[hello server]
server receive message ...
~~~

