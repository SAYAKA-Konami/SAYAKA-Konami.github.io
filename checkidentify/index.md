# MCSW-Check identify of user


<h1>MCSW--Chapter One</h1>

![image-1](/take-it-easy.png)

<h2 align="center">MCSW面向的用户</h2>

我希望打造的是一个面向学校内部的讨论平台。考虑的主要因素如下：

从宏观角度出发，毕业生的去向和学校的排名是息息相关的。虽然一定程度上会局限信息的覆盖范围，但是对于在校的学生会更有参考价值。——在论文中应当采用SCAU和其他高校毕业生去向数据的横向对比。

<h2 align="center">后端服务的划分</h2>

首先需要介绍一下微服务：

![image-2](/微服务.png)

根据上述微服务的特性。我暂且将 ***MCSW*** 的后端服务板块划分成了如下样式

![image-服务拆分](/服务拆分.png)

## 账号注册与管理

**这是本章节的论述范围。**

该模块具体负责的是用户的注册与权限的管理。其中最大的问题就是如何限制用户群体的“面貌”。

我的解决方案是利用学校官网的信息门户的登录功能来对用户做甄别。

下图是学校信息门户的登录界面：

![image-3](/信息门户.png)

由于每个学生的学号都是唯一的。所以可以在用户注册时令其提供相应的账号信息，然后把“鉴别”的任务交给学校官网提供的登录接口。

### 利用HTTP抓包的形式查看登录信息门户所需要的参数及对应的URL

利用浏览器提供的开发者工具查看点击登录后的HTTP请求：

![image-20221125172705719](/抓包1.png)



在上图中我们可以看到该登录接口对应的URL。

![image-20221125172903447](/抓包2.png)

与之对应的是登录请求中携带的参数。可以看到的是密码被做了加密处理。

以下是请求所需的参数名：

| username | password | captcha | warn | It   | execution | _eventId | submit |
| -------- | -------- | ------- | ---- | ---- | --------- | -------- | ------ |

#### 为啥要在前端加密？

我猜测应该是为了防止中间人攻击所以先做了一次加密。但是可以肯定的是，既然是在页面发送时就做了加密，那么相应的代码中就会有体现加密的过程。

#### 返回登录页面查看页面结构：

![image-20221125174422034](/信息门户html-js.png)

可以看到除了浏览器渲染所需要的html文件还有几个对应的JS文件。如果你接触过计网的话应该有听过***RSA*** ——一种非对称性加密算法。显然我们填入的密码就应当是给该JS中的函数加密之后再发送出去的。

### 查看HTML中关于密码的加密措施

如下是 ***login.html*** 中关于密码加密的代码：

```html
 $(".btn-submit").click(function(){
		$("#password").attr({maxlength:"1000"});
	    var thisPwd = document.getElementById("password").value;  
		if(thisPwd.length != 256){
		    setMaxDigits(131);
    	    var key = new RSAKeyPair("010001", '', "008e9fdac2a933c27a8262eb0ab8004aa74571e1e7c27beb436ce17c37df778d8861a9a2afddc04a6e80da995e34754e1e002864f2480f0471257880b55359e8232601244593333eb9f0f99b894fe13538a80bfd14aeb94bb8108959140231195a9e9f488f7d5cc72a112d6a19576cb05eaf629435538907ccc9b008d64595646d");
	  		var result = encryptedString(key, encodeURIComponent(thisPwd));
	  		$("#password").val(result);
	  	}    	
    });	
```

可以看到它调用了先前看到的RSA.js中的函数。

让我们来看一看RSA.js中的代码：

```javascript
function RSAKeyPair(encryptionExponent, decryptionExponent, modulus)
{
	this.e = biFromHex(encryptionExponent);
	this.d = biFromHex(decryptionExponent);
	this.m = biFromHex(modulus);
	// We can do two bytes per digit, so
	// chunkSize = 2 * (number of digits in modulus - 1).
	// Since biHighIndex returns the high index, not the number of digits, 1 has
	// already been subtracted.
	this.chunkSize = 2 * biHighIndex(this.m);
	this.radix = 16;
	this.barrett = new BarrettMu(this.m);
}

function twoDigit(n)
{
	return (n < 10 ? "0" : "") + String(n);
}

function encryptedString(key, s)
	// Altered by Rob Saunders (rob@robsaunders.net). New routine pads the
	// string after it has been converted to an array. This fixes an
	// incompatibility with Flash MX's ActionScript.
{
	var a = new Array();
	var sl = s.length;
	var i = 0;
	while (i < sl) {
		a[i] = s.charCodeAt(i);
		i++;
	}

	while (a.length % key.chunkSize != 0) {
		a[i++] = 0;
	}

	var al = a.length;
	var result = "";
	var j, k, block;
	for (i = 0; i < al; i += key.chunkSize) {
		block = new BigInt();
		j = 0;
		for (k = i; k < i + key.chunkSize; ++j) {
			block.digits[j] = a[k++];
			block.digits[j] += a[k++] << 8;
		}
		var crypt = key.barrett.powMod(block, key.e);
		var text = key.radix == 16 ? biToHex(crypt) : biToString(crypt, key.radix);
		result += text + " ";
	}
	return result.substring(0, result.length - 1); // Remove last space.
}

function decryptedString(key, s)
{
	var blocks = s.split(" ");
	var result = "";
	var i, j, block;
	for (i = 0; i < blocks.length; ++i) {
		var bi;
		if (key.radix == 16) {
			bi = biFromHex(blocks[i]);
		}
		else {
			bi = biFromString(blocks[i], key.radix);
		}
		block = key.barrett.powMod(bi, key.d);
		for (j = 0; j <= biHighIndex(block); ++j) {
			result += String.fromCharCode(block.digits[j] & 255,
			                              block.digits[j] >> 8);
		}
	}
	// Remove trailing null, if any.
	if (result.charCodeAt(result.length - 1) == 0) {
		result = result.substring(0, result.length - 1);
	}
	return result;
}
```

代码理解起来需要一定的密码学知识。所以我这里不打算用Java去复现这段代码。我打算在java中执行这段代码。

### ScriptEngine

得益于JDK内置的脚本解析器。我们直接将所需要的JS文件放置到项目下然后改写一下 ***login.html*** 中有关于加密密码的代码就可以得到同样的结果。



```
// 测试类
@Test
    public void printPasswd() {
        String[] path = new String[]{"js/code.js", "js/RSA.js", "js/BigInt.js", "js/Barrett.js"};
        String passwd = "wn485233";
        //用脚本解释器，使用javascript解释器
        ScriptEngineManager manager = new ScriptEngineManager();
        ScriptEngine engine = manager.getEngineByName("javascript");
        try {
            for (String p : path) {
                ClassPathResource classPathResource = new ClassPathResource(p);
                engine.eval(new InputStreamReader(classPathResource.getInputStream()));
            }
            // 操作js函数
            passwd = (String) engine.eval("generatePasswd('" + passwd + "')");
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println(passwd);

    }
```


