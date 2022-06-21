+++
title = "“逃逸分析”与“TLAB”"
author = "crzbird"
github_url = "https://github.com/crzbird"
head_img = ""
created_at = 2021-02-12T21:35:15
updated_at = 2021-02-12T21:35:15
description = "关于逃逸分析"
tags = ["java","jvm"]
+++
# “逃逸分析”与“TLAB”
## 逃逸分析：
### 案例
方法中创建的对象是否会“逃逸”到方法外，即某个方法中创建的对象是否会被其他方法使用。
如下代码：
![image-85925f48ba74415ba9784483e8bd32e6](https://user-images.githubusercontent.com/40804675/174811009-ce8f74df-224c-4cd6-9f1c-d7d9607b707e.png)

testUnEscape()方法中创建user只进行打印，也就是此对象作用域只在此方法中，并不会逃逸到方法外。
再比如：
![image-d131cbbb288a4a03b5d215e1264fd25e](https://user-images.githubusercontent.com/40804675/174811082-3fc16494-cba3-43a2-9daf-0d52936c9994.png)


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
![image-d0e9d8ed029b4b58bf4130dca3582c51](https://user-images.githubusercontent.com/40804675/174811134-04e447e4-2342-4ebf-9a02-e178fe2608eb.png)

User中的id则为标量，而User对象成为聚合量。如果user不会发生逃逸，如testUnEscape(),首先不会创建user对象，而是使用分解user为id标量，testUnEscape()实际运行为：
![image-11b3d07ade7345849745290599f6dd5f](https://user-images.githubusercontent.com/40804675/174811190-3348f072-3929-4de2-8096-be9409d74bff.png)
### 逃逸分析测试案例（-XX:(+/-)DoEscapeAnalysis）
![image-7a245b02934f4c289464a26ab1d1a58f](https://user-images.githubusercontent.com/40804675/174811238-54dc20af-08ae-4e87-9fe3-7e60a8d3ab37.png)
以上单纯new出100万User对象，接下来查看堆中User对象数量。
1.开启逃逸分析（-XX:+DoEscapeAnalysis，java6后默认开启）：

![image-a3345306ca6540d896f40720abc30c64](https://user-images.githubusercontent.com/40804675/174811278-d8b20770-09ec-4a85-9deb-c5c02bba3528.png)
耗时2ms
![image-b1b4a524cd684ac0a74c548e0a0cd151](https://user-images.githubusercontent.com/40804675/174811349-6861a91a-b4a6-44be-955b-63ef4c9b7e3e.png)
User对象数量为：160871。
2.关闭逃逸分析(-XX:-DoEscapeAnalysis)：
![image-cc0017641ee34d4e85a1362d5ba76289](https://user-images.githubusercontent.com/40804675/174811380-70e19968-6e5d-40a4-bed7-a88132eed8bb.png)
耗时11ms
![image-1a8da62ef71e4db8a6b973446dd673a1](https://user-images.githubusercontent.com/40804675/174811413-686f6d2c-1904-4d46-a80f-cbbcf6edb9b4.png)
User对象数量为：100万。
### 逃逸分析总结：
逃逸分析是否开启直接影响到应用性能，耗时和空间不在一个量级。非逃逸对象随方法出栈销毁，很大程度上减轻GC压力。
## TLAB（Thread Local Allocation Buffer）
众所周知堆内存是线程共享的，多线程对共享资源操作为保证安全则必定需要同步操作（JVM采用CAS和重试来保证多线程对堆内存操作的安全）。而同步操作又必然有性能上损失，由此产生了TLAB线程本地分配缓冲区。每一个线程创建时会在堆上（EDEN区）申请小部分私有空间，默认为eden区的1%。TLAB中有三个指针start（起始）、top（已耗）、end（终止），当top与end碰撞则会触发refill；refill：如end-top小于等于A.size(对象大小)，1.将原来TLAB剩余空间以“filler object”填充（int[]），保证线性。2.在eden区再申请一段TLAB用来存放A对象，如果申请失败则会触发YGC。3.如果对象大于阈值则直接进入老年代：
![image-6957397882e34bcd84d9c41bb21d4f6c](https://user-images.githubusercontent.com/40804675/174811488-bdfa0d0d-51d4-4f59-8055-057aabaa0216.png)
众所周知堆内存是线程共享的，多线程对共享资源操作为保证安全则必定需要同步操作（JVM采用CAS和重试来保证多线程对堆内存操作的安全）。而同步操作又必然有性能上损失，由此产生了TLAB线程本地分配缓冲区。每一个线程创建时会在堆上（EDEN区）申请小部分私有空间，默认为eden区的1%。TLAB中有三个指针start（起始）、top（已耗）、end（终止），当top与end碰撞则会触发refill；refill：如end-top小于等于A.size(对象大小)，1.将原来TLAB剩余空间以“filler object”填充（int[]），保证线性。2.在eden区再申请一段TLAB用来存放A对象，如果申请失败则会触发YGC。3.如果对象大于阈值则直接进入老年代：
jvm默认为0，任何对象都会再新生代分配。
## 对象分配大体过程：
![image-a84b6863f4a84e71bb3ee7aba13008f9](https://user-images.githubusercontent.com/40804675/174811540-f224ae0b-8ed4-44af-b1bb-7eb5f5d10d4f.png)
