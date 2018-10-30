# class文件里的LocalVariableTable



在学习Springmvc的时候，到参数绑定的时候，遇到了很多大大小小的坑。基本数据类型和String类型，可以直接传递，只要方法参数的变量名和传递过来的一致就可以，不一致的时候就在参数类型前面加上@RequestParam指定。这时就有一个疑问了，编译后的class文件是没有保留方法参数的变量名的，Springmvc是怎么知道的呢？

```java
@RestController
public class TestController {
    @Autowired
    private StudentService studentService;
    public Student findStudentById(String id){
        return studentService.findStudentById(id);
    }
}
```

通常我们通过反编译看到的class文件，方法里的参数通常都是var1、var2..诸如此类，在JDK8之前也不能通过反射获取方法参数的变量名。

### LocalVariableTable 本地变量表

按jvm规范所述,本地变量表存在于Code属性中，而Code属性又是methodInfo的一个属性。可以理解为，当一个方法有方法体时，就会出现相应的Code属性，而且在code属性中，除具体的执行代码外，还会有其它的信息。如LineNumberTable(用于描述每一行代码所在的位置)。

![](images\javap后的LocalVariableTable.png)

图中的和LocalVariableTable同级的MethodParameters是jdk8新增的特性，javac和IDE默认都没有开启，需要javac  -parameters生效，有了这个调试信息就可以在反射中获取本地变量名了。



本地变量表在方法中属于调试信息的一部分，因此默认javac是不会生成的，需要javac -g或者javac -g:vars（-g代表生成调试信息，包括vars本地变量调试信息、lines 行号调试信息、source 源文件调试信息)。不过在我们常用的IDE中是默认开启的。

idea设置

![](images\idea设置开启调试.png)

Generate debugging info打勾即可生成调试信息，默认是勾选的。

eclipse设置

![](images\eclise中开启调试.png)



因为class文件生成调试信息在编译时是可选的，为了稳定，建议在Controller中的基础数据类型参数，都用上注解绑定参数。



### 附javac、javap命令选项

**javap命令参数**

```
-help  --help  -?        输出此用法消息
-version                 版本信息
-v  -verbose             输出附加信息
-l                       输出行号和本地变量表
-public                  仅显示公共类和成员
-protected               显示受保护的/公共类和成员
-package                 显示程序包/受保护的/公共类和成员 (默认)
-p  -private             显示所有类和成员
-c                       对代码进行反汇编
-s                       输出内部类型签名
-sysinfo                 显示正在处理的类的系统信息 (路径, 大小, 日期, MD5 散列)
-constants               显示静态最终常量
-classpath <path>        指定查找用户类文件的位置
-bootclasspath <path>    覆盖引导类文件的位置
```

**javac命令参数**

```
-g                         生成所有调试信息
-g:none                    不生成任何调试信息
-g:{lines,vars,source}     只生成某些调试信息
-nowarn                    不生成任何警告
-verbose                   输出有关编译器正在执行的操作的消息
-deprecation               输出使用已过时的 API 的源位置
-classpath <路径>            指定查找用户类文件和注释处理程序的位置
-cp <路径>                   指定查找用户类文件和注释处理程序的位置
-sourcepath <路径>           指定查找输入源文件的位置
-bootclasspath <路径>        覆盖引导类文件的位置
-extdirs <目录>              覆盖所安装扩展的位置
-endorseddirs <目录>         覆盖签名的标准路径的位置
-proc:{none,only}          控制是否执行注释处理和/或编译。
-processor <class1>[,<class2>,<class3>...] 要运行的注释处理程序的名称; 绕过默认的搜索进程
-processorpath <路径>        指定查找注释处理程序的位置
-parameters                生成元数据以用于方法参数的反射
-d <目录>                    指定放置生成的类文件的位置
-s <目录>                    指定放置生成的源文件的位置
-h <目录>                    指定放置生成的本机标头文件的位置
-implicit:{none,class}     指定是否为隐式引用文件生成类文件
-encoding <编码>             指定源文件使用的字符编码
-source <发行版>              提供与指定发行版的源兼容性
-target <发行版>              生成特定 VM 版本的类文件
-profile <配置文件>            请确保使用的 API 在指定的配置文件中可用
-version                   版本信息
-help                      输出标准选项的提要
-A关键字[=值]                  传递给注释处理程序的选项
-X                         输出非标准选项的提要
-J<标记>                     直接将 <标记> 传递给运行时系统
-Werror                    出现警告时终止编译
@<文件名>                     从文件读取选项和文件名
```



  