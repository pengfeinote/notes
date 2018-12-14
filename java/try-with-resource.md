1. 如果一个外部资源的句柄对象（如InputStream）实现了AutoCloseable接口，那么可以将资源的关闭写成以下形式：
	try (Inputstream in = new FileInputStream()) {
	} catch (Exception e) {
	}
	当try catch被调用后，java会确保资源的close方法被调用
2. 实现原理:只是jdk 1.7的一个语法糖，字节码与源代码相同
