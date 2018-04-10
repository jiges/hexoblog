---
title: JVM启动流程(源码分析)
date: 2017-12-15 08:17:35
tags: [OpenJDK,jvm]
categorys: OpenJDK
---
{% asset_img 积累.jpg 不要小看那一丢丢的知识，一天天的积累，量变终将质变 %}
<!-- more -->
> 众所周知，`Java`程序的入口是在`main`方法中，然而在执行`main`方法之前，虚拟机是如何启动的？是如何找到`main`方法的？又是如何将参数传进来的？

# 前言
`java`程序员都知道，是通过`java`命令来运行程序的，运行程序时可以附加一些参数。具体的参数可以参考[【Java平台标准版工具参考】](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/java.html)。`java`命令标准的命令格式是：`java [options] classname [args]` 或者 `java [options] -jar filename [args]`
用以下示例程序来调试`JVM`：
```java
public class HelloJNI {
		
	public static void main(String[] args) {
        System.out.println("Hello Java");
    }
}
//启动命令：
//java -d64 -server -Xss512k -cp "/home/ccr/jvm/javatest/" HelloJNI hello
```

# JVM启动流程
`JVM`的启动流程，可以分为以下几步：
* `JVM`运行环境的设置和检查。
* 通过`CreateExecutionEnvironment`函数查找`JAVA_DLL`动态库是否存在，能不能访问。并设置一些变量。
* 加载动态库，将动态库中的一些函数链接至本地变量。
* 解析`[options]`和`[args]`参数。
* 新建线程初始化虚拟机。
* 加载并执行主类。

# 源码分析
整个过程的入口是`/home/ccr/jvm/openjdk/jdk/src/share/bin/main.c`文件中的`main`函数:
```c++
int main(int argc, char **argv)
{
    int margc;
    char** margv;
    const jboolean const_javaw = 0;//是否以javaw的方式启动
    margc = argc;//参数格式
    margv = argv;//接收命令行参数
    return JLI_Launch(margc, margv,
			sizeof(const_jargs) / sizeof(char *), // 1
			const_jargs, //0x0
			sizeof(const_appclasspath) / sizeof(char *), // 1
			const_appclasspath,  //0x0
			FULL_VERSION, // 1.8.0-internal-debug-ccr_2017_11_08_15_43-b00
			DOT_VERSION, // "1.8"
			(const_progname != NULL) ? const_progname : *margv,  //const_progname=java
			(const_launcher != NULL) ? const_launcher : *margv, //const_launcher = openjdk
			(const_jargs != NULL) ? JNI_TRUE : JNI_FALSE, //const_jargs = 0x0
			const_cpwildcard, const_javaw, const_ergo_class); 
			// const_cpwildcard = 1 '\\001'  const_javaw = 0  const_ergo_class = 0
}
```
`JLI_Launch`函数定义在`/home/ccr/jvm/openjdk/jdk/src/share/bin/java.c`文件中。该函数是`jvm`启动的前半部分，加载动态库。解析参数。
```c++
int JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* ergonomics class policy */
)
{
	//enum LaunchMode {               // cf. sun.launcher.LauncherHelper
    //LM_UNKNOWN = 0,
    //LM_CLASS,
    //LM_JAR
	//};
	//启动模式，jar启动，还是main class 启动
    int mode = LM_UNKNOWN;
	//存储启动类类名
    char *what = NULL;
	//存储class path
    char *cpath = 0;
	//主类引用
    char *main_class = NULL;
    int ret;
	//加载JAVA_DLL时，将一些函数链接到此变量中
    InvocationFunctions ifn;
    jlong start, end;
	// JAVA_DLL动态库路径
    char jvmpath[MAXPATHLEN];
	// jre路径
    char jrepath[MAXPATHLEN];
	// jvm.cfg 文件路径
    char jvmcfg[MAXPATHLEN];

    _fVersion = fullversion;
    _dVersion = dotversion;
    _launcher_name = lname;
    _program_name = pname;
    _is_java_args = javaargs;
    _wc_enabled = cpwildcard;
    _ergo_policy = ergo;

	//根据环境变量中的“_JAVA_LAUNCHER_DEBUG”参数来设置是否打印调试信息
	//_launcher_debug = false
    InitLauncher(javaw);
	//打印一些调试信息
    DumpState();
    if (JLI_IsTraceLauncher()) {//判断_launcher_debug变量
        int i;
        printf("Command line args:\n");
        for (i = 0; i < argc ; i++) {
            printf("argv[%d] = %s\n", i, argv[i]);
        }
        AddOption("-Dsun.java.launcher.diag=true", NULL);
    }

    /*
     * Make sure the specified version of the JRE is running.
     *
     * There are three things to note about the SelectVersion() routine:
     *  1) If the version running isn't correct, this routine doesn't
     *     return (either the correct version has been exec'd or an error
     *     was issued).
     *  2) Argc and Argv in this scope are *not* altered by this routine.
     *     It is the responsibility of subsequent code to ignore the
     *     arguments handled by this routine.
     *  3) As a side-effect, the variable "main_class" is guaranteed to
     *     be set (if it should ever be set).  This isn't exactly the
     *     poster child for structured programming, but it is a small
     *     price to pay for not processing a jar file operand twice.
     *     (Note: This side effect has been disabled.  See comment on
     *     bugid 5030265 below.)
     */
	 //需要指定版本的jre才能运行，通过命令行“-version:”命令，或者读取jar文件的mainfest文件
	 //查看链接详情
    SelectVersion(argc, argv, &main_class);

	//准备环境，搜索jrepath和jvmpath，具体实现在下面找
    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               jvmcfg,  sizeof(jvmcfg));
	//动态链接库中的一些函数
	//创建虚拟机
    ifn.CreateJavaVM = 0;
	//获取默认初始化参数
    ifn.GetDefaultJavaVMInitArgs = 0;

    if (JLI_IsTraceLauncher()) {
        start = CounterGet();
    }

	//加载JAVA_DLL到内存，并将动态库中的函数链接至ifn.CreateJavaVM，ifn.CreateJavaVM，ifn.GetCreatedJavaVMs
    if (!LoadJavaVM(jvmpath, &ifn)) {
        return(6);
    }

    if (JLI_IsTraceLauncher()) {
        end   = CounterGet();
    }

    JLI_TraceLauncher("%ld micro seconds to LoadJavaVM\n",
             (long)(jint)Counter2Micros(end-start));

    ++argv;
    --argc;

    if (IsJavaArgs()) {
        /* Preprocess wrapper arguments */
        TranslateApplicationArgs(jargc, jargv, &argc, &argv);
        if (!AddApplicationOptions(appclassc, appclassv)) {
            return(1);
        }
    } else {
        /* Set default CLASSPATH */
        cpath = getenv("CLASSPATH");
        if (cpath == NULL) {
            cpath = ".";
        }
		//设置CLASSPATH
		//调用AddOption函数 将"-Djava.class.path=."字符串插入到JavaVMOption *options变量中
        SetClassPath(cpath);
    }

    /* Parse command line options; if the return value of
     * ParseArguments is false, the program should exit.
     */
	 //解析所有option参数，并将其存到JavaVMOption *options变量中
    if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath))
    {
        return(ret);
    }

    /* Override class path if -jar flag was specified */
	//-jar模式
    if (mode == LM_JAR) {
        SetClassPath(what);     /* Override class path */
    }

    /* set the -Dsun.java.command pseudo property */
	//设置用来向java main方法暴露类名和参数，或者为java main存储方法名和参数
	//"-Dsun.java.command=HelloJNI hello"
    SetJavaCommandLineProp(what, argc, argv);

    /* Set the -Dsun.java.launcher pseudo property */
    SetJavaLauncherProp();

    /* set the -Dsun.java.launcher.* platform properties */
    SetJavaLauncherPlatformProps();

	//处理splash screen，开启新线程去初始化JVM
    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```  
