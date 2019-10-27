

#                      JVM 体系结构

### jvm 体系结构

![1566272336587](images/1566272336587.png)

![1566272349642](images/1566272349642.png)

- 类加载器
  - 加载class 类
- 执行引擎
  - 解析JVM 字节码指令，得到执行结果
  - ![1566272514466](images/1566272514466.png)
- java内存管理

![1566272575617](images/1566272575617.png)

间结果等，而pc寄存器则会指向即将执行的下一条指令。

### 机器如何执行代码

![1566272687081](images/1566272687081.png)

![1566272695145](images/1566272695145.png)

### 执行引擎的架构设计

![1566272747256](images/1566272747256.png)

- 执行引擎的执行过程

```java
public class Math {
    public static void main(String[] args) {
         int a =1;
         int b =2;
         int c = (a+b)*10;
    }
}
```

![1566272892722](images/1566272892722.png)

![1566272909889](images/1566272909889.png)

![1566272938167](images/1566272938167.png)

- java方法调用栈

```java
public class Math {
    public static void main(String[] args) {
         int a =1;
         int b =2;
         int c = (a+b)*10;
    }
    public  static  int math(int a,int b){
        return (a+b)/10;
    }
}
```

![1566273126322](images/1566273126322.png)

![1566273133457](images/1566273133457.png)

![1566273388996](images/1566273388996.png)

![](images/1566273168711.png)

![1566273214189](images/1566273214189.png)