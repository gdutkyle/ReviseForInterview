# HTTP 基础   
## 一什么是URL？  
### 1.1URL和URI有什么区别？  
URI：Uniform resource identifer，统一资源表示符，用来表示唯一的一个资源。Web上每一个可用的资源都是由一个URI来定位的，URI一般由三个部分组成：  

1 访问资源的命名机制；  

2 存放资源的主机名；  

3 资源自身的名称，由路径表示，着重强调于资源  

URL（Uniform Resource Locator），统一资源定位符，是用来表示访问后台资源的位置和访问资源的方法。一般来说  

**URL = <协议>://<主机>:<端口号>/<路径>**  

也即是说，URL可以用来标识一个资源，而且也可以用来标识要如何去获取这个资源。所以，从集合的角度来说，URI其实是包含了URL的。  

最近在知乎看到一个很简单明了的解释方法。
[https://www.zhihu.com/question/21950864/answer/154309494](https://www.zhihu.com/question/21950864/answer/154309494)  
简单来总结下这个作者的解释，URI其实就是把这个资源独一无二的标识出来。比如我们可以用身份证号码123456789去标识张三这个人。也可以通过中国现有的户籍制度：XX省XX市XX区XX街道XX小区XX栋XX号张三。我们可以简单的认为，身份证号就是张三这个人（资源）的URI，而上面的地址，就是张三这个人（资源）的URL。  
## 二 HTTP协议   
### 2.1 HTTP操作流程  
当用户在浏览器输入[https://github.com/gdutkyle/ReviseForInterview](https://github.com/gdutkyle/ReviseForInterview)并按下回车键的时候，浏览器究竟做了什么？  

1 浏览器分析连接指向的URL；  

2 浏览器向DNS请求解析github.com的ip地址；  

3 DNS解析出github的ip地址为：192.30.255.112；  

4 浏览器和github服务器建立TCP连接；  

5 浏览器发出GET命令；  

6 服务器返回指定的文件给浏览器；  

7 TCP连接断开；  

8 浏览器根据返回的文件，显示在浏览器上。  

在上面的这个流程中，我们可以看到：  

1 HTTP使用了TCP作为传输协议来保证数据的可靠传输，HTTP不需要考虑数据丢弃或者重传的实现机制。而**HTTP是面向无连接**的，也即是，在浏览器和服务端发起HTTP连接的时候，不需要先建立连接。  

2 **HTTP是无状态的**，也就是客户端第二次访问同一个服务器上的同一个资源时，服务端的响应流程跟第一次相同，所以服务端不会记住曾经访问的客户，也不会记录这个客户对这个资源访问了多少次。  

3 灵活：HTTP允许传输任何类型的数据对象  

4 简单快速：客户端向服务端发送请求时，只需要传入请求路径和方法。     

### 2.2 HTTP1.1的持续连接  
在前面2.1的时候提到，因为HTTP是面向无连接的，也就是说，服务端和客户端的每次通讯都要连接、断开、连接、断开，当一个页面需要下载多个资源的时候，就会使服务端产生很重的负担，因此，在http1.1的时候，引入了持续连接的概念。  

**持续连接**  
服务端在对这个客户端的请求发生响应后，不马上断开连接，而是保持一段时间内连接状态。使得同一个客户和该服务器可以继续在这条连接上传输后续的请求报文和响应报文。持续连接主要有两种工作方式：

*1 非流水线方式*  
客户端在收到上一个响应后，才可以进行下一个发送操作，因此，在tcp连接建立后，客户每访问一次对象都要用去一个RTT时间。虽然这样可以使得http的请求连接少一个RTT时间。但缺点是，服务端每响应完一个请求后，就会处于空闲状态，浪费服务端的资源。  

*2 流水线方式*  
客户端在收到HTTP响应报文之前，就可以联系的发送新的请求报文，当请求一个个到达服务端后，服务端就可以连续返回响应报文。因此，使用流水线传输，只需要使用一个RTT时间，就可以完成整个操作，提高了文档下载的效率。  
### 2.3 HTTP报文结构  

**1 请求报文**：客户端向服务端发送请求报文  
（1）开始行  
开始行，在请求报文中叫做请求行，在响应报文中叫做状态行。  
请求行=方法+URL+HTTP版本号  
（2）首部行：用来说明浏览器、服务器或者报文主体的一些信息。  
（3）实体主体，可以添加任何其他的数据  
**2 响应报文**：服务端返回客户端的数据：  
（1）状态行：http版本号+状态码  
（2）消息报头（date content-type content-length）  
（3）空行  
（4）响应正文  
### 2.4 Get和Post方法的区别  
（1）Get方法的请求数据会附加在请求的url后面，而post的请求是把数据放在请求报文的body中。Get方法的数据会在浏览器中显示出来，而post的数据则不会。  

（2）Get方法在特定的浏览器和服务器上有特殊的长度要求，如IE对url长度的限制在2083个字节；而post的数据，理论上不存在大小的限制。但是严格上说，HTTP并没有规定get的长度限制，只是浏览器的限制，并不是本身协议的限制！  

（3）安全性：Get的数据因为会出现在浏览器的地址栏，所以用户数据可能会被浏览器缓存，容易被其他人拿到敏感信息；其次，Get方法可能会造成cross-site request forgery攻击。  

###  2.5 HTTP状态码    

状态代码有三位数字组成，第一个数字定义了响应的类别，共分五种类别:

1xx：指示信息--表示请求已接收，继续处理  

2xx：成功--表示请求已被成功接收、理解、接受  

3xx：重定向--要完成请求必须进行更进一步的操作  

4xx：客户端错误--请求有语法错误或请求无法实现  

5xx：服务器端错误--服务器未能实现合法的请求  

*常见状态码：*

200 OK                        //客户端请求成功  

203 Non-Authoritative Information //非授权信息。请求成功。但返回的meta信息不在原始的服务器，而是一个副本  

300 Multiple Choices          //多种选择。请求的资源可包括多个位置，相应可返回一个资源特征与地址的
列表用于用户终端（例如：浏览器）选择  

301 Moved Permanently         //永久移动。请求的资源已被永久的移动到新URI  

400 Bad Request               //客户端请求有语法错误，不能被服务器所理解  

401 Unauthorized              //请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用 
  
403 Forbidden                 //服务器收到请求，但是拒绝提供服务  

404 Not Found                 //请求资源不存在，eg：输入了错误的URL  

500 Internal Server Error     //服务器发生不可预期的错误  

503 Server Unavailable        //服务器当前不能处理客户端的请求，一段时间后可能恢复正常  


  
 


