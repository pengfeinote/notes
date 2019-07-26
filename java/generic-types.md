## java泛型

### 编译时 or 运行时

java的泛型在编译时起作用，在运行时被擦除。

1. 在编译时起作用：

```java
List<Integer> list = new ArrayList<>();
list.add("string");
```
如上，你无法将一个字符串赋值给一个泛型为Integer的list

2. 在运行时被擦除

```java
List<Integer> l1 = new ArrayList<>();
List<String> l2 = new ArrayList<>();

System.out.println(l1.getClass() == l2.getClass());
//true

System.out.println(Arrays.toString(l1.getClass().getTypeParameters()));
//[E]
```
生成的字节码中并不会存储泛型的具体类型，仅仅存储了声明泛型时的泛型参数，所以通过getTypeParameters仅获取到了[E]

java之所以不在字节码中存储泛型信息，是为了避免系统中生成太多的类造成运行时太多消耗加重系统负担


### 如何获取泛型信息： 匿名内部类
