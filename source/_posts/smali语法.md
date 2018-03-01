---
title: smali语法
date: 2018-02-26 10:13:50
tags: smali
subtitle: smali
categories: smali
---

> ### smali中的包信息

```java
.class public Lcom/a;
.super Lcom/b;
.source "c.java"
```

这是一个由c.java编译得到的smail文件。他是com.a这个包下的一个类，继承自com.b这个类。

> ### smali中的声明

```java
# annotations
.annotations system Ldalvik/annotation/MemberClasses;
value={
  Lcom/a$z;
  Lcom/a$x;
}
.end annotation
```

这个模块是内部类的声明，a这个类中有两个成员内部类：z和x。

> ### smali中的寄存器

在smail中所有的操作都必须经过寄存器来进行。本地寄存器用v来表示，例如v0，v1，v2等。参数寄存器用p来表示，例如p0，p1，p2等。特别的一点是，p0并不一定是函数中的第一个参数，在非静态方法中，它指代“this”，p1表示函数的第一个参数。而在静态方法中，p0就是函数的第一个参数。

> ### 寄存器的表示

```java
const/4 v0,0x1
iput-boolean v0,p0,Lcom/a;->isTrue:Z
```

上面的代码块首先使用了v0本地寄存器，并把值0x1存到v0中，第二句用`iput-boolean`这个指令把v0中的值存放到com.a.isTrue这个成员变量中，即：`this.isTrue=true；`。

> ### smali中的成员变量

```java
.field public/private [static] [final] varName:<类型>
```

对于不同的成员变量有不同的指令。一般的获取指令有:`iget,sget,iget-boolean,sget-boolean,iget-object,sget-object` 一般的操作指令有：`iput,sput,iput-boolean,sput-boolean,iput-object,sput-object`

其中没有-object后缀的表示操作的是基本数据类型，反之操作的是对象。操作boolean类型需要添加后缀-boolean。

> ### smali成员变量指令

```java
sget-object v0,Lcom/a;->ID:Ljava/lang/String;
```

获取ID这个String类型的成员变量放到寄存器V0中。

```java
iget-object v0,p0,Lcom/a;->view:Lcom/a/view;
```

iget比sget多了一个参数寄存器p0这个就是指代该成员变量所在类的实例，即p0指代“this”。

对于获取array数组的操作的指令是aget，aget-object。

```java
const/4 v3,0x0
sput-object v3,Lcom/a;->timer:Lcom/a/timer;
```

以上两行指令相当于`timer=null;` 如果后缀为-boolean那么0x0代表false；

```java
.local v0,args:Landroid/os/Message;
const/4 v1,0x12
iput v1,v0,Landroid/os/Message;->what:I;
```

上面3行指令相当于：`args.what=18;`

> ### smali函数的调用

smali中的函数和成员变量一样也分为两种类型，direct method和virtual method。

direct method代表的函数是private的，virtual method则代表的是public和protected，所以在调用函数时有invoke-direct和invoke-virtual，invoke-static，invoke-super及invoke-interface等几种不同的指令。对于参数多于4个的函数，调用的形式为invoke-XXXXX/range

invoke-static表示调用的是静态函数。例如`invoke-static {},Landroid/os/Debug;->waitForDebugger()V;`。

invoke-static后面的{}表示调用此函数的实例和参数列表，由于这个方法是static的也不需要参数，所以{}为空。

```java
const-string v0,"NDKLIB"
invoke-static {v0},Ljava/lang/System;->lodaLibrary(Ljava/lang/Sting;)V
```

这两行指令表示调用static void System.loadlibrary(String)方法。

invoke-super调用父类方法的指令，生命周期方法中用的较多。

`invoke-direct {p0},Landroid/app/TabActivity;-><init>()V` 这个init()方法就是定义在TabActivity中的private方法。

```java
sget-object v0,Lcom/a;->b:Lcom/c;
invoke-virtual {v0,v1},Lcom/c;->Messages(Ljava/lang/Object;)V
```

v0是b:Lcom/c ,v1是传递给void Messages(Object)方法的参数。

如果函数的参数多于4个（不包括4个）时调用方法的指令后跟/range ,例如：

```java
invoke-direct/range {v0...v5},Lcmb/pb/ui/PBContainerActivity;->h(ILjava/lang/CharSequence;Ljava/lang/String;Landroid/content/Intent;I)Z
```

需要传递v0到v5共6个参数，{}中采用省略号形式，且连续。

> ### smali中函数结果的返回操作

java中调用函数和返回函数结果可以用一条语句完成，但在smail中则需要分开完成，如果调用的函数返回非void，那么还需要用到move-result（返回基本数据类型）或者move-result-object（返回对象类型）。

```java
const-string v0,"aaa"
invoke-static {v0},Lcmb/pbi;->t(Ljava/lang/String;)Ljava/lang/String;
move-result-object v2
```

其中v2就是保存调用t方法后返回的String类型的字符串。

> ### smali中的if语句

```java
.method private ifRegistered()Z
  .locals 2    //此函数中的本地寄存器的个数
  .prologue
  const/4 v0,0x1   //v0赋值为1
  .local v0,tempFlag:Z
  if-eqz v0,:cond_0  //判断v0是否等于0，等于0则跳到cond_0执行
  const/4 v1,0x1    //符合条件分支
  :goto_0    //标签
  retun v1   //返回v1的值
  :cond_0    //标签
  const/4 v1,0x0    //cond_0分支
  goto:goto_0    //跳到goto_0执行 即返回v1的值 改成return v1也可以
.end method
```

> ### smali中的for语句

```java
const/4 v0,0x0   //v0=0
.local v0,i:I
:goto_0
if-It v0,v3,:cond_0  //v0小于v3则跳转到cond_0并执行分支：cond_0
return-void
  :cond_0  //标签
iget-object v1,p0,Lcom/a/MainActivity;->listStrings:Ljava/util/List; //引用对象
const-string v2,"a"
invoke-interface {v1,v2},Ljava/util/List;->add(Ljava/lang/Object;)Z //list是接口，执行接口方法add
add-int/lit8 v0,v0,0x1  //将第二个v0寄存器中的值，加上0x1的值放入第一个寄存器中
goto:goto_0  //回到goto_0标签
```

