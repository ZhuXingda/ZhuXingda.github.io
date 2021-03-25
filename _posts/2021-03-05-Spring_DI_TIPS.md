---
layout:       post
title: Spring在Bean初始化结束后手动依赖注入
subtitle:     ""
date:         2021-03-05
author:       "ZhuXingda"
header-mask:  0.3
catalog:      true
multilingual: true
comments: true
tags:
    - Spring
    - java
---
>遇到的情况：三个用Spring进行依赖管理的类A、B、C，依赖关系如下
>```java
>@Component
>class A{
> 	List<B> bList;
> 	@PostConstruct
> 	void init(){
> 		for(int i,i<10,i++){
> 			bList.add(new B(i));
> 		}
> 	}
>        ...
> }
> 
> @Component
> class B{
>        int i;
> 	@Autowired
> 	C c;
>        ...
> }
> 
> @Component
> class C{...}
>```
在这种情况下如果用手动调用构造方法创建B类对象，显然会导致bList中
B对象的C域为null, 常见的情况不止这一种类型，总结起来就是本来由Spring
管理的类却有一些属性必须由调用类来初始化（产生了耦合），本质上就是一个对象设计上的
问题，最好的办法是重新设计各个类的职责来消除耦合。

不消除耦合的解决办法：
```java
@Component
class A  implements ApplicationContextAware {
    public static ApplicationContext applicationContext;
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        A.applicationContext = applicationContext;
    }

 	List<B> bList;
 	@PostConstruct
 	void init(){
 		for(int i,i<10,i++){
            B b = (B)applicationContext.getBean("b");
            b.setI(i);
 			bList.add(b);
 		}
 	}
        ...
 }
 
 @Component
 @Scope(value = "prototype")
 class B{
        int i;
 	@Autowired
 	C c;
        ...
 }
 
 @Component
 class C{...}
```
即手动调用applicationContext来依赖注入，可以解决问题但不是好方法。

