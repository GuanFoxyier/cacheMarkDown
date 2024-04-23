> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/joker_zsl/article/details/136534664)

不少网站会在代码中加入‘debugger’，使你F12时一直卡在debugger，这种措施会让新手朋友束手无策。

js中创建debugger的方式有很多，基础的形式有：

①直接创建debugger

```
debugger;
```

②通过eval创建debugger（在虚拟机中创建）

```
eval('debugger');
```

③通过Function创建debugger（在虚拟机中创建）

```


1.  Function('debugger').call()
2.  Function('debugger').apply()
3.  Function().constructor('debugger').apply('action')


```

而我们遇到的debugger，多数是在这些形式的基础上，或配合定时器，或加上循环，甚至经过ob混淆。。。

对于过无限debugger，常用的方法有以下这些：

1、一律不在此处暂停
----------

![](https://img-blog.csdnimg.cn/direct/8352e564ecb64df9b7e25ec0d76e6953.png)

2、添加条件断点
--------

 在debugger处添加一个结果为false的条件，让这个断点不会被运行

![](https://img-blog.csdnimg.cn/direct/77e98f6d37b24999b809f6a4dfa5e4b6.png)

3、替换文件
------

 如果debugger是在js文件中，直接替换文件，删掉debugger，一劳永逸

4、hook
------

对于一些在虚拟机（VM）中生成的debugger，使用‘一律不在此处暂停（Never pause here）’后会出现网站变的很卡的情况，因为有些网站做了检测，这么做会卡爆你的浏览器。这个时候我们就需要hook。

根据debugger创建方式的不同，使用不同的hook代码

### 4.1、hook evel

```


1.  eval_back = eval
2.  eval = function (args) {
3.      if (args.includes('debugger')) {
4.          return null
5.      } else {
6.          return eval_back(args);
7.      }
8.  }


```

### 4.2、hook constructor

```


1.  Function.prototype._constructor = Function.prototype.constructor;
2.  Function.prototype.constructor = function() {
3.      if(arguments.toString().includes('debugger')){
4.          return null
5.      }
6.     return Function.prototype._constructor.apply(this,arguments);
7.  }


```

### 4.3、hook setInterval

```


1.  setInterval_back = setInterval
2.  setInterval = function(a,b){
3.      if(a.toString().includes('debugger')){
4.        return null;
5.      }
6.      return setInterval_back(a, b);
7.  }


```

要注意注入hook代码要在执行debugger之前，所以要先在debugger前面的地方打断点，再刷新页面注入hook代码

5、油猴插件
------

有些网站，刷新之后会新开虚拟机，之前打下的断点就没有了，无法在执行debugger之前注入hook。对于这种情况，可以用油猴写hook脚本来监控网站，不用提前打断点。