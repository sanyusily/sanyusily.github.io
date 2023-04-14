---
layout: article
title: '起名的艺术'
---


> 计算机科学中仅存在两件难事：缓存失效和事物命名。<br>
&mdash; <cite>Phil Karlton</cite>

-----

**定理 1：代码应该是写给人来读的，只不过顺便能在机器执行而已。**

代码大部分时候是用来维护的，而不是用来实现功能的，这个准则适用于大部分的软件工程。比如我所知道的一个软件系统，开发了三个月即上线使用，而用于维护的时间却是以年为单位的，开发者花大量的时间用于调整代码以确保其正确运行。

因此，编写可读代码成了用于衡量代码质量的重要标准之一（另外一个熟知的标准是正确性）。

![代码质量衡量标准]({{site.img_url}}/2017-code-quality.jpg){:.center}

-----

**定理 2：好的代码胜过好的注释。**

好的代码自己本身就是最好的文档。当你打算加注释的时候，问问自己“我如何才能把我的代码改善到不需增加注释？”重构自己的代码，然后使文档让其更清楚。

注释的恰当用法是弥补我们在用代码表达意图时遭遇的失败。比如解释为什么要这么做，或者是对一个复杂晦涩的代码段做必要的阐释。

下面这种毫无意义的注释，只会增加你代码文件的字节数：

```c
num += 1;           // 用户人数加 1
```

正确的做法应该是把变量起一个具有准确含义的名字：

```c
userCount += 1;
```


再如，一个受过正规训练的程序员绝不会同意把一个函数名起成 `killBill`，职业直觉告诉他们应该去写一个 `killPeople` 的函数，然后把 `Bill` 当作这个函数的参数。

如此看来，给变量（函数）起名也成了一种艺术。

-----

起名的首要原则是把信息装到名字中。代码中的名字，便是你和其他读代码的人之间的桥梁，准确的名字，才能传递准确的信息。

**避免使用抽象的名字。**例如不要使用 `tmp` 或 `getData` 这样空洞抽象的名字。

**可以在名字中增加额外的信息。**例如 `string hex_id`、`size_kb` 等。在给布尔类型命名时，加上 `is`、`has`、`can`、`should` 让含义更清晰，比如 `bool use_ssl = true` 好于 `bool disable_ssl = false`。

**有目的地使用大小写、下划线进行命名。**例如用 `CamelCase` 表示类名，用 `lower_separated` 表示变量名，用 `CONSTANT_NAME` 表示常量名。在使用 jQuery 的代码中，一个非常有用的规范是，给 jQuery 返回的结果加上 `$` 作为前缀：

```js
var $all_images = $('img');   // $all_images is a jQuery object
var height = 250;             // height is not
```

**为作用域大的名字采用更长的名字。**同理，可以在小的作用域里使用短的名字。

**使用不会被误解的名字。**例如用 `first` 和 `last` 来表示包含的范围（闭区间），用 `begin` 和 `end` 来表示排除的范围（开区间）：

```java
print integer_range(start=2, stop=4)
# Does this print [2,3] or [2,3,4] (or something else)?

set.PrintKeys(first="Bart", last="Maggie")
```

**使用英文来命名。**有一些国内的程序员为了方便，喜欢用拼音甚至缩写来命名，这应该被严令禁止。