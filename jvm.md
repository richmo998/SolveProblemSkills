jvm问题排查：
//=========================//
## 生产web应用程序不可用排查思路
//=========================//

先检查业务日志，如果应用服务已经不能输出日志了，则需要分析jvm日志
top 命令检查服务是cpu过高还是内存使用过高
  内存占用过高
  1.堆内存分配不合理，需要调整jvm相关参数
  2.程序写得不严谨，出现大量对象没有回收，导致内存过高
  
  cpu占用过高
  1.判断当前的业务量是否真为高并发海量访问场景，是则可通过负载均衡进行水平扩展
  2.程序问题,多线程阻塞或是系统资源未释放等
  3.获取GC过于频繁或触发fullGC
  
  内存和cpu占用都高
  二者都很高，一般就是多种问题同时引发
  
根据top命令的初步判断进行细化排查（现在一般都是一台服务器一个应用）
top -Hp {pid} :查看进程的线程使用情况
ps huH p  {pid}  | wc -l ：统计进行的总线程数

// java -Xms32M -Xmx32M -XX:+HeapDumpOnOutOfMemoryError -cp xxx.jar com.test.Test1 :自定义进程的堆内存大小，以及没有内存后可导出jvm日志信息
ps -ef|grep tomcat : 获取到应用的pid进程号

top -Hp {pid} :查看进程的线程使用情况

jstack -l xxx :直接查看线程的堆栈信息，一般用于初步查看（生产的jvm信息一般会很多很大）

jstack -l xxx > 1.txt 可采用导出成其他问价进行排查（一般生产会进行重启服务，所以需要导出事后进行分析）

jmap -dump:file=xxx pid 将进程jvm信息dump为xxx文件

jhat xxx :jhat打开dump的日志信息(jhat -J-Xmx1024m D:/javaDump.hprof 指定内存大小)
通过 http://localhost:7000/ 进行浏览器访问（主要排查对象引用未释放场景）

如果是多线程问题则根据dump的堆栈信息，进行详细排查，主要关注以下信息
    死锁，Deadlock（重点关注） 
    执行中，Runnable   
    等待资源，Waiting on condition（重点关注） 
    等待获取监视器，Waiting on monitor entry（重点关注）
    暂停，Suspended
    对象等待中，Object.wait() 或 TIMED_WAITING
    阻塞，Blocked（重点关注）  
    停止，Parked   

printf "%x\n" xxxx :获取线程id的十六进制码

jstack pid | grep 线程id的十六进制码  ：在线程的堆栈信息中搜索线程的信息，可以定位到当前代码行

常见的jvm问题

1、stackoverflow：

  每当java程序启动一个新的线程时，java虚拟机会为他分配一个栈，java栈以帧为单位保持线程运行状态；当线程调用一个方法是，jvm压入一个新的栈帧到这个线程的栈中，只要这个方法还没返回，这个栈帧就存在。 
  如果方法的嵌套调用层次太多(如递归调用),随着java栈中的帧的增多，最终导致这个线程的栈中的所有栈帧的大小的总和大于-Xss设置的值，而产生生StackOverflowError溢出异常。

2、outofmemory：

    2.1、栈内存溢出

    java程序启动一个新线程时，没有足够的空间为改线程分配java栈，一个线程java栈的大小由-Xss设置决定；JVM则抛出OutOfMemoryError异常。

    2.2、堆内存溢出

    java堆用于存放对象的实例，当需要为对象的实例分配内存时，而堆的占用已经达到了设置的最大值(通过-Xmx)设置最大值，则抛出OutOfMemoryError异常。

    2.3、方法区内存溢出

    方法区用于存放java类的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。在类加载器加载class文件到内存中的时候，JVM会提取其中的类信息，并将这些类信息放到方法区中。 
    当需要存储这些类信息，而方法区的内存占用又已经达到最大值（通过-XX:MaxPermSize）；将会抛出OutOfMemoryError异常对于这种情况的测试，基本的思路是运行时产生大量的类去填满方法区，直到溢出。这里需要借助CGLib直接操作字节码运行时，生成了大量的动态类。



//=========================//
## jvm分析工具介绍 
//=========================//


https://www.cnblogs.com/duanxz/p/4890475.html
一、jdk工具之jps（JVM Process Status Tools）命令使用

二、jdk命令之javah命令(C Header and Stub File Generator)

三、jdk工具之jstack(Java Stack Trace)

四、jdk工具之jstat命令(Java Virtual Machine Statistics Monitoring Tool)

