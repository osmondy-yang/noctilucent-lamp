## 如何解决循环依赖问题

循环依赖问题在Spring中主要有三种情况：

- （1）通过构造方法进行依赖注入时产生的循环依赖问题。
- （2）通过setter方法进行依赖注入且是在多例（原型）模式下产生的循环依赖问题。
- （3）通过setter方法进行依赖注入且是在单例模式下产生的循环依赖问题。

在Spring中，只有第（3）种方式的循环依赖问题被解决了，其他两种方式在遇到循环依赖问题时都会产生异常。其实也很好解释：

- 第（1）种构造方法注入的情况下，在new对象的时候就会堵塞住了，其实也就是”先有鸡还是先有蛋“的历史难题。
- 第（2）种setter方法（多例）的情况下，每一次getBean()时，都会产生一个新的Bean，如此反复下去就会有无穷无尽的Bean产生了，最终就会导致OOM问题的出现。

## Spring bean生命周期13个环节

1. 先从缓存中获取
2. 获取不到实例化CLassA
3. 依赖注入CLassA的成员变量
4. 初始化ClassA
5. 将ClassA放入缓存