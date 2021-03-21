## 【SpringBoot】那些遇到过的坑

在学习和使用 spring boot 的过程中, 遇到了无数的坑, 该篇文章记录和总结这些坑, 往者不可谏, 来者犹可追...

### 1. yaml配置文件中特殊字符

之前在 application.yml 中配置了 actuator web监控的范围, 将其值改为 *, 但是 * 是特殊字符, 在 yaml 中需要加上单引号, 因为没加, 报了个深坑错误, 排查了好久好久....

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

这个 * 值未加单引号, 报错如下

![boot_error_1](/Users/sherlock/Desktop/notes/allPics/SpringBoot/boot_error_1.png)

但从错误看, 一开始会认为是 jdk 的 class 路径的问题, 但是这个错就是 特殊字符转义的错误...



### 2. Springboot单元测试指定入口类的问题

在开始学习的时候, 在单元测试中 @StringBootTest 并未指定值,  但是在测试时,一直读不到配置文件中绑定到对象上的值.

经过排查后, 发现需要在 @StringBootTest(classes = {Application.class}) 指定 SpringBoot 的启动类