## 环境准备动态库查找
准备环境，搜索`jrepath`和`jvmpath`的实现在`/home/ccr/jvm/openjdk/jdk/src/solaris/bin/java_md_solinux.c`文件中，该函数做了一些参数的检查，如`-d32`和`-d64`，如果和平台支持的数据模型不符，就会退出。然后通过`GetJREPath`函数，获取`JRE`路径，在读取`jvm.cfg`文件。改文件记录了`jvm`以何种类型启动(`-server`、`-client`等)，`-Server VM`启动慢，但是一旦运行起来后，性能将会有很大的提升。`-Client VM`启动快，运行相对较慢。文件记录的默认是`-server`类型。然后在获取`JAVA_DLL`的路径，在`Linux`上表现为`libjava.so`路径,在`windows`上是`jvm.dll`路径。另外该函数还检查了`LD_LIBRARY_PATH`环境变量，用户也可以通过这个环境变量来指定动态库文件。
```c++
void CreateExecutionEnvironment(int *pargc, char ***pargv,
                           char jrepath[], jint so_jrepath,
                           char jvmpath[], jint so_jvmpath,
                           char jvmcfg[],  jint so_jvmcfg){
	//通过Linux的“/proc/self/exe”机制读取进程可执行文件（java）的绝对路径并将其塞进execname变量
	//“/home/ccr/jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/bin/java”
	SetExecname(*pargv); 
	{		
		char *arch        = (char *)GetArchPath( ( 8 * sizeof(void*)) ); //获取系统的属性  amd64
		char * jvmtype    = NULL;
		int  argc         = *pargc;
		char **argv       = *pargv; //命令行参数
		int running       = ( 8 * sizeof(void*)); // 64
	  
		int wanted        = running;      /* What data mode is being
                                           asked for? Current model is
                                           fine unless another model
                                           is asked for */
		jboolean mustsetenv = 0;
		char *runpath     = ((void *)0); /* existing effective LD_LIBRARY_PATH setting */
		char* new_runpath = ((void *)0); /* desired new LD_LIBRARY_PATH string */
		char* newpath     = ((void *)0); /* path on new LD_LIBRARY_PATH */
		char* lastslash   = ((void *)0);
		char** newenvp    = ((void *)0); /* current environment */

		char** newargv    = ((void *)0);
		int    newargc    = 0;
		
		
	  /*
       * Starting in 1.5, all unix platforms accept the -d32 and -d64
       * options.  On platforms where only one data-model is supported
       * (e.g. ia-64 Linux), using the flag for the other data model is
       * an error and will terminate the program.
       */
	   //从1.5版本开始，所有Unix平台接收-d32和-d64操作，一种平台只能支持一种数据模型，用另外一种就会报错
	   //这一整段代码目的是：看看命令行参数中有没有-d32和-d64，有就记录下来，然后删除这个参数
	   { /* open new scope to declare local variables */
        int i;

        newargv = (char **)JLI_MemAlloc((argc+1) * sizeof(char*));
        newargv[newargc++] = argv[0];

        /* scan for data model arguments and remove from argument list;
           last occurrence determines desired data model */
		//扫描所有参数，删除数据模型的参数。strcmp是字符串比较
        for (i=1; i < argc; i++) {

          if (strcmp(( argv[i] ), ( "-J-d64" )) == 0 || strcmp(( argv[i] ), ( "-d64" )) == 0) {
            wanted = 64;
            continue;
          }
          if (strcmp(( argv[i] ), ( "-J-d32" )) == 0 || strcmp(( argv[i] ), ( "-d32" )) == 0) {
            wanted = 32;
            continue;
          }
          newargv[newargc++] = argv[i];
		  //false，判断_is_java_args变量，JLI_Launch中初始化，由main传入
          if (IsJavaArgs()) {
            if (argv[i][0] != '-') continue;
          } else {
			//java 标准参数格式java [ options ] classname [ args ] 或者 java [ options ] -jar 文件名 [ args ]
			//-classpath或-cp出现说明options结束，剩下的是类名和参数
            if (strcmp(( argv[i] ), ( "-classpath" )) == 0 || strcmp(( argv[i] ), ( "-cp" )) == 0) {
              i++;
              if (i >= argc) break;
              newargv[newargc++] = argv[i];
              continue;
            }
			//options结束的标志，除了-cp 后面的参数
            if (argv[i][0] != '-') { i++; break; }
          }
        }
			
		/* copy rest of args [i .. argc) */
		//拷贝剩余的参数
        while (i < argc) {
          newargv[newargc++] = argv[i++];
        }	
		newargv[newargc] = NULL;

        /*
         * newargv has all proper arguments here
         */

        argc = newargc;
        argv = newargv;
      }		
	   /* If the data model is not changing, it is an error if the
         jvmpath does not exist */
      if (wanted == running) {
        /* Find out where the JRE is that we will be using. */
		//获取JRE路径
        if (!GetJREPath(jrepath, so_jrepath, arch, 0) ) {
          JLI_ReportErrorMessage("Error: Could not find Java SE Runtime Environment.");
          exit(2);
        }
		//jvm.cfg 路径  "/home/ccr/jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/lib/amd64/jvm.cfg"
        snprintf(jvmcfg, so_jvmcfg, "%s%slib%s%s%sjvm.cfg",
                     jrepath, "/", "/",  arch, "/");
        /* Find the specified JVM type */
		//读取jvm.cfg的配置文件，填充knownVMs变量，knownVMs数据结构在下面
        if (ReadKnownVMs(jvmcfg, 0) < 1) {
          JLI_ReportErrorMessage("Error: no known VMs. (check for corrupt jvm.cfg file)");
          exit(1);
        }

        jvmpath[0] = '\0';
		//检查JVM的类型，如果命令行没有给出类型（指的是 -client 或者 -server参数），
		//则会检查"JDK_ALTERNATE_VM"环境变量或者命令行'-XXaltjvm='参数。
		//如果都没有指定，则使用默认的类型（定义在jvm.cfg配置文件中）
        jvmtype = CheckJvmType(pargc, pargv, 0);
        if (strcmp(( jvmtype ), ( "ERROR" )) == 0) {
            JLI_ReportErrorMessage("Error: could not determine JVM type.");
            exit(4);
        }

		//获取JAVA_DLL路径  "/home/ccr/jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/lib/amd64/server/libjvm.so"
        if (!GetJVMPath(jrepath, jvmtype, jvmpath, so_jvmpath, arch, 0 )) {
          JLI_ReportErrorMessage("Error: missing `%s' JVM at `%s'.\nPlease install or use the JRE or JDK that contains these missing components.", jvmtype, jvmpath);
          exit(4);
        }
        /*
         * we seem to have everything we need, so without further ado
         * we return back, otherwise proceed to set the environment.
         */
		//测试所有需要的环境变量是否被设置，主要检查"LD_LIBRARY_PATH"环境变量，
		//LD_LIBRARY_PATH环境变量用于在程序加载运行期间查找动态链接库时指定除了系统默认路径之外的其他路径
		//实现中的一句注释  no environment variable is a good environment variable
        mustsetenv = RequiresSetenv(wanted, jvmpath);
        JLI_TraceLauncher("mustsetenv: %s\n", mustsetenv ? "TRUE" : "FALSE");

        if (mustsetenv == 0) {
            JLI_MemFree(newargv);
            return;
        }
      } else {  /* do the same speculatively or exit */
        JLI_ReportErrorMessage("Error: This Java instance does not support a %d-bit JVM.\nPlease install the desired version.", wanted);
        exit(1);
      }
	  //设置了LD_LIBRARY_PATH环境变量，并且目录中存在lib/$LIBARCH/{server,client}/libjvm.so文件
	  if (mustsetenv) {
            ......
        }
        
        exit(1);
    }
}
```  
函数`GetJREPath`使用来获取`JRE`路径的。在`CreateExecutionEnvironment`函数中，通过调用`SetExecname`函数将可执行文件的绝对路径塞进`execname`变量，比如这里是`jdkpath/jdk/bin/java`。经过简单处理后得到`jdkpath/jdk`。该函数还检查了链接库的访问权限。
```c++
/*
 * Find path to JRE based on .exe's location or registry settings.
 */
