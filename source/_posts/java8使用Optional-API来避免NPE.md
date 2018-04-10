---
title: java8使用Optional API来避免NPE
date: 2018-03-19 14:09:32
tags: [java,NPE]
categories: [java]
---
> 写过`Java`程序的同学,都遇到过`NullPointerException`,为了防止空指针异常，程序中往往要添加很多非`Null`判断。这些判断不仅麻烦，还影响程序的欣赏性和可读性。既然非空判断不可避免，那有没有合适的框架来优雅的解决这一问题呢，`java8`之前，官方并未提供这样的语法糖，可以使用`Guava`等外部API来解决这一问题。直到`java8`提供了`Optional API`。
《阿里巴巴JAVA开发手册》也推荐开发者使用`Optional API`来避免`NPE`。

{% asset_img 开发手册-1.jpg 开发手册 %}
<!-- more -->
![开发手册-2.jpg](开发手册-2.jpg)

# Optional类
可以将`Optional`类理解成一个容器,他将一个未知变量封装起来（这个变量可能为空），向外界提供了方便的`API`来获取这个变量，获取变量时`Optional`会进行`null`的检查。
![Optional类](Optional.png)
## 构造一个 Optional
* `Optional.of(T value)`:该方法通过一个非`null`的`value`来构造一个`Optional`，返回的`Optional`包含了`value`这个值。对于该方法，传入的参数一定不能为`null`，否则便会抛出`NullPointerException`。
* `Optional.ofNullable(T value)`:通过一个`value`来构造一个`Optional`,该`value`可能为`null`。
* `Optional.empty()`:构造一个空的`Optional`,即封装的`value`为`null`。有时方法需要返回`null`时可以用此法构造。

`Optional`的`isPresent()`方法用来判断是否包含值，`get()`用来获取`Optional`包含的值 —— 值得注意的是，如果值不存在，将会抛出 NoSuchElementException 异常。
假设从数据库取某个`User`实体`getUser(Long id)`已经是个客观存在的不能改变的方法，那么利用 `isPresent` 和 `get` 两个方法，我们现在能写出下面的代码：
```java 
Optional<User> user = Optional.ofNullable(getUser(id));
if (user.isPresent()) {
    String username = user.get().getUsername();
	// 使用 username
    System.out.println("Username is: " + username); 
}
```
看上去貌似优雅了点，但和`if (user != null)`没有多大区别，而且还因为封装增加了代码量。所以我们来看看`Optional`还提供了哪些方法，让我们以正确的姿势使用`Optional`。

## ifPresent
![ifPresent](ifPresent.png)
如果`optional`中的值非空，则调用给定的函数，否则什么也不做。那我们上面的例子可以这么写：
```java 
Optional<User> user = Optional.ofNullable(getUser(id));
user.ifPresent(u -> System.out.println("Username is: " + u.getUsername()));
``` 
是不是要比上面优雅的多。如果将打印用户名称封装成静态函数`UserUtil.printName(User user)`，则调用变为`user.ifPresent(UserUtil::printName);`
## orElse
![orElse](orElse.png)
如果`optional`中的值非空，则返回该值，否则返回一个默认值(与`get()`比较)。该函数适合在需要默认值的场景中使用。
```java 
Optional<User> user = Optional.ofNullable(getUser(id));
System.out.println("Username is: " + user.orElse(new User("Unknown")).getUsername());
``` 
## orElseGet
![orElseGet](orElseGet.png)
有时候不是需要简单的一个默认值，而是要一些操作之后得到默认值，这就需要`orElseGet`函数。
```java
Optional<User> user = Optional.ofNullable(getUser(id));
System.out.println("Username is: " + user.orElseGet(() -> {
            //do something
            return new User();
        }).getUsername());
``` 
## orElseThrow
![orElseThrow](orElseThrow.png)
与`orElse`不同的是，`orElseThrow`方法当`Optional`中的值非空时，返回该值；否则抛出异常。该函数适合在需要抛异常的场景中使用。
```java 
Optional<User> user = Optional.ofNullable(getUser(id));
System.out.println("Username is: " + user.orElseThrow(() -> {
			return new EntityNotFoundException("用户不存在:"+id);
		}).getUsername());
``` 
## map
![map](map.png)
如果`optional`中的值为空，则返回一个空的`optional`，否则返回一个新的`Optional`,该`Optional`的值是处理后的值。有多层次调用时需要用到该函数。
假设人员有上下级关系，某`user`的上级为`leader`，要打印`leader`的名称，代码如下：
```java 
Optional<String> leaderName = Optional.ofNullable(getUser(id))
				.map(User::getLeader)
                .map(User::getUserName)
                .map(String::toLowerCase);
System.out.println(leaderName.orElse("Unknown"));
``` 
对于多层级的调用，`Optional API`会更加优雅。
## flatMap
![flatMap](flatMap.png)
与`map`不同的是，`flatMap`在调用`mapper`后返回的必须是`Optional`对象，而`map`会自动将`mapper`返回的值包装成`Optional`对象。
```java 
Optional<String> leaderName = Optional.ofNullable(getUser(id))
				.flatMap(user -> Optional.of(user.getLeader()))
                .flatMap(leader -> Optional.of(user.getUserName()))
                .flatMap(userName -> Optional.of(userName.toLowerCase()));
System.out.println(leaderName.orElse("Unknown"));
``` 
## filter
![flatMap](flatMap.png)
`filter`方法接受一个`Predicate`来对`Optional`中包含的值进行过滤，如果包含的值满足条件，那么还是返回这个`Optional；否则返回`Optional.empty`。
```java 
Optional<String> leaderName = Optional.ofNullable(getUser(id))
				.map(User::getLeader)
                .map(User::getUserName)
                .map(String::toLowerCase)
				.filter(userName -> !userName.equals("Boss"));
System.out.println(leaderName.orElse("Unknown"));
``` 
## Optional对Collection的处理
假设`getUsers(Collection ids)`返回`User`对象集合,那么怎么去除集合里面的空对象呢代码如下：
```java 
public List<User> getUsers(Collection<Integer> userIds) {
       return userIds.stream()
            .map(id -> Optional.ofNullable(getUser(id))) // 获得 Stream<Optional<User>>
            .filter(Optional::isPresent)// 去掉不包含值的 Optional
            .map(Optional::get)
            .collect(Collectors.toList());
}
``` 
`java9`已经对上面的处理做了增强，这里不做讨论。
# 总结
`Optional`能帮助我们在代码里优雅的处理`NPE`问题，虽然这种方法并没有提高程序的效率，但在绝大部分情况下不会影响程序。如果还没有用到`java8`的同学，可以考虑引入`Google`的`Guava`库 —— 事实上，早在`Java6`的年代，`Guava`就提供了`Optional`的实现。
# 参考
https://segmentfault.com/a/1190000008692522
https://www.zhihu.com/question/47997295

