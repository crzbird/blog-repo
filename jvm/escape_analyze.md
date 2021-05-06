+++
title = "“逃逸分析”与“TLAB”"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "123"
tags = ["java","jvm"]
+++
# “逃逸分析”与“TLAB”
## 逃逸分析：
### 案例
方法中创建的对象是否会“逃逸”到方法外，即某个方法中创建的对象是否会被其他方法使用。
如下代码：
![image.png](https://image.bytetrick.com/2020/10/image-85925f48ba74415ba9784483e8bd32e6.png)

testUnEscape()方法中创建user只进行打印，也就是此对象作用域只在此方法中，并不会逃逸到方法外。
再比如：
![image.png](https://image.bytetrick.com/2020/10/image-d131cbbb288a4a03b5d215e1264fd25e.png)

testEscape()方法中创建的user对象返回给调用方，则此对象会逃逸到方法外（此方法为“全局逃逸”）。
### 逃逸状态：
#### 1.全局逃逸（GlobalEscape）
- 即一个对象的作用范围逃出了当前方法或者当前线程，如下场景：
- 对象是一个静态变量
- 对象是一个已经发生逃逸的对象
- 对象作为当前方法的返回值
#### 2.参数逃逸（ArgEscape）
- 即一个对象被作为方法参数传递或者被参数引用，但在调用过程中不会发生全局逃逸，这个状态是通过被调方法的字节码确定的。
#### 3.非逃逸：
- 即方法中的对象没有发生逃逸，即对象总用于仅在此方法范围。
### 逃逸分析所做的优化：
- #### 同步消除：对象不发生逃逸，则对此对象所作的同步操作会被消除。
- #### 栈上分配：对象会作为栈上分配的候选，而非直接分配到堆。
- #### 标量替换：
标量是指最小的数据单元；如：
![image.png](https://image.bytetrick.com/2020/10/image-d0e9d8ed029b4b58bf4130dca3582c51.png)
User中的id则为标量，而User对象成为聚合量。如果user不会发生逃逸，如testUnEscape(),首先不会创建user对象，而是使用分解user为id标量，testUnEscape()实际运行为：
![image.png](https://image.bytetrick.com/2020/10/image-11b3d07ade7345849745290599f6dd5f.png)
### 逃逸分析测试案例（-XX:(+/-)DoEscapeAnalysis）
![image.png](https://image.bytetrick.com/2020/10/image-7a245b02934f4c289464a26ab1d1a58f.png)
以上单纯new出100万User对象，接下来查看堆中User对象数量。
1.开启逃逸分析（-XX:+DoEscapeAnalysis，java6后默认开启）：

![image.png](https://image.bytetrick.com/2020/10/image-a3345306ca6540d896f40720abc30c64.png)
耗时2ms

![image.png](https://image.bytetrick.com/2020/10/image-b1b4a524cd684ac0a74c548e0a0cd151.png)
User对象数量为：160871。

2.关闭逃逸分析(-XX:-DoEscapeAnalysis)：
![image.png](https://image.bytetrick.com/2020/10/image-cc0017641ee34d4e85a1362d5ba76289.png)
耗时11ms
![image.png](https://image.bytetrick.com/2020/10/image-1a8da62ef71e4db8a6b973446dd673a1.png)
User对象数量为：100万。
### 逃逸分析总结：
逃逸分析是否开启直接影响到应用性能，耗时和空间不在一个量级。非逃逸对象随方法出栈销毁，很大程度上减轻GC压力。

## TLAB（Thread Local Allocation Buffer）
众所周知堆内存是线程共享的，多线程对共享资源操作为保证安全则必定需要同步操作（JVM采用CAS和重试来保证多线程对堆内存操作的安全）。而同步操作又必然有性能上损失，由此产生了TLAB线程本地分配缓冲区。每一个线程创建时会在堆上（EDEN区）申请小部分私有空间，默认为eden区的1%。TLAB中有三个指针start（起始）、top（已耗）、end（终止），当top与end碰撞则会触发refill；refill：如end-top小于等于A.size(对象大小)，1.将原来TLAB剩余空间以“filler object”填充（int[]），保证线性。2.在eden区再申请一段TLAB用来存放A对象，如果申请失败则会触发YGC。3.如果对象大于阈值则直接进入老年代：![image.png](https://image.bytetrick.com/2020/10/image-6957397882e34bcd84d9c41bb21d4f6c.png)
jvm默认为0，任何对象都会再新生代分配。
## 对象分配大体过程：
![image.png](https://image.bytetrick.com/2020/10/image-a84b6863f4a84e71bb3ee7aba13008f9.png)