四、jdk工具之jstat命令2(Java Virtual Machine Statistics Monitoring Tool)详解

五、jdk工具之jmap（java memory map）、 mat之四--结合mat对内存泄露的分析

六、jdk工具之jinfo命令(Java Configuration Info)

七、jdk工具之jconsole命令(Java Monitoring and Management Console)

八、jdk工具之JvisualVM、JvisualVM之二--Java程序性能分析工具Java VisualVM

九、jdk工具之jhat命令(Java Heap Analyse Tool)

十、jdk工具之Jdb命令(The Java Debugger)

十一、jdk命令之Jstatd命令(Java Statistics Monitoring Daemon)

十一、jdk命令之Jstatd命令(Java Statistics Monitoring Daemon)

十二、jdk工具之jcmd介绍（堆转储、堆分析、获取系统信息、查看堆外内存）

十三、jdk命令之Java内存之本地内存分析神器：NMT 和 pmap

MAC下载独立MAT内存分析工具

https://blog.csdn.net/mahl1990/article/details/79298616

MAT独立分析工具使用说明 https://blog.51cto.com/wwdhks/2121247


//=========================//
##  排查类加载器之-ClassNotFound Exception 
//=========================//
- 问题背景：
   进行大数据sqoop数据同步项目微服务化，将原有的jmx协议服务，转换成restful服务或dubbo服务，以便实现服务使用方语言多样化以及
sqoop微服务可以灵活使用nginx进行负载均衡以实现服务的高可用高并发。
   sqoop任务需要通过RunJar的方式将任务提交到yarn进行执行，所以需要在原有项目启动的情况下，使用ClassLoader加载机制将工具包加载，
并调用其中的main方法进行任务提交。
   在使用 URLClassLoader(classPath.toArray(new URL[0]))进行加载后，一直报出 ClassNotFound Exception
```
//外部引用jar配置信息：sqoop.jobjar=/Users/richmo/work/code-resources/bdp-dht/bdp-dht-sqoop/target/bdp-dht-sqoop-2.0.0.jar

            File file = new File("/Users/richmo/work/code-resources/bdp-dht/bdp-dht-sqoop/target/bdp-dht-sqoop-2.0.0.jar");
            URL url = file.toURI().toURL();
            List<URL> list = new ArrayList<>();
            list.add(url);
            URLClassLoader urlClassLoader = new URLClassLoader(list.toArray(new URL[0]));
            
            String mainClassName="org.spring.springboot.sqoop.SqoopApi";
            Thread.currentThread().setContextClassLoader(urlClassLoader);
            Class<?> mainClass = Class.forName(mainClassName, true, urlClassLoader);
            
            System.out.println("URLClassLoader加载Class.forName："+mainClass);
```