static jboolean GetJREPath(char *path, jint pathsize, const char * arch, jboolean speculative)
{
    char libjava[4096];
	//找到JDK路径并传到path变量中
    if (GetApplicationHome(path, pathsize)) {
        /* Is JRE co-located with the application? */
		// /home/ccr/jvm/openjdk/build/linux-x86_64-normal-server-slowdebug/jdk/lib/amd64/libjava.so
        snprintf(libjava, sizeof(libjava), "%s/lib/%s/" "libjava.so", path, arch);
		//是否可访问
        if (access(libjava, 0) == 0) {
            JLI_TraceLauncher("JRE path is %s\n", path);
            return 1;
        }

        /* Does the app ship a private JRE in <apphome>/jre directory? */
        snprintf(libjava, sizeof(libjava), "%s/jre/lib/%s/" "libjava.so", path, arch);
        if (access(libjava, 0) == 0) {
            strcat(( path ), ( "/jre" ));
            JLI_TraceLauncher("JRE path is %s\n", path);
            return 1;
        }
    }

    if (!speculative)
      JLI_ReportErrorMessage("Error: could not find " "libjava.so");
    return 0;
}
/*
 * If app is "/foo/bin/javac", or "/foo/bin/sparcv9/javac" then put
 * "/foo" into buf.
 */
