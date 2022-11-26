# Chain of Responsibility Pattern


<h1>借助Spring实现责任链</h1>

## 需求

### 对请求注册的数据做一些前置处理

这个需求很常见。而且Spring也提供了相应的注解来解决此类问题。但注解通常只负责解决格式上的问题。像不允许重名、不允许重复注册这类需要借助数据库来实现的功能就只能在代码层面来解决了。

## 解耦

设计模式的出现很大程度上都是为了解耦。

面向对象编程提倡的一个关键的点是开闭原则。即对修改关闭，但对扩展开放（~~emmmm有些机翻的味道，大概知道什么意思就行~~）。

所以在数据的前置校验中，我不希望把逻辑直接添加到注册的功能中。更希望的是用一行代码来执行前置处理。把前置处理的逻辑都放在幕后。

最后的代码长这样：

```java
 public CommonResult<String> register(UserDto userDto){
        CommonResult<String> handle = handleRegister.handle(userDto);
        if (handle != null) return handle;
       	// 省略注册的逻辑实现
    }
```

## 面向接口编程

> 其实初学的时候完全不能理解为什么要这么做，感觉很多此一举。根据后来的经验发现，觉得多此一举而又被业界承认的观点往往是为了方便某一场景。

### 你可以把它看成一个链表

事实上责任链这个名字也暗含着这层意思。数据从链的这一头出发，途径各个过滤器。就好像在遍历链表一样。

![image-20221126231316185](/责任链.jpg)

中途如果不符合某个过滤器的“心思”，那么就会遭受它的“制裁”。或是对被过滤的东西做修剪，最糟糕的情况下会直接将其丢弃。就像流水线上的空盒子会被检测机器或人工丢弃一样。

### 把它当成链表来实现

我们可以定义一个抽象类。

```java
public abstract class HandleRegister {

    protected HandleRegister next;

    public abstract CommonResult<String> handle(UserDto userDto);

    public void setNext(HandleRegister handleRegister){
        next = handleRegister;
    }

    public CommonResult<String> pass(HandleRegister handleRegister, UserDto userDto){
        if (handleRegister == null) return null;
        else return handleRegister.handle(userDto);
    }


}
```

这个抽象类定义了（请记住这里是定义，毕竟抽象类就是用来定义或者搭建一个类的基础骨架的。至于为什么还请继续看下去。）一个`next`字段。它的意思是指当前过滤器的下一个过滤器。

作为一个过滤器，他们都有一个名为“**过滤**”的举动。只是不同的过滤器的执行逻辑不同。所以我们把`handle`的实现交给子类。

你可能会很好奇`pass(HandleRegister handleRegister, UserDto userDto)`方法是用来干什么的。是的，这就是为什么我不直接采用接口来约定“**过滤**”这一举动的原因。

假设没有这个方法，那每一个子类在执行完过滤逻辑之后应当怎么做呢？

```
// 执行过滤逻辑
doFilter();
// 假设被过滤的数据安全的度过了这个过滤器
// 如果这个过滤器是整条链上的最后一个节点，那么证明被过滤的数据安全的度过了审核
if (next == null) return true;
// 不然的话就交给下一个过滤器
else next.handle(data);
```

可以看到，每个执行器执行过滤之后都要做相同的“动作”——把待过滤的东西交给下一个。

既然它们这个动作都一样，那我们就可以将这个动作抽象出来。把它看成过滤器应当具有的一个“行为”。

所以就有了上面代码中的`pass(HandleRegister handleRegister, UserDto userDto)`。它其实就是这个**传递**行为的定义！

### 子类会长什么样？

让我们来看看校验重复昵称的过滤器：

```java
@Component
public class CheckName extends HandleRegister {

    @Autowired
    UserDao userDao;

    @Override
    public CommonResult<String> handle(UserDto userDto) {
        int count = userDao.countByName(userDto.getName());
        if (count == 1) return CommonResult.failed(REPEAT_NAME);
        else return pass(next, userDto);
    }


}
```

- 需要强调的是，每个子类我们都会将其放置到IOC容器中。毕竟本章的标题是“借助**Spring**”。

可以看到，子类只需要实现“**过滤**”的逻辑即可。

- 这里不要忘记了。每个子类都还从抽象类中继承者一个`next`和一个`setNext()`方法。

### 怎么把它们组装起来？

接下来就需要借助Spring的能力了。

```java
@Configuration
public class InitHandleChain {

    @Autowired
    List<HandleRegister> handleRegisters;

    @Bean(name = "registerChain")
    public HandleRegister buildFirst(){
        for (int i = 0; i < handleRegisters.size() - 1; i++) {
            handleRegisters.get(i).setNext(handleRegisters.get(i + 1));
        }
        return handleRegisters.get(0);
    }
}
```

由于子类都是继承自同一抽象类。所以它们的“类型”是一致的。我们可以借助Spring将它们一并注入到打造链条的地方。（Spring真的是一个非常好的框架！！）

`buildFirst`的逻辑非常的简单。就是组装过滤器，让他们形成一个链条并且在最后返回链条的头指针！

## 乱七八糟的看完了，好处呢？

如果你还记得我前面说过的话的话，你应该可以发现。如果在之后的时间中你突发奇想或者被迫需要加入或删除一个过滤器的话。你完全不需要动注册方法中的代码。只需要新建一个过滤器或者删掉一个过滤器。为此你可以把你更多的精力放在**过滤**的逻辑上而不是重新到业务代码中阅读并定位你究竟得对代码做怎样的删除或修改。

如果你是一个规划能力较弱的人——就如同我一样。在写之前没有考虑到所有情况的话，那么你会发现开闭原则拯救了你！

> 哦对了，上面的代码都是我在MCSW项目中的写的。如果你感兴趣的话，就到我[主页](https://github.com/SAYAKA-Konami/MCSW)去看看吧


