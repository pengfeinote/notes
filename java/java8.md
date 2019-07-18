## java8

### lambda表达式


为什么lambda只能访问final

### interface中的default和static
1. <b>static方法</b>  
与普通类的静态方法类似，可以设置域修饰符(public, protected, private)，只能通过接口静态调用
2. <b>default方法</b>  
接口的默认实现，实现接口的类如果不重写这个函数，则调用接口函数；如果类实现了两个接口，两个接口有同样命名的default函数，则类必须重写该函数