jboolean GetApplicationHome(char *buf, jint bufsize)
{
	//GetExecName()直接返回execname变量，在SetExecname(*pargv)函数中初始化
    const char *execname = GetExecName();
    if (execname != ((void *)0)) {
		//将格式化的数据写入字符串,超出则截断
        snprintf(buf, bufsize, "%s", execname);
        buf[bufsize-1] = '\0';
    } else {
        return 0;
    }

	//strrchr(const char *str, int c) 在参数 str 所指向的字符串中搜索最后一次出现字符 c（一个无符号字符）的位置。
    if (strrchr(( buf ), ( '/' )) == 0) {
        buf[0] = '\0';
        return 0;
    }
	
	//直接截除最后一次出现'/'以及后面的字符,'/foo/bin/java'->'/foo/bin'
    *(strrchr(( buf ), ( '/' ))) = '\0';    /* executable file      */
	//strlen 计算字符串长度
	//字符串长度<4，其实就是找不到jdk路径
    if (strlen(( buf )) < 4 || strrchr(( buf ), ( '/' )) == 0) {
        buf[0] = '\0';
        return 0;
    }
	//字符串比较，buf + strlen(( buf ) - 4 实际上是地址的操作或者可以理解为数组下标相加
    if (strcmp(( "/bin" ), ( buf + strlen(( buf )) - 4 )) != 0)
		//直接截除最后一次出现'/'以及后面的字符,'/foo/bin'->'/foo'
		//这一段不理解，为什么会有这个判断，从注释看，有可能java命令不在bin目录中，而在bin/sparcv9 或者 bin/amd64目录中
        *(strrchr(( buf ), ( '/' ))) = '\0';        /* sparcv9 or amd64     */
    if (strlen(( buf )) < 4 || strcmp(( "/bin" ), ( buf + strlen(( buf )) - 4 )) != 0) {
		//不可能在找到bin目录了
        buf[0] = '\0';
        return 0;
    }
	//直接截除最后一次出现'/'以及后面的字符,'/foo/bin'->'/foo'
    *(strrchr(( buf ), ( '/' ))) = '\0';    /* bin                  */

    return 1;
}
```  
`jvm.cfg`文件的定义。默认选择的是`server`类型。以及`knownVMs`变量数据结构
```c
-server KNOWN
-client IGNORE

//knownVMs变量数据结构
struct vmdesc {
    char *name;
    int flag;
    char *alias;
    char *server_class;
};
enum vmdesc_flag {
    VM_UNKNOWN = -1,
    VM_KNOWN,
    VM_ALIASED_TO,
    VM_WARN,
    VM_ERROR,
    VM_IF_SERVER_CLASS,
    VM_IGNORE
};
``` 
## 动态库加载
使用`dlopen`函数打开动态库，并将其中的`JNI_CreateJavaVM`、`JNI_GetDefaultJavaVMInitArgs`、`JNI_GetCreatedJavaVMs`函数链接至`ifn`变量中。
```c++
jboolean LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    void *libjvm;

    JLI_TraceLauncher("JVM path is %s\n", jvmpath);

	//打开动态库，立刻模式和全局模式
    libjvm = dlopen(jvmpath,  RTLD_NOW + RTLD_GLOBAL);
    if (libjvm == ((void *)0)) {
        JLI_ReportErrorMessage("Error: dl failure on line %d", 864);
        JLI_ReportErrorMessage("Error: failed %s, because %s", jvmpath, dlerror());
        return 0;
    }

	//函数链接
    ifn->CreateJavaVM = (CreateJavaVM_t)
        dlsym(libjvm, "JNI_CreateJavaVM");
    if (ifn->CreateJavaVM == ((void *)0)) {
        JLI_ReportErrorMessage("Error: failed %s, because %s", jvmpath, dlerror());
        return 0;
    }

    ifn->GetDefaultJavaVMInitArgs = (GetDefaultJavaVMInitArgs_t)
        dlsym(libjvm, "JNI_GetDefaultJavaVMInitArgs");
    if (ifn->GetDefaultJavaVMInitArgs == ((void *)0)) {
        JLI_ReportErrorMessage("Error: failed %s, because %s", jvmpath, dlerror());
        return 0;
    }

    ifn->GetCreatedJavaVMs = (GetCreatedJavaVMs_t)
        dlsym(libjvm, "JNI_GetCreatedJavaVMs");
    if (ifn->GetCreatedJavaVMs == ((void *)0)) {
        JLI_ReportErrorMessage("Error: failed %s, because %s", jvmpath, dlerror());
        return 0;
    }

    return 1;
}
``` 
## options解析
通过`AddOption`函数将所有`options`记录在`JavaVMOption`变量中。
```c++ 
void AddOption(char *str, void *info)
{
    /*
     * Expand options array if needed to accommodate at least one more
     * VM option.
     */
    if (numOptions >= maxOptions) {
        if (options == 0) {
            maxOptions = 4;
            options = JLI_MemAlloc(maxOptions * sizeof(JavaVMOption));
        } else {
            JavaVMOption *tmp;
            maxOptions *= 2;
            tmp = JLI_MemAlloc(maxOptions * sizeof(JavaVMOption));
            memcpy(tmp, options, numOptions * sizeof(JavaVMOption));
            JLI_MemFree(options);
            options = tmp;
        }
    }
	//将参数，插入到options变量中
    options[numOptions].optionString = str;
    options[numOptions++].extraInfo = info;

	//判断是否是-Xss参数，设置栈大小
    if (JLI_StrCCmp(str, "-Xss") == 0) {
        jlong tmp;
        if (parse_size(str + 4, &tmp)) {
            threadStackSize = tmp;
        }
    }
	
	//判断是否是-Xmx参数，设置最大堆内存
    if (JLI_StrCCmp(str, "-Xmx") == 0) {
        jlong tmp;
        if (parse_size(str + 4, &tmp)) {
            maxHeapSize = tmp;
        }
    }

	//判断是否是-Xmx参数，设置初始堆内存
    if (JLI_StrCCmp(str, "-Xms") == 0) {
        jlong tmp;
        if (parse_size(str + 4, &tmp)) {
           initialHeapSize = tmp;
        }
    }
}
//options变量结构
typedef struct JavaVMOption {
    char *optionString;
    void *extraInfo;
} JavaVMOption;


