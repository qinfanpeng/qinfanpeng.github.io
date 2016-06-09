---
layout: post
title:  "一次利用 byebug 和 RubyMine 刨根问底的过程"
date:   2016-5-19 12:55:19 +0800
categories: jekyll update
---

先看个现象：

```ruby
class User
  include Mongoid::Document
  has_many :orders
end
class Order
  include Mongoid::Document
  belongs_to :user
end

user.orders.pluck :id         # 正常返回id
user.orders.try :pluck, :id   # 返回 nil
```
奇怪的是第二行居然返回`nil`。`user.orders.try :pluck, :id`返回`nil`大致有以下三种可能：

1. `user.orders`返回`nil`，不过由于`user.orders.pluck :id`可以正常返回 id，排除这种可能；
2. `orders.pluck :id` 返回 `nil`，同上，也可排除这种可能；
3. `user.orders.try :pluck, :id`中的`user.orders`上没有`pluck`方法。这看起来是最不可能的情况，但是...

为了弄清原因，再来做个试验：

```ruby
user.orders.try! :pluck, :id  # NoMethodError: undefined method `pluck' for #<Array:0x0...>
```
这次我们用的是`try!`而非`try`，其结果正好符合上面的第3中情况。唯有这样假设方能解释得通：`user.orders.pluck :id`中的`user.orders`返回的是`Mongoid::Criteria`；而`user.orders.try :pluck, :id`中的`user.orders`返回的是`Array`。下面我们来借助`byebug`和`RubyMine`看看是否如此：

1. 光标hover至一处调用`pluck`的地方，Go to Declaration（⌘B），搜索找到`mongoid`中的定义`gems/mongoid-5.0.1/lib/mongoid/contextual/mongo.rb:399`，加上`byebug`:

```ruby
  def pluck(*fields)
    byebug
    normalized_select = fields.inject({}) do |hash, f|
      hash[klass.database_field_name(f)] = 1
      hash
    end

    view.projection(normalized_select).reduce([]) do |plucked, doc|
      values = normalized_select.keys.map do |n|
        n =~ /\./ ? doc[n.partition('.')[0]] : doc[n]
      end
      plucked << (values.size == 1 ? values.first : values)
    end
  end
```

2. 打开 Rails Console，运行`user.orders.pluck :id`，然后运行`backtrace`:
![Mongoid relation many](/images/mongoid_relation_many.png)
3. 定位到`gems/mongoid-5.0.1/lib/mongoid/relations/referenced/many.rb:414` :
 
```ruby
  def method_missing(name, *args, &block)
    byebug
    if target.respond_to?(name)
      target.send(name, *args, &block)
    else
      klass.send(:with_scope, criteria) do
        criteria.public_send(name, *args, &block)
      end
    end
  end
```

  再次借助`byebug`不难发现这里的`target`已经是`orders`数组了，其class为`Mongoid::Relations::Targets::Enumerable`。
由这里的代码得知：对于target支持的方法，直接在target上调用；target不支持的方法则在`criteria`调用。拿我们前面的例子来说：
`user.orders.pluck :id`中的target`orders`（Array）不支持`pluck`方法，所以其实是在`user.orders`返回的`criteria`上调用的`pluck`；而`user.orders.try :pluck, :id`，中的 target`user.orders`（Array）支持`try`方法，所以直接在target`user.orders`上调用了，又因`Array`上没有`pluck`，故而返回nil。

所以这样？

```ruby
user.orders.try :map, &:id
```

