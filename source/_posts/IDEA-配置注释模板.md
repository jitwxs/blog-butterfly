---
title: IDEA 配置注释模板
categories:
  - 开发工具
  - IDEA
abbrlink: 4135e0a9
date: 2018-08-01 09:11:15
icons: [fas fa-fire red]
copyright_author: Jitwxs
---

## 一、类注释

打开 IDEA 的 `Settings`，点击 `Editor-->File and Code Templates`，点击右边 `File` 选项卡下面的 `Class`，在其中添加图中红框内的内容：

```java
/**
 * @author jitwxs
 * @date ${YEAR}年${MONTH}月${DAY}日 ${TIME}
 */
```

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201901/20190123201211263.png)

在我提供的示例模板中，说明了作者和时间，IDEA 支持的所有的模板参数在下方的 `Description` 中被列出来。

保存后，当你创建一个新的类的时候就会自动添加类注释。如果你想对接口也生效，同时配置上图中的 `Interface` 项即可。

## 二、方法注释

**不同于目前网络上互相复制粘贴的方法注释教程，本文将实现以下功能：**
 
- 根据形参数目自动生成 `@param` 注解
- 根据方法是否有返回值智能生成 `@Return` 注解

相较于类模板，为方法添加注释模板就较为复杂，首先在 `Settings` 中点击 `Editor-->Live Templates`。

点击最右边的 `+`，首先选择 `2. Template Group...` 来创建一个模板分组：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201901/20190123201855342.png)

在弹出的对话框中填写分组名，我这里叫做 userDefine：

![创建模板分组](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201901/20190123202104218.png)

然后选中刚刚创建的模板分组 `userDefine`，然后点击 `+`，选择 `1. Live Template`：

![创建模板](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201901/20190123202225989.png)

此时就会创建了一个空的模板，我们修改该模板的 `Abbreviation`、`Description` 和 `Template text`。需要注意的是，`Abbreviation` 必须为 `*`，最后检查下 `Expand with` 的值是否为 <kbd>Enter</kbd> 键。

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190729235756426.png)

上图中· `Template text` 内容如下，请直接复制进去，**需要注意首行没有 `/`，且 `*` 是顶格的**。

```java
*
 * 
 * @author jitwxs
 * @date $date$ $time$$param$ $return$
 */
```

注意到右下角的 `No applicable contexts yet` 了吗，这说明此时这个模板还没有指定应用的语言：

![No applicable contexts yet](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190730000025806.png)

点击 `Define`，在弹框中勾选`Java`，表示将该模板应用于所有的 Java 类型文件。

![设置 applicable contexts](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190730000437518.png)

还记得我们配置 `Template text` 时里面包含了类似于 `$date$` 这样的参数，此时 IDEA 还不认识这些参数是啥玩意，下面我们对这些参数进行方法映射，让 IDEA 能够明白这些参数的含义。点击 `Edit variables` 按钮：

![](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190730000545850.png)

为每一个参数设置相对应的 `Expression`：

![设置 Expression](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190730000854659.png)

需要注意的是，`date` 和 `time` 的 `Expression` 使用的是 IDEA 内置的函数，直接使用下拉框选择就可以了，而 `param` 这个参数 IDEA 默认的实现很差，因此我们需要手动实现，代码如下：

```groovy
groovyScript("def result = '';def params = \"${_1}\".replaceAll('[\\\\[|\\\\]|\\\\s]', '').split(',').toList(); for(i = 0; i < params.size(); i++) {if(params[i] != '')result+='* @param ' + params[i] + ((i < params.size() - 1) ? '\\r\\n ' : '')}; return result == '' ? null : '\\r\\n ' + result", methodParameters())
```

另外 `return` 这个参数我也自己实现了下，代码如下：

```groovy
groovyScript("return \"${_1}\" == 'void' ? null : '\\r\\n * @return ' + \"${_1}\"", methodReturnType())
```

>注：你还注意到我并没有勾选了 `Skip if defined` 属性，它的意思是如果在生成注释时候如果这一项被定义了，那么鼠标光标就会直接跳过它。我并不需要这个功能，因此有被勾选该属性。

点击 OK 保存设置，大功告成！

## 三、检验成果

### 3.1 类注释

类注释只有在**新建类时才会自动生成**，效果如下：

![类注释](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190730001256977.png)

### 3.2 方法注释

将演示以下几种情况：

1. 无形参
2. 单个形参
3. 多个形参
4. 无返回值
5. 有返回值

![方法注释](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/201907/20190730001836247.png)

## 四、Q & A

**（1）为什么模板的 `Abbreviation` 一定要叫 `*` ？`Expand with` 要保证是  <kbd>Enter</kbd>  键？**

答：因为 IDEA 模板的生成逻辑是 `模板名 + 生成键`，当生成键是 <kbd>Enter</kbd> 时，我们输入 `* + Enter` 就能够触发模板。

这也同时说明了为什么注释模板首行是一个 `*` 了，因为当我们先输入 `/*`，然后输入 `* + Enter`，触发模板，首行正好拼成了 `/**`，符合 Javadoc 的规范。

**（2）注释模板中为什么有一行空的 `*`？**

答：因为我习惯在这一行写方法说明，所以就预留了一行空的写，你也可以把它删掉。

**（3）注释模板中 `$time$$param$` 这两个明明不相干的东西为什么紧贴在一起？**

答：首先网上提供的大部分 param 生成函数在无参情况下仍然会生成一行空的 `@param`，因此我对param 函数的代码进行修改，使得在无参情况下不生成 `@param`，但是这就要求 `$param$` 要和别人处在同一行中，不然没法处理退格。

**（4）为什么 return 参数不使用 `methodReturnType()`， 而要自己实现？**

答：`methodReturnType()` 在无返回值的情况下会返回 void，这并没有什么意义，因此我对 methodReturnType() 返回值进行了处理，仅在有返回值时才生成。

**（5）为什么 `$return$` 不是单独一行？**

答：因为当 `methodReturnType()` 返回 null 时，无法处理退格问题，原因同第三点。
