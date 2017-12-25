# 腾讯Xlog接入指南与踩过的坑  
## 一 为什么要接入日志打印系统？  
相信大家在开发应用的时候，总会遇到bug，这个时候，如果bug是在我们本地开发的过程中发现的，那么我们把手机插入android studio进行联调，就可以马上定位到出错的堆栈，报错的信息。但是，在我们的应用发布出去，给用户使用的时候，如果出现了bug，那么，我们就很难定位出问题的所在。一般来说，我们需要使用错误上报系统，把错误上报给我们，比如腾讯的bugly，阿里的友盟等第三方的错误收集插件。但是很多时候，bugly上报的错误并不能正确的把我们需要的错误信息反馈给我们，或者说，我们无法定位在出现了bug的时候，用户之前的操作是什么。在Xlog还没有上线的时候，我们发现一个bug需要怎么做呢？首先，我们需要联系到这个用户，争取获得用户的配合。然后我们就发一个私包给用户，用户安装后，重现刚才的bug，然后我们把操作日志写入本地的错误日志，中，让用户通过交流软件发给我们，卧槽，这个步奏，想掀凳子有木有？？  
## 二 为什么使用Xlog？ 
在上面我们的最后一个问题暴露出来后，为什么我们不自己写一个日志系统呢？只是把文本写入本地txt文件中，多么容易的事，一个inputStream OutputStream我们不是很玩的很溜么？何必使用别人的东西？？好吧，在回答这个问题的时候，我建议大家先看下这篇文章  

[https://mp.weixin.qq.com/s/cnhuEodJGIbdodh0IxNeXQ](https://mp.weixin.qq.com/s/cnhuEodJGIbdodh0IxNeXQ)  

如果大家嫌这篇文章太长，那么我可以给大家一个简单的总结，就是：**XLog使用流式方式对单行日志进行压缩，压缩加密后写入作为log中间buffer的mmap中。使用xlog方案，除非io损坏或者磁盘没有可用空间，基本可以保证不会丢失任何一行日志**。  

一个优秀的日志打印系统，需要保证：  

**1. 不能把用户的隐私信息打印到日志文件里，不能把日志明文打到日志文件里。**

**2. 不能影响程序的性能。最基本的保证是使用了日志不会导致程序卡顿。**

**3. 不能因为程序被系统杀掉，或者发生了 crash，crash 捕捉模块没有捕捉到导致部分时间点没有日志， 要保证程序整个生命周期内都有日志。**
 
**4. 不能因为部分数据损坏就影响了整个日志文件，应该最小化数据损坏对日志文件的影响。**  

所以，既然有XLog这种好东西，我们在没有把握自己开发一个更加优秀的xlog打印框架，我们就不要重复造轮子了，哈哈。  

## 三 Xlog接入  
**step 1： 引入XLog依赖，在我们主工程的build.gradle中加入**  

    dependencies {
    	compile 'com.tencent.mars:mars-xlog:1.0.6'
    }   

**step 2：开始初始化Xlog环境**  

    private void initXlogEnv(){
        System.loadLibrary("stlport_shared");
        System.loadLibrary("marsxlog");
        final String SDCARD = Environment.getExternalStorageDirectory().getAbsolutePath();
        final String logPath = SDCARD + "/xlogdemo/log";
        final String cachePath = this.getFilesDir() + "/xlogdemo/xlog";
        if (BuildConfig.DEBUG) {
            Xlog.appenderOpen(Xlog.LEVEL_DEBUG, Xlog.AppednerModeAsync, cachePath, logPath, "", "");
            Xlog.setConsoleLogOpen(true);

        } else {
            Xlog.appenderOpen(Xlog.LEVEL_INFO, Xlog.AppednerModeAsync, cachePath, logPath, "MarsSample", "");
            Xlog.setConsoleLogOpen(false);
        }

    }  
## 四：开始踩坑之旅 
ok到了这里，按照Xlog官方给的接入步奏，我们这个时候应该就可以在我们的sdcard根目录上看到logFile.log文件了。ok，我们开开心心的来run一下看看  
![couldn't find "libstlport_shared.so"](https://i.imgur.com/lWqmh3g.png) 
 
卧槽，官方文档又来搞事情了  

**踩坑1：couldn't find "libstlport_shared.so** 

为什么会出现这个bug？在接入Xlog到我们的项目中时，我建了一个demo，跑出来是不会报错的，但是一旦接入到我们工程中，就抛了这个错。在仔细看了官方文档后，知道了，原来xlog因为考虑依赖包的大小，只提供了armeabi和x86_64两种cpu架构。  
![xlog官方文档](https://i.imgur.com/9KCMrsP.png)  

而我们的工程中，又是只支持armeabi-v7a架构。所以这个时候需要怎么办呢？  
解决方法：这个时候，我们只需要先使用armeabi编译打包，然后把生成的**libstlport_shared.so**和**libmasxlog.so**拷贝到我们的工程中，放在对应log的module中即可，像这样  
![xlog手动加载so](https://i.imgur.com/Wx6i0dx.png)  

**踩坑2：没有对应的sdcard没有生成目录文件**  

如果遇到问题，需要检查一下是否在manifest中有申请sdcard写入权限。还有在android6.0以上系统，有没有动态去申请存储权限。  

     <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"></uses-permission>

**踩坑3：同一个进程中，xlog不可以使用日志打印文件。**  

踩这个坑的原因是因为我们在日志上报系统中，会需要重点记录部分用户行为，而这部分日志上报行为是需要上报给服务端的。所以为了尽量减少日志包的大小，同时让为了使重要的日志和其它日志相隔开，我们想把这部分抽离出来一个文件，方便服务端分析和管理。但经过尝试后，这个行为xlog是不允许的，首先xlog会抛错，错误信息是当前的日志文件已经被打开。  
分析：其实这种一个进程只能写入一个文件是可以理解的，因为如果一个进程多文件的话，xlog需要频繁的去控制文件流的输出和关闭，这样会导致性能问题，也可能会造成日志丢失。 
 
**踩坑4:多进程xlog日志打印**  

xlog是支持多进程打印的，但是多进程打印需要制定每一个进程对应一个文件。在这里，我建议使用进程的名字作为一个文件夹，然后再放入对应的日志文件。这样便于我们以后日志的管理。当我们开启更多进程的时候，也会自动生成，不用去一一匹配。