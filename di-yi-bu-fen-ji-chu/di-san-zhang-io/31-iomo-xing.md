这部分的东西在网络编程经常能看到，不过在所有IO处理中都是类似的。

**IO请求的两个阶段**：

**等待资源阶段**：IO请求一般需要请求特殊的资源（如磁盘、RAM、文件），当资源被上一个使用者使用没有被释放时，IO请求就会被阻塞，直到能够使用这个资源。

**使用资源阶段**：真正进行数据接收和发生。

举例说就是**排队**和**服务。**

在**等待数据**阶段，IO分为阻塞IO和非阻塞IO。

**阻塞IO**：资源不可用时，IO请求一直阻塞，直到反馈结果（有数据或超时）。

**非阻塞IO**：资源不可用时，IO请求离开返回，返回数据标识资源不可用

在**使用资源**阶段，IO分为同步IO和异步IO。

**同步IO**：应用阻塞在发送或接收数据的状态，直到数据成功传输或返回失败。

**异步IO**：应用发送或接收数据后立刻返回，数据写入OS缓存，由OS完成数据发送或接收，并返回成功或失败的信息给应用。



![](http://dl.iteye.com/upload/picture/pic/78320/aea1fdff-576b-3871-bff0-8f8d3032029f.png)



**按照Unix的5个IO模型划分**

* 阻塞IO
* 非阻塞IO
* IO复用
* 信号驱动的IO
* 异步IO



从性能上看，异步IO的性能无疑是最好的。



**各种IO的特点**

* 阻塞IO：使用简单，但随之而来的问题就是会形成阻塞，需要独立线程配合，而这些线程在大多数时候都是没有进行运算的。Java的BIO使用这种方式，问题带来的问题很明显，一个Socket需要一个独立的线程，因此，会造成线程膨胀。
* 非阻塞IO：采用轮询方式，不会形成线程的阻塞。Java的NIO使用这种方式，对比BIO的优势很明显，可以使用一个线程进行所有Socket的监听（select）。大大减少了线程数。

* 同步IO：同步IO保证一个IO操作结束之后才会返回，因此同步IO效率会低一些，但是对应用来说，编程方式会简单。Java的BIO和NIO都是使用这种方式进行数据处理。
* 异步IO：由于异步IO请求只是写入了缓存，从缓存到硬盘是否成功不可知，因此异步IO相当于把一个IO拆成了两部分，一是发起请求，二是获取处理结果。因此，对应用来说增加了复杂性。但是异步IO的性能是所有很好的，而且异步的思想贯穿了IT系统放放面面。




