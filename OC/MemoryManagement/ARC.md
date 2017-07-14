苹果在Xcode 4.2提供了ARC(Automatic Reference Counting)支持，自动管理内存。在ARC之前，是MRC(manual reference counting)，程序员需要通过显示调用retain/release来管理引用计数。ARC与MRC的不同在于ARC是在编译期自动添加代码来保证对象的生命周期。

# 基本内存管理原则
内存管理原则基于对象的所有权，当对象有至少一个所有者时，就保持存在。当没有所有者时，则销毁。
- 你拥有你创建的对象。(比如通过alloc\new\copy\mutableCopy创建)
- 你可以通过retain方法拥有一个对象。
- 当不需要一个对象时，需要释放所有权。
- 你不能释放你并不拥有的对象。

# NSObject协议

# 
