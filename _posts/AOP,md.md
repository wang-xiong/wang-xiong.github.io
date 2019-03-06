AOP

AOP是Aspect Oriented Programming，译为面向切面编程。

纵向关系OOP，横向关系AOP

AOP方法

Android AOP常用的方法有JNI Hook和静态织入。

**动态织入Hook方式**

1.Dexposed

2.Xposed

3.epic 在native层修改java method对应的native指针

动态字节码生成

1.Cglib+Dexmaker

Cglib 是一个强大的,高性能的 Code 生成类库， 原理是在运行期间目标字节码加载后，通过字节码技术为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。由于是通过子类来代理父类，因此不能代理被 final 字段修饰的方法。

但是 Cglib 有一个很致命的缺点：底层是采用著名的 ASM 字节码生成框架，使用字节码技术生成代理类，也就是通过操作字节码来生成的新的 .class 文件，而我们在 Android 中加载的是优化后的 .dex 文件，也就是说我们需要可以动态生成 .dex 文件代理类，因此 Cglib 不能在 Android 中直接使用。有大神根据 Dexmaker 框架（dex代码生成工具）来仿照 Cglib 库动态生成 .dex 文件，实现了类似于 Cglib 的 AOP 的功能。详细的用法可参考：[将cglib动态代理思想带入Android开发](https://link.juejin.im?target=http%3A%2F%2Fblog.csdn.net%2Fzhangke3016%2Farticle%2Fdetails%2F71437287)

**静态织入方式**

- 在编译期织入，切面直接以字节码的形式编译到目标字节码文件中，，这要求使用特殊的 Java 编译器。
- 在类装载期织入，这要求使用特殊的类装载器。

1. APT
2. AspectJ
3. ASM
4. Javassist
5. DexMaker
6. ASMDEX

![img](https://user-gold-cdn.xitu.io/2018/11/30/167652d47ef065f1?imageslim)