//解析options
static jboolean ParseArguments(int *pargc, char ***pargv,
               int *pmode, char **pwhat,
               int *pret, const char *jrepath)
{
    int argc = *pargc;
    char **argv = *pargv;
    int mode = LM_UNKNOWN;
    char *arg;

    *pret = 0;

    while ((arg = *argv) != 0 && *arg == '-') {
        argv++; --argc;
        if (strcmp(( arg ), ( "-classpath" )) == 0 || strcmp(( arg ), ( "-cp" )) == 0) {
            do {
                if ( argc < 1) {
                    JLI_ReportErrorMessage( "Error: %s requires class path specification" , arg );
                    printUsage = 1 ;
                    *pret = 1;
                    return 1 ;
                }
            }
            while ( 0 );
            SetClassPath(*argv);
            mode = LM_CLASS;
            argv++; --argc;
        } else if (strcmp(( arg ), ( "-jar" )) == 0) {
            do {
                if ( argc < 1) {
                    JLI_ReportErrorMessage( "Error: %s requires jar file specification" , arg );
                    printUsage = 1 ;
                    *pret = 1;
                    return 1 ;
                }
            }
            while ( 0 );
            mode = LM_JAR;
        } else if (strcmp(( arg ), ( "-help" )) == 0 ||
                   strcmp(( arg ), ( "-h" )) == 0 ||
                   strcmp(( arg ), ( "-?" )) == 0) {
            printUsage = 1;
            return 1;
        } else if (strcmp(( arg ), ( "-version" )) == 0) {
            printVersion = 1;
            return 1;
        } else if (strcmp(( arg ), ( "-showversion" )) == 0) {
            showVersion = 1;
        } else if (strcmp(( arg ), ( "-X" )) == 0) {
            printXUsage = 1;
            return 1;
/*
 * The following case checks for -XshowSettings OR -XshowSetting:SUBOPT.
 * In the latter case, any SUBOPT value not recognized will default to "all"
 */
        } else if (strcmp(( arg ), ( "-XshowSettings" )) == 0 ||
                JLI_StrCCmp(arg, "-XshowSettings:") == 0) {
            showSettings = arg;
        } else if (strcmp(( arg ), ( "-Xdiag" )) == 0) {
            AddOption("-Dsun.java.launcher.diag=true", ((void *)0));
/*
 * The following case provide backward compatibility with old-style
 * command line options.
 */
        } else if (strcmp(( arg ), ( "-fullversion" )) == 0) {
            JLI_ReportMessage("%s full version \"%s\"", _launcher_name, GetFullVersion());
            return 0;
        } else if (strcmp(( arg ), ( "-verbosegc" )) == 0) {
            AddOption("-verbose:gc", ((void *)0));
        } else if (strcmp(( arg ), ( "-t" )) == 0) {
            AddOption("-Xt", ((void *)0));
        } else if (strcmp(( arg ), ( "-tm" )) == 0) {
            AddOption("-Xtm", ((void *)0));
        } else if (strcmp(( arg ), ( "-debug" )) == 0) {
            AddOption("-Xdebug", ((void *)0));
        } else if (strcmp(( arg ), ( "-noclassgc" )) == 0) {
            AddOption("-Xnoclassgc", ((void *)0));
        } else if (strcmp(( arg ), ( "-Xfuture" )) == 0) {
            AddOption("-Xverify:all", ((void *)0));
        } else if (strcmp(( arg ), ( "-verify" )) == 0) {
            AddOption("-Xverify:all", ((void *)0));
        } else if (strcmp(( arg ), ( "-verifyremote" )) == 0) {
            AddOption("-Xverify:remote", ((void *)0));
        } else if (strcmp(( arg ), ( "-noverify" )) == 0) {
            AddOption("-Xverify:none", ((void *)0));
        } else if (JLI_StrCCmp(arg, "-prof") == 0) {
            char *p = arg + 5;
            char *tmp = JLI_MemAlloc(strlen(( arg )) + 50);
            if (*p) {
                sprintf(tmp, "-Xrunhprof:cpu=old,file=%s", p + 1);
            } else {
                sprintf(tmp, "-Xrunhprof:cpu=old,file=java.prof");
            }
            AddOption(tmp, ((void *)0));
        } else if (JLI_StrCCmp(arg, "-ss") == 0 ||
                   JLI_StrCCmp(arg, "-oss") == 0 ||
                   JLI_StrCCmp(arg, "-ms") == 0 ||
                   JLI_StrCCmp(arg, "-mx") == 0) {
            char *tmp = JLI_MemAlloc(strlen(( arg )) + 6);
            sprintf(tmp, "-X%s", arg + 1); /* skip '-' */
            AddOption(tmp, ((void *)0));
        } else if (strcmp(( arg ), ( "-checksource" )) == 0 ||
                   strcmp(( arg ), ( "-cs" )) == 0 ||
                   strcmp(( arg ), ( "-noasyncgc" )) == 0) {
            /* No longer supported */
            JLI_ReportErrorMessage("Warning: %s option is no longer supported.", arg);
        } else if (JLI_StrCCmp(arg, "-version:") == 0 ||
                   strcmp(( arg ), ( "-no-jre-restrict-search" )) == 0 ||
                   strcmp(( arg ), ( "-jre-restrict-search" )) == 0 ||
                   JLI_StrCCmp(arg, "-splash:") == 0) {
            ; /* Ignore machine independent options already handled */
        } else if (ProcessPlatformOption(arg)) {
            ; /* Processing of platform dependent options */
        } else if (RemovableOption(arg)) {
            ; /* Do not pass option to vm. */
        } else {
            AddOption(arg, ((void *)0));
        }
    }

    if (--argc >= 0) {
        *pwhat = *argv++;
    }

    if (*pwhat == ((void *)0)) {
        *pret = 1;
    } else if (mode == LM_UNKNOWN) {
        /* default to LM_CLASS if -jar and -cp option are
         * not specified */
        mode = LM_CLASS;
    }

    if (argc >= 0) {
        *pargc = argc;
        *pargv = argv;
    }

    *pmode = mode;

    return 1;
}

