## 建造者(Builder)模式

1. 简单的建造者模式一般用于参数比较多的类当中。当一个类参数比较多时，构造函数难于实现，因为如果使用无参构造函数，设置参数太麻烦，如果使用有参构造函数，构造函数的数量会比较多，看起来比较繁杂，使用建造者模式，可以方便的构造多参类的对象。
2. 复杂的建造者模式将一个对象的构建与表示相分离，通过建造者模式可以构建出不同的表示
3. 简单builder模式示例

>类示例

```
public class Human {
	private string head;
	private string body;
	private string foot;
	
	private Human() {
	}
	
	private Human(Builder builder) {
		this.head = builder.head;
		this.body = builder.body;
		this.foot = builder.foot;
	}
	
	public static class Builder() {
		private string head;
		private string body;
		private string foot;
		
		public Builder withHead(string head) {
			this.head = head;
		}
		
		public Builder withBody(string body) {
			this.body = body;
		}
		
		public Builder withFoot(string foot) {
			this.foot = foot;
		}
		
		public Human build() {
			return new Human(this);
		}
	}
}
```

>类使用

```
Human human = new Human.Builder()
	.withHead(head).build();
```


4. 复杂builder模式示例(Build类为虚类，Build(args)生成不同的Builder然后传不同的参数，然后生成不同的对象)