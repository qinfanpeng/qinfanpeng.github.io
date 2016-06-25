---
layout: post
title:  "探秘模块混入（include Module）背后的故事"
date:   2016-06-24 12:55:19 +0800
categories: jekyll update
---

先上段代码：

```ruby
class Person
  def wear
    '穿衣'
  end
end

class Economist < Person
end

economist = Economist.new
economist.wear # => ? ①

module Professor
  def wear
    "戴眼镜"
  end
end

class Economist < Person
  include Professor
end

economist.wear # => ？②

module Professor
  def wear
    "#{super}，戴眼镜"
  end
end

economist.wear # => ？③

module Expert
  def refute_rumor
    '辟谣'
  end
end

module Professor
  include Expert
end

economist.refute_rumor # => ？④

Economist.include Professor
economist.refute_rumor # => ？⑤

```

相信对于上面四处问号，我们都有自己的答案了。毫无疑问①处会返回“穿衣”，这里我们都不会错；②是返回“穿衣”还是“戴眼镜”呢？估计要是没有③处的对比的话，估计有人会回答错误，②的正确答案是“戴眼镜”；那么③的正确答案自然是“穿衣，戴眼镜”了；④处会返回“辟谣”吗？，并不会，这里会报错：`No Method Error`；⑤会正确返回“辟谣”

不过前面答对与否，我们最好理解背后的原理，尤其是④处的情况。①处最好理解，调用了继承自`Person`的`wear`方法而已，继承体系大致如下：
![class_extend](/images/class_extend.png)

下面来看②③两处处，由前面的答案可知②调用的是来自`Professor`module中的`wear`方法，而非`Person`Class中的。结合②③两处，可大致得出如下继承体系：
![module_include](/images/module_include.png)

等等，如果是这样的话，那么`Professor`被另一个class include的时候，该怎么办呢？总不能说一个module有多个super吧。记得以前看《Ruby元编程》的时候，里面有很多类似的图。为了节约时间就不去翻书了，直接去Google Image里搜索关键字:`ruby include`，不难发现这张图：

![include_module_copy](/images/include_module_copy.png)

“被include的实际是module的副本，而非module本身”， 如此一来module 被include无论多少次都无所谓了，因为更改的是那些对应副本的`super`。所以前面的图应该改成这样才对：
![module_include2](/images/module_include2.png)

再看下③处，这里我们是在`Professor`被include后（实际上被 include的是的`Professor`的副本），用类似打开类的方式修改的`Professor`，并没有修改其副本。这里正确返回“穿衣，戴眼镜”，能说明`Professor`和它的副本共享了同一份方法实现，即是说拷贝的时候并没拷贝底层的方法，类似下面这样：
![module_and_copy_share_methods](/images/module_and_copy_share_methods.png)

与③不同是④是通过在`Professor`include另外一个module`Expert`来修改的，而这个过程又发生在`Professor`被 include之后。这种情况下`Economist`感知不到`Professor`的改变。究其深层原因，结合前面的经验，不难得出以下结果：
![change_module_after_inclued](/images/change_module_after_inclued.png)

我们索性一鼓作气画一下⑤处的情况：
![include_module_which_include_another_module](/images/include_module_which_include_another_module.png)

#### 小结
1. 任何时候include module都include的是对应的副本，而非module本身。这是由于include会改变继承体系，如果直接include module本身的话，就使得该module难以再被其他地方include了，而每次都include module的副本则不会有这样的问题。
2. module和它被include的副本共享同一份方法实现，因而直接修改方法，会反映到已经include的地方去。
3. 通include另一个module B的方式修改一个已经被include了的 module A，不会体现到include A 的地方，因为B并没有被include A的副本中去。

#### 参考资料

1. [《Ruby Under a Microscope》](https://book.douban.com/subject/24718740/)（对应中文版《Ruby原理剖析》由张汉东先生翻译，应该不久就可以和大家见面了。如果说《Ruby元编程》让我们了解很多Ruby高级用法的话，那么《Ruby原理剖析》则在更深层次上系统地阐述了这些高级用法背后的原理）
2. [《Ruby元编程（第2版）》](https://book.douban.com/subject/26575429/)