- 排查步骤：
```
   //在restful测试接口中加入三种加载类方式对比
   
           try {
           //当前环境加载
               Class<?> aClass = Class.forName("org.spring.springboot.sqoop.SqoopApi");
               Method main = aClass.getMethod("main", String[].class);
   
               System.out.println("Class.forName:加载========"+aClass);
   
   
             //自定义从文件下加载
               DiskClassLoader diskClassLoader = new DiskClassLoader("/Users/richmo/work/github/springboot-learning-example/springboot-restful/target/classes/org/spring/springboot/sqoop");
               Thread.currentThread().setContextClassLoader(diskClassLoader);
               Class<?> mainClass = Class.forName("org.spring.springboot.sqoop.SqoopApi", true, diskClassLoader);
               System.out.println("自定义类:加载========"+mainClass);
   
   
              //指定类加载器，以及jar路径（此处正式模拟原来环境的加载class方式）
        
               myClassLoader myClassLoader = new myClassLoader();
               ClassLoader classLoader = myClassLoader.createClassLoader(new File("/Users/richmo/work/code-resources/bdp-dht/bdp-dht-sqoop/target/bdp-dht-sqoop-2.0.0.jar"),
                       new File("~/work/github/springboot-learning-example/springboot-restful/target"));
   
               Class<?> aClass1 = classLoader.loadClass("org.spring.springboot.sqoop.SqoopApi");
   
               System.out.println("createClassLoader 加载："+aClass1);
   
   
           } catch (Exception e) {
               e.printStackTrace();
           }
```
   1.对比运行环境
     --1）.项目未改成springboot启动的情况下，运行没有问题，RunJar可以正常提交yarn任务，证明提供的jar是没有问题的；
     --2）.项目改成springboot后，再idea中直接运行，然后postman请求测试方法，可以正常提交yarn任务；
     --3）.将项目打成可运行jar后，启动后再进行postman请求后，报出ClassNotFound Exception ： org.spring.springboot.sqoop.SqoopApi
   通过运行方式以及环境对比，
      初步定为springboot内置tomcat为运行容器，是不是tomcat的类加载机制导致，无法加载指定jar下的class文件？
      但是却无法解释在idea工具中启动项目，又是可正常运行的，这就自相矛盾。
      
      进一步使用jconsole在三种运行环境下，将JVM加载的类包信息拷贝出来进行对比，发现只有使用java -jar方式启动springboot项目的时候少加载了很多
   jar包以及classes文件，所以认为是否为java -jar方式启动项目是否存在有类加载丢失情况？
   
      debug loadClass(..)源码：着重对比idea直接启动下能加载到class，java -jar 启动下又找不到class
      在idea启动springboot项目后，debug发现当前URLClassLoader加载器并没有org.spring.springboot.sqoop.SqoopApi该类（按理，已指定jar路径，就应该从当前类加载器可以findClass）
   而是调用到了父加载? 这就证明，当前加载出的org.spring.springboot.sqoop.SqoopApi这个类并不是从我们对应的jar中获取到的。
      
      继续查看hadoop中RunJar的源码，发现加载/Users/richmo/work/code-resources/bdp-dht/bdp-dht-sqoop/target/bdp-dht-sqoop-2.0.0.jar这个包之前做了unJar的操作，
   当时我就决定也模拟这个解压操作，然后从解压的文件中，利用文件夹的访问来加载。结果在我的测试类中我依然失望的发现还是类找不到。
      反思，为什么从自己解压的文件中加载不到这个类，然后我把路径切换到idea自动编译生产的target目录下，惊喜出现了，类找到了。
      至此我开始怀疑，我解压的这些文件真的没有org.spring.springboot.sqoop.SqoopApi这个类。
      所以我模拟解压了jar,并去这个解压的类中找到SqoopApi这个类，结果惊奇的发现这个类其实是cn.wonhigh.dc.client.sqoop.SqoopApi
 
 ## 最终结果     
      有木有想吐血的冲动，这就是类名相同，但是累路劲完全一样的两个，所以一直在当前jar包中加载不出传入的类的原因，
      主要是由于，配置文件中的jar是以前使用的一个工具包，所以当时没有去怀疑这个包有问题；
      现在我将项目迁移为springboot类型，就修改了工程路径，调用的时候是通过当前工程org.spring.springboot.sqoop.SqoopApi这个类调用的，但是真正提交yarn任务
      却是用了原来就的jar包中的类，导致一直报org.spring.springboot.sqoop.SqoopApi这类无法加载。
   
   总结：其实，发现问题的原因是很简单，却花了很长时间。原因是，
        第一：没有去质疑配置文件/Users/richmo/work/code-resources/bdp-dht/bdp-dht-sqoop/target/bdp-dht-sqoop-2.0.0.jar是有问题的（固化思维，认为一直运行都是正常的）；
        第二：应该抓住问题的本质，既然包这个jar包没有这个类就应该直接解压这个包，然后去找org.spring.springboot.sqoop.SqoopApi这个类就能立马发现问题。
     
      
      
      以下是debug过程，RunJar的源码，发现的秘密：
   ```
    //此处调用前，是先初始化了一个SqoopApi对象，并将丢向传入了，执行体（当前这个类对象为： org.spring.springboot.sqoop.SqoopApi）
    final SqoopApi sqoopApi = SqoopApi.getSqoopApi();
    
        try {
            SqoopApiExample.testSqoopApiExecuteSucceed(sqoopApi);

        } catch (MaxHadoopJobRequestsException e) {
            e.printStackTrace();
        }
        
      ...
      //内部执行方法
      JobExecutionResult jobExecutionResult = sqoopApi.execute("succeedJob", command, paras, options);
      
      ...
      //执行提交，此处需要特别注意 SqoopApi.class这个参数，实际就是org.spring.springboot.sqoop.SqoopApi
      submitMrJobjar(sqoopJobJarPath, SqoopApi.class, sqoopApiArgs);
      
      //最终调用RunJar.main(allArgs)提交到yarn集群
        /**
           * This is to submit MR job to Yarn cluster.
           */
          private void submitMrJobjar(String jobJar, Class mrJobMainClass, String[] sqoopApiArgs) {
              File file = new File(jobJar);
              if (file.exists() && file.isFile()) {
                  String[] runJarArgs = {jobJar, mrJobMainClass.getName()};
                  String[] allArgs = new String[runJarArgs.length + sqoopApiArgs.length];
                  System.arraycopy(runJarArgs, 0, allArgs, 0, runJarArgs.length);
                  System.arraycopy(sqoopApiArgs, 0, allArgs, runJarArgs.length, sqoopApiArgs.length);
                  try {
                      RunJar.main(allArgs);
                  } catch (Throwable t) {
                      log.error("Error in invoking RunJar.main", t);
                  }
              } else {
                  String noJobJarError = String.format(
                          "Couldn't find job jar of '%s', please put the sqoop job jar into this path: [%s]",
                          jobJar, jobJar);
                  log.error(noJobJarError);
                  throw new IllegalArgumentException(noJobJarError);
              }
          }
      
      
      //RunJar的核心执行代码如下
       /** Run a Hadoop job jar.  If the main class is not in the jar's manifest,
         * then it must be provided on the command line. */
        public static void main(String[] args) throws Throwable {
          new RunJar().run(args);
        }
      
        public void run(String[] args) throws Throwable {
          String usage = "RunJar jarFile [mainClass] args...";
      
          if (args.length < 1) {
            System.err.println(usage);
            System.exit(-1);
          }
      
          int firstArg = 0;
          String fileName = args[firstArg++];
          File file = new File(fileName);
          if (!file.exists() || !file.isFile()) {
            System.err.println("Not a valid JAR: " + file.getCanonicalPath());
            System.exit(-1);
          }
          String mainClassName = null;
      
          JarFile jarFile;
          try {
          //将传入的jar的配置路径转化为JarFile类，此处加载的路径为：/Users/richmo/work/code-resources/bdp-dht/bdp-dht-sqoop/target/bdp-dht-sqoop-2.0.0.jar
          
            jarFile = new JarFile(fileName);
          } catch(IOException io) {
            throw new IOException("Error opening job jar: " + fileName)
              .initCause(io);
          }
      
          //检查该类的Manifest查看主执行方法
          
          Manifest manifest = jarFile.getManifest();
          if (manifest != null) {
            mainClassName = manifest.getMainAttributes().getValue("Main-Class");
          }
          jarFile.close();
      
          if (mainClassName == null) {
            if (args.length < 2) {
              System.err.println(usage);
              System.exit(-1);
            }
            mainClassName = args[firstArg++];
          }
          mainClassName = mainClassName.replaceAll("/", ".");
      
          File tmpDir = new File(System.getProperty("java.io.tmpdir"));
          ensureDirectory(tmpDir);
      
          final File workDir;
          try { 
            workDir = File.createTempFile("hadoop-unjar", "", tmpDir);
          } catch (IOException ioe) {
            // If user has insufficient perms to write to tmpDir, default  
            // "Permission denied" message doesn't specify a filename. 
            System.err.println("Error creating temp dir in java.io.tmpdir "
                               + tmpDir + " due to " + ioe.getMessage());
            System.exit(-1);
            return;
          }
      
          if (!workDir.delete()) {
            System.err.println("Delete failed for " + workDir);
            System.exit(-1);
          }
          ensureDirectory(workDir);
      
          ShutdownHookManager.get().addShutdownHook(
            new Runnable() {
              @Override
              public void run() {
                FileUtil.fullyDelete(workDir);
              }
            }, SHUTDOWN_HOOK_PRIORITY);
      
      
          //将这个jar文件进行解压到目标文件夹
          unJar(file, workDir);
      
          //这里就是创建类加载器，实际返回的是一个URLClassLoader
          ClassLoader loader = createClassLoader(file, workDir);
      
          Thread.currentThread().setContextClassLoader(loader);
          
          //mainClassName这里实际就是：org.spring.springboot.sqoop.SqoopApi
          //上述所说的异常都是从这里抛出的
          Class<?> mainClass = Class.forName(mainClassName, true, loader);
          
          Method main = mainClass.getMethod("main", new Class[] {
            Array.newInstance(String.class, 0).getClass()
          });
          String[] newArgs = Arrays.asList(args)
            .subList(firstArg, args.length).toArray(new String[0]);
          try {
            main.invoke(null, new Object[] { newArgs });
          } catch (InvocationTargetException e) {
            throw e.getTargetException();
          }
        }
```

