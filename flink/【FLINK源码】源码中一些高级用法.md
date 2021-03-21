## 【FLINK源码】源码中一些高级用法

### 1. java.util.ServiceLoader

ServiceLoader的使用是要在根目录有一个文件夹META-INF/services/，其主要是对这个目录进行扫描，文件名是你需要提供服务的类（接口）全称（即包名.类名）。类中的内容，一行就是改接口的一个具体的实现类。包结构如下：

#### 例子

ServiceLoader的使用是要在根目录有一个文件夹META-INF/services/，其主要是对这个目录进行扫描，文件名是你需要提供服务的类（接口）全称（即包名.类名）。类中的内容，一行就是改接口的一个具体的实现类。包结构如下：

```java
src/main/resources/META-INF/services/com.github.test.IService
```

```java
package com.github.test;
public interface IService {
	public String sayHello();
}

package com.github.test
public class ServiceImpl1 implements IService {
	@Override
	public String sayHello() {
		return "hello, test1";
	}
}

package com.github.test
public class ServiceImpl2 implements IService {
	@Override
	public String sayHello() {
		return "hello, test2";
	}
}
```

文件***com.github.test.IService***内容: (配置文件就是实现类的类全称:)

```java
com.java.util.test.ServiceImpl1
com.java.util.test.ServiceImpl2
```

配置完成后就是最主要的使用方法了：

```java
package com.github.test;
public class ServiceLoaderTest {
	public static void main(String[] args) {
		ServiceLoader<TestService> loader = ServiceLoader.load(TestService.class);
		for(TestService service : loader) {
			System.out.println(service.sayHello());
		}
	}
}
```

运行结果:

```
hello, test1
hello, test2
```

### 2. Collections.unmodifiableList()

考虑线程安全问题, 返回不可变List

### 3. Callable\<T> 

使用***Call***<T>作为接口中的的方法参数,来实现回调