```  
## 开启线程初始化虚拟机并执行main方法
开启线程传入线程执行的函数`JavaMain`，在`JavaMain`中初始化虚拟机并找到主类和执行`main`方法。
```c++
//开启新线程初始化JVM
ContinueInNewThread(InvocationFunctions* ifn, jlong threadStackSize,
                    int argc, char **argv,
                    int mode, char *what, int ret)
{

    /*
     * If user doesn't specify stack size, check if VM has a preference.
     * Note that HotSpot no longer supports JNI_VERSION_1_1 but it will
     * return its default stack size through the init args structure.
     */
	 //如果用户没有用"-Xss"参数指定栈深，则使用默认的，通过GetDefaultJavaVMInitArgs函数获取
    if (threadStackSize == 0) {
      struct JDK1_1InitArgs args1_1;
      memset((void*)&args1_1, 0, sizeof(args1_1));
      args1_1.version = 0x00010001;
      ifn->GetDefaultJavaVMInitArgs(&args1_1);  /* ignore return value */
      if (args1_1.javaStackSize > 0) {
         threadStackSize = args1_1.javaStackSize;
      }
    }

    { /* Create a new thread to create JVM and invoke main method */
      JavaMainArgs args;
      int rslt;

      args.argc = argc;
      args.argv = argv;
      args.mode = mode;
      args.what = what;
      args.ifn = *ifn;

	  //创建线程，第一个参数是线程执行函数，第二个是线程属性，第三个是函数参数
      rslt = ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
      /* If the caller has deemed there is an error we
       * simply return that, otherwise we return the value of
       * the callee
       */
      return (ret != 0) ? ret : rslt;
    }
}

int ContinueInNewThread0(int ( *continuation)(void *), jlong stack_size, void * args) {
    int rslt;
    pthread_t tid;
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);

    if (stack_size > 0) {
      pthread_attr_setstacksize(&attr, stack_size);
    }
	//第一个参数为指向线程标识符的指针。第二个参数用来设置线程属性。
	//第三个参数是线程运行函数的起始地址。最后一个参数是运行函数的参数。
    if (pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args) == 0) {
      void * tmp;
	  //代码中如果没有pthread_join主线程会很快结束从而使整个进程结束，从而使创建的线程没有机会开始执行就结束了。
	  //加入pthread_join后，主线程会一直等待直到等待的线程结束自己才结束，使创建的线程有机会执行。
      pthread_join(tid, &tmp);
      rslt = (int)tmp;
    } else {
     /*
      * Continue execution in current thread if for some reason (e.g. out of
      * memory/LWP)  a new thread can't be created. This will likely fail
      * later in continuation as JNI_CreateJavaVM needs to create quite a
      * few new threads, anyway, just give it a try..
      */
	  //直接执行函数
      rslt = continuation(args);
    }

    pthread_attr_destroy(&attr);
    return rslt;
}


