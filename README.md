# 1212300204_p4
http://lxr.oss.org.cn/source/Documentation/driver-model/design-patterns.txt

  2 Device Driver Design Patterns
  3 ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  4 
  5 This document describes a few common design patterns found in device drivers.
  6 It is likely that subsystem maintainers will ask driver developers to
  7 conform to these design patterns.
  8 
  9 1. State Container
 10 2. container_of()
 11 
 12 
 13 1. State Container
 14 ~~~~~~~~~~~~~~~~~~
 15 
 16 While the kernel contains a few device drivers that assume that they will
 17 only be probed() once on a certain system (singletons), it is custom to assume
 18 that the device the driver binds to will appear in several instances. This
 19 means that the probe() function and all callbacks need to be reentrant.
 20 
 21 The most common way to achieve this is to use the state container design
 22 pattern. It usually has this form:
 23 
 24 struct foo {
 25     spinlock_t lock; /* Example member */
 26     (...)
 27 };
 28 
 29 static int foo_probe(...)
 30 {
 31     struct foo *foo;
 32 
 33     foo = devm_kzalloc(dev, sizeof(*foo), GFP_KERNEL);
 34     if (!foo)
 35         return -ENOMEM;
 36     spin_lock_init(&foo->lock);
 37     (...)
 38 }
 39 
 40 This will create an instance of struct foo in memory every time probe() is
 41 called. This is our state container for this instance of the device driver.
 42 Of course it is then necessary to always pass this instance of the
 43 state around to all functions that need access to the state and its members.
 44 
 45 For example, if the driver is registering an interrupt handler, you would
 46 pass around a pointer to struct foo like this:
 47 
 48 static irqreturn_t foo_handler(int irq, void *arg)
 49 {
 50     struct foo *foo = arg;
 51     (...)
 52 }
 53 
 54 static int foo_probe(...)
 55 {
 56     struct foo *foo;
 57 
 58     (...)
 59     ret = request_irq(irq, foo_handler, 0, "foo", foo);
 60 }
 61 
 62 This way you always get a pointer back to the correct instance of foo in
 63 your interrupt handler.
 64 
 65 
 66 2. container_of()
 67 ~~~~~~~~~~~~~~~~~
 68 
 69 Continuing on the above example we add an offloaded work:
 70 
 71 struct foo {
 72     spinlock_t lock;
 73     struct workqueue_struct *wq;
 74     struct work_struct offload;
 75     (...)
 76 };
 77 
 78 static void foo_work(struct work_struct *work)
 79 {
 80     struct foo *foo = container_of(work, struct foo, offload);
 81 
 82     (...)
 83 }
 84 
 85 static irqreturn_t foo_handler(int irq, void *arg)
 86 {
 87     struct foo *foo = arg;
 88 
 89     queue_work(foo->wq, &foo->offload);
 90     (...)
 91 }
 92 
 93 static int foo_probe(...)
 94 {
 95     struct foo *foo;
 96 
 97     foo->wq = create_singlethread_workqueue("foo-wq");
 98     INIT_WORK(&foo->offload, foo_work);
 99     (...)
100 }
101 
102 The design pattern is the same for an hrtimer or something similar that will
103 return a single argument which is a pointer to a struct member in the
104 callback.
105 
106 container_of() is a macro defined in <linux/kernel.h>
107 
108 What container_of() does is to obtain a pointer to the containing struct from
109 a pointer to a member by a simple subtraction using the offsetof() macro from
110 standard C, which allows something similar to object oriented behaviours.
111 Notice that the contained member must not be a pointer, but an actual member
112 for this to work.
113 
114 We can see here that we avoid having global pointers to our struct foo *
115 instance this way, while still keeping the number of parameters passed to the
116 work function to a single pointer.

这个设计模式对于高精度定时器或者和它相似的地方将会在回调中返回一个指向结构体成员的指针的单个参数。

container_of（）所做的是通过对offsetof（）从标准C的宏中允许类似的面向对象的行为，以获得一个指向包含结构从指针到成员。注意，被包含的成员不能是一个指针，而是一个实际成员。

我们可以从这里看到我们避免全局的指针指向我们的foo结构体实例，同时要保持的传递到功函数为单个指针参数的个数。
