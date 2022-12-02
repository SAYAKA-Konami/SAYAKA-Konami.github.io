# Travel


<h1>Clash中的负载均衡</h1>

## Just follow me now

![image-20221202205057392](/Clash插件.png)

![image-20221202205202087](/clash编辑规则.jpg)

在配置文件中键入以下内容：

```yaml
parsers:
  - reg: 'slbable$'
    yaml:
      append-proxy-groups:
        - name: ⚖️ 负载均衡-散列
          type: load-balance
          url: 'http://www.google.com/generate_204'
          interval: 300
          strategy: consistent-hashing
        - name: ⚖️ 负载均衡-轮询
          type: load-balance
          url: 'http://www.google.com/generate_204'
          interval: 300
          strategy: round-robin
      commands:
        - proxy-groups.⚖️ 负载均衡-散列.proxies=[]proxyNames
        - proxy-groups.0.proxies.0+⚖️ 负载均衡-散列
        - proxy-groups.⚖️ 负载均衡-轮询.proxies=[]proxyNames
        - proxy-groups.0.proxies.0+⚖️ 负载均衡-轮询
```

上述文件中有这样一串代码：`reg: slbable$`。如果你熟悉正则表达式的话，那么你应该知道这段代码的意思是匹配结尾为 ***slbable*** 的URL。

![image-20221202205519941](/更改url.jpg)

![image-20221202205639765](/添加标识符.jpg)

![image-20221202205724440](/触发更新.jpg)

![image-20221202205749348](/效果.jpg)

## 为什么要这么做？

机场会给出若干个可用节点。如果我们是手动选择了一个节点A，那么就相当于告诉 ***Clash*** 说向A发送网络请求。A接收到请求之后会代我们转发。

![image-20221202210031177](/负载均衡clash.jpg)

如此其实暗含着两个问题：

1. 机场提供了那么多个请求。我难道白白放着不用吗？
2. 全部请求都打到了同一台服务器上，它速度一慢乃至于宕机的话不就得手动切换了？

### 所以才引入了负载均衡的策略！！！！

![image-20221202210312433](/clash负载均衡演示图.jpg)