int JavaMain(void * _args)
{
    JavaMainArgs *args = (JavaMainArgs *)_args;
    int argc = args->argc;
    char **argv = args->argv;
    int mode = args->mode;
    char *what = args->what;
    InvocationFunctions ifn = args->ifn;

	//这两个参数在创建虚拟机时被初始化
    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jclass mainClass = ((void *)0);
    jclass appClass = ((void *)0); // actual application class being launched
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

	//Linux 并未实现
    RegisterThread();

    /* Initialize the virtual machine */
    start = (0);
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage("Error: Could not create the Java Virtual Machine.\n" 
		"Error: A fatal exception has occurred. Program will exit.");
        exit(1);
    }

    if (showSettings != ((void *)0)) {
        ShowSettings(env, showSettings);
        ......
    }

    if (printVersion || showVersion) {
        PrintJavaVersion(env, showVersion);
        ......
    }

    /* If the user specified neither a class name nor a JAR file */
    if (printXUsage || printUsage || what == 0 || mode == LM_UNKNOWN) {
        PrintUsage(env, printXUsage);
        ......
    }

	//释放Knownvms变量
    FreeKnownVMs();  /* after last possible PrintUsage() */

    //打印调试的信息
	......
	
    ret = 1;

    /*
     * Get the application's main class.
     *
     * See bugid 5030265.  The Main-Class name has already been parsed
     * from the manifest, but not parsed properly for UTF-8 support.
     * Hence the code here ignores the value previously extracted and
     * uses the pre-existing code to reextract the value.  This is
     * possibly an end of release cycle expedient.  However, it has
     * also been discovered that passing some character sets through
     * the environment has "strange" behavior on some variants of
     * Windows.  Hence, maybe the manifest parsing code local to the
     * launcher should never be enhanced.
     *
     * Hence, future work should either:
     *     1)   Correct the local parsing code and verify that the
     *          Main-Class attribute gets properly passed through
     *          all environments,
     *     2)   Remove the vestages of maintaining main_class through
     *          the environment (and remove these comments).
     *
     * This method also correctly handles launching existing JavaFX
     * applications that may or may not have a Main-Class manifest entry.
     */
	//获取主类
    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    /*
     * In some cases when launching an application that needs a helper, e.g., a
     * JavaFX application with no main method, the mainClass will not be the
     * applications own main class but rather a helper class. To keep things
     * consistent in the UI we need to track and report the application main class.
     */
	//在checkAndLoadMain方法中已经赋值为mainClass
    appClass = GetApplicationClass(env);
    NULL_CHECK_RETURN_VALUE(appClass, -1);
    /*
     * PostJVMInit uses the class name as the application name for GUI purposes,
     * for example, on OSX this sets the application name in the menu bar for
     * both SWT and JavaFX. So we'll pass the actual application class here
     * instead of mainClass as that may be a launcher or helper class instead
     * of the application class.
     */
	//Linux未实现
    PostJVMInit(env, appClass, vm);
    /*
     * The LoadMainClass not only loads the main class, it will also ensure
     * that the main method's signature is correct, therefore further checking
     * is not required. The main method is invoked here so that extraneous java
     * stacks are not in the application stack trace.
     */
	 //获取main方法的引用
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    CHECK_EXCEPTION_NULL_LEAVE(mainID);

    /* Build platform specific argument array */
	//创建java类型的参数
    mainArgs = CreateApplicationArgs(env, argv, argc);
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
	//调用main方法
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    ret = (*env)->ExceptionOccurred(env) == ((void *)0) ? 0 : 1;
    LEAVE();
}
``` 
### 初始化虚拟机
虚拟机初始化的具体实现并不在`java.c`文件中，而是在动态链接库中，其实现比较复杂，大体上应该是创建虚拟机环境，包括内存模型、线程模型、以及垃圾回收等。返回虚拟机环境的一些接口，这些接口定义在`jni.h`文件中。这里不对实现做深入研究，简单看下几个接口的作用。
请参考:[https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/functions.html]
* `jmethodID GetStaticMethodID(JNIEnv *env, jclass clazz,const char *name, const char *sig);`获取类中静态方法的引用。参数分别是：虚拟机环境、类、方法名、方法声明。
* `CallStatic<type>Method(JNIEnv *env, jclass clazz,jmethodID methodID, ...);`调用静态方法。`...`是方法参数。

```c++

//初始化JVM  ifn是JAVA_DLL库中的几个函数
static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    JavaVMInitArgs args;
    jint r;

	//作用是在一段内存块中填充某个给定的值，它对较大的结构体或数组进行清零操作的一种最快方法。
    memset(&args, 0, sizeof(args));
	//下面这些参数在环境准备时已经准备好了
    args.version  = 0x00010002;
    args.nOptions = numOptions;
    args.options  = options;
    args.ignoreUnrecognized = 0;

    if (JLI_IsTraceLauncher()) {
        int i = 0;
        printf("JavaVM args:\n    ");
        printf("version 0x%08lx, ", (long)args.version);
        printf("ignoreUnrecognized is %s, ",
               args.ignoreUnrecognized ? "JNI_TRUE" : "JNI_FALSE");
        printf("nOptions is %ld\n", (long)args.nOptions);
        for (i = 0; i < numOptions; i++)
            printf("    option[%2d] = '%s'\n",
                   i, args.options[i].optionString);
    }
	//创建虚拟机实例并返回，之前只是加载动态库，到这里才正式创建虚拟机
    r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
    JLI_MemFree(options);
    return r == 0;
}
``` 
### 加载主类
`JVM`加载`HelloJNI`类时并不是直接加载，而是先加载`sun.launcher.LauncherHelper`,调用该类中的`checkAndLoadMain`方法去加载`HelloJNI`类。然后在调用`HelloJNI`的`main`方法。这样做的目的是把类加载的方式交给用户自己定义。

```c++
//加载主类
static jclass
LoadMainClass(JNIEnv *env, int mode, char *name)
{
    jmethodID mid;
    jstring str;
    jobject result;
    jlong start, end;
	//找到 sun/launcher/LauncherHelper 类
    jclass cls = GetLauncherHelperClass(env);
	//检查是否为null
    NULL_CHECK0(cls);
    if (JLI_IsTraceLauncher()) {
        start = (0);
    }
	//用虚拟机提供的函数获取"sun/launcher/LauncherHelper"类中"checkAndLoadMain"静态方法
    NULL_CHECK0(mid = (*env)->GetStaticMethodID(env, cls,
                "checkAndLoadMain",
                "(ZILjava/lang/String;)Ljava/lang/Class;"));

	//获取java语言中的String，java中String是个对象，而不是字符数组
    str = NewPlatformString(env, name);
	//调用checkAndLoadMain方法，加载主类
    result = (*env)->CallStaticObjectMethod(env, cls, mid, 1, mode, str);

    if (JLI_IsTraceLauncher()) {
        end   = (0);
        printf("%ld micro seconds to load main class\n",
               (long)(jint)(1));
        printf("----%s----\n", "_JAVA_LAUNCHER_DEBUG");
    }

    return (jclass)result;
}

