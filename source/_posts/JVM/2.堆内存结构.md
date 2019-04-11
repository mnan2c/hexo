title: 堆内存结构

tags:
  - JVM

categories:
  - JVM

---
## 1. 结构
JVM内存结构主要有三大块：堆内存、方法区和栈，堆内存是JVM中最大的一块。
![](/img/java/jvm-struct.png)
1. 新生代（Young Generation）:存储新创建的对象，通常占堆内存的1/3
  - Eden区
  - Survivor区
    - From Survivor（Survivor 0）
    - To Survivor（Survivor 1）
2. 老年代（Old Generation）：存储较大或生存时间久的对象
3. 永久代

补充：默认Eden：From Survivor：To Survivor的比例是8:1:1

## 2. 对象创建之后的流转
1. 对象A被new出来之后，是被存放在Eden区的。
2. 当发生一次GC之后，Eden区存活下来的对象A会被复制到Survivor 1区；Survivor 0 中存活的对象也会被复制到Survivor 1中。
3. GC会清空Eden和Survivor 0 中存储的所有对象。
4. 交换Survivor 0和Survivor 1的角色：即此时有数据的Survivor 1作为From Survivor，被清空的Survivor 0作为To Survivor。要保证在GC发生之前，To Survivor永远是空的那个。
5. 下次GC发生时，重复上述步骤。

注意：发生GC时，From Survivor中存活的对象并不是全部都会被复制到To Survivor中，而是根据这个对象在Survivor区中存活了多久而决定去向，当一个对象在Survivor中存活了很久（即经历了多次GC还没死），就会在发生GC时被复制到旧生代中。

## 3. 相关问题
为什么对象在新生代中复制来复制去的，而不是将死的直接清除，老的直接复制到旧生代？
： 这样做的好处就是减少了内存碎片，而直接清除的话会使内存很零碎。