jclass GetLauncherHelperClass(JNIEnv *env)
{
    if (helperClass == ((void *)0)) {
        do {
            if (( helperClass = FindBootStrapClass ( env , "sun/launcher/LauncherHelper" ) ) == ((void *)0) ) {
                JLI_ReportErrorMessage( "Error: A JNI error has occurred, please check your installation and try again" );
                return 0 ;
            }
        }
        while ( 0 );
    }
    return helperClass;
}


jclass FindBootStrapClass(JNIEnv *env, const char* classname)
{
   if (findBootClass == NULL) {
	   //根据 动态链接库 操作句柄(handle)与符号(symbol)，返回符号对应的地址。使用这个函数不但可以获取函数地址，也可以获取变量地址。
	   //在handle为RTLD_DEFAULT的情况中，linker将遍历系统中已经加载的所有动态链接库，并在每一个动态连接库里查找那个符号。
	   //获取函数JVM_FindClassFromBootLoader的指针
       findBootClass = (FindClassFromBootLoader_t *)dlsym(RTLD_DEFAULT,
          "JVM_FindClassFromBootLoader");
       if (findBootClass == NULL) {
           JLI_ReportErrorMessage(DLL_ERROR4,
               "JVM_FindClassFromBootLoader");
           return NULL;
       }
   }
   return findBootClass(env, classname);
}

/*
 * Returns a new Java string object for the specified platform string.
 */
static jstring NewPlatformString(JNIEnv *env, char *s)
{
    int len = (int)JLI_StrLen(s);
    jbyteArray ary;
    jclass cls = GetLauncherHelperClass(env);
    NULL_CHECK0(cls);
    if (s == NULL)
        return 0;

    ary = (*env)->NewByteArray(env, len);
    if (ary != 0) {
        jstring str = 0;
        (*env)->SetByteArrayRegion(env, ary, 0, len, (jbyte *)s);
        if (!(*env)->ExceptionOccurred(env)) {
            if (makePlatformStringMID == NULL) {
                NULL_CHECK0(makePlatformStringMID = (*env)->GetStaticMethodID(env,
                        cls, "makePlatformString", "(Z[B)Ljava/lang/String;"));
            }
            str = (*env)->CallStaticObjectMethod(env, cls,
                    makePlatformStringMID, USE_STDERR, ary);
            (*env)->DeleteLocalRef(env, ary);
            return str;
        }
    }
    return 0;
}
``` 
而checkAndLoadMain又做了哪些事情呢。
* 如果是`jar`模式，则从`manifest`文件中获取类名。
* 使用`System ClassLoader`加载类。
* 确保`main`方法的有效性和可访问。
* `FX Application`相关。

```java
/**
 * This method does the following:
 * 1. gets the classname from a Jar's manifest, if necessary
 * 2. loads the class using the System ClassLoader
 * 3. ensures the availability and accessibility of the main method,
 *    using signatureDiagnostic method.
 *    a. does the class exist
 *    b. is there a main
 *    c. is the main public
 *    d. is the main static
 *    e. does the main take a String array for args
 * 4. if no main method and if the class extends FX Application, then call
 *    on FXHelper to determine the main class to launch
 * 5. and off we go......
 *
 * @param printToStderr if set, all output will be routed to stderr
 * @param mode LaunchMode as determined by the arguments passed on the
 * command line
 * @param what either the jar file to launch or the main class when using
 * LM_CLASS mode
 * @return the application's main class
 */
public static Class<?> checkAndLoadMain(boolean printToStderr,
                                        int mode,
                                        String what) {
    initOutput(printToStderr);
    // get the class name
    String cn = null;
    switch (mode) {
        case LM_CLASS:
            cn = what;
            break;
        case LM_JAR:
            cn = getMainClassFromJar(what);
            break;
        default:
            // should never happen
            throw new InternalError("" + mode + ": Unknown launch mode");
    }
    cn = cn.replace('/', '.');
    Class<?> mainClass = null;
    try {
        mainClass = scloader.loadClass(cn);
    } catch (NoClassDefFoundError | ClassNotFoundException cnfe) {
        if (System.getProperty("os.name", "").contains("OS X")
            && Normalizer.isNormalized(cn, Normalizer.Form.NFD)) {
            try {
                // On Mac OS X since all names with diacretic symbols are given as decomposed it
                // is possible that main class name comes incorrectly from the command line
                // and we have to re-compose it
                mainClass = scloader.loadClass(Normalizer.normalize(cn, Normalizer.Form.NFC));
            } catch (NoClassDefFoundError | ClassNotFoundException cnfe1) {
                abort(cnfe, "java.launcher.cls.error1", cn);
            }
        } else {
            abort(cnfe, "java.launcher.cls.error1", cn);
        }
    }
    // set to mainClass
    appClass = mainClass;

    /*
     * Check if FXHelper can launch it using the FX launcher. In an FX app,
     * the main class may or may not have a main method, so do this before
     * validating the main class.
     */
    if (mainClass.equals(FXHelper.class) ||
            FXHelper.doesExtendFXApplication(mainClass)) {
        // Will abort() if there are problems with the FX runtime
        FXHelper.setFXLaunchParameters(what, mode);
        return FXHelper.class;
    }

    validateMainClass(mainClass);
    return mainClass;
}
``` 

> 参考：[简书占小狼](http://www.jianshu.com/p/b91258bc08ac)
