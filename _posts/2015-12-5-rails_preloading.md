---
layout: post
title:  "【翻译】Rails（3&4）预加载（preload）的3种方式"
date:   2015-12-28 12:55:19 +0800
categories: jekyll update
---

>　　作者：[Robert Pankowecki](https://twitter.com/arkency)
>原文地址：[http://blog.arkency.com/2013/12/rails4-preloading/](http://blog.arkency.com/2013/12/rails4-preloading/)

> 本文是2013年的一篇老文章，资深人士请绕道。前些天自己遇到了些麻烦，详见：[https://ruby-china.org/topics/28225](https://ruby-china.org/topics/28225) ，本来当初也看到过这篇文章，只因自己粗心，而没及时深读本文（毕竟对我而言英文没有中文那么直观），否则将节约我不少时间。翻译此文，希望能节约他人的几秒钟。

![#includes_#preload_#eager_load](http://blog.arkency.com/assets/images/preloading/header-fit.png)

　　和Rails的ActiveRecord打交道，你大概已经习惯用`#includes`来预加载数据。但你知道为啥它生成的SQL有时小而美，有时却是个巨型查询，且每个表、列都被重命名过（"users"."id" AS t0_r0，"addresses"."id" AS t1_r0）？你知道`##preload`和`#eager_load` 也能帮你做这事儿吗？你知道Rails4中预加载方面有哪些变化吗？如果都不知道的话，请耐心读下去。本文不会太长，将助你了解预加载中哪些你不熟悉的方面。

　　首先让我们从Active Record关联关系（associations）定义开始，本文通篇将以此例为基础。

```ruby 
    class User < ActiveRecord::Base
      has_many :addresses
    end
    
    class Address < ActiveRecord::Base
      belongs_to :user
    end
``` 

　　下面是些实验数据，以便我们验证查询结果：

```ruby
    rob = User.create!(name: "Robert Pankowecki", email: "robert@example.org")
    bob = User.create!(name: "Bob Doe", email: "bob@example.org")

    rob.addresses.create!(country: "Poland", city: "Wrocław", postal_code: "55-555", street: "Rynek")
    rob.addresses.create!(country: "France", city: "Paris", postal_code: "75008", street: "8 rue Chambiges")
    bob.addresses.create!(country: "Germany", city: "Berlin", postal_code: "10551", street: "Tiergarten")
``` 

#### Rails 3
　　预加载数据时，通常你都会选择`#includes`，大概由于从Rails 2甚至Rails 1开始，就推荐使用该方法了。它会魔法般地生成两个查询：

```ruby
    User.includes(:addresses)
    #  SELECT "users".* FROM "users" 
    #  SELECT "addresses".* FROM "addresses" WHERE "addresses"."user_id" IN (1, 2)
``` 

　　另外两个方法呢？我们先来看它们的实际表现。

```ruby
    User.preload(:addresses)
    #  SELECT "users".* FROM "users" 
    #  SELECT "addresses".* FROM "addresses" WHERE "addresses"."user_id" IN (1, 2)
```

　　到底是`#preload`类似于`#includes`呢？还是`#includes`类似于`#preload`？请继续阅读。

`#eager_load`行为如下：

```ruby
    User.eager_load(:addresses)
``` 
```sql
    #  SELECT
    #  "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4, 
    #  "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7 
    #  FROM "users" 
    #  LEFT OUTER JOIN "addresses" ON "addresses"."user_id" = "users"."id"
```

　　结果完全不一样，对吧。问题的关键在于Rails有两种方式进行数据预加载：一种通过单独的（separate）查询去加载其他表的数据；另一种只用单个查询(带`left join`)一次性加载所有数据。

　　用`#preload`，就意味着你希望总是用单独的（separate）查询；用`#eager_load`，表明你用的是单个查询（一次性加载所有数据）。那么`#includes`呢？它会帮你选择从上面选择一种方式，即你把决定权交给了Rails。那么它是根据什么决定的呢？你可能会问，是基于查询条件(query conditions)。我们先来看这样一个例子：`#includes`把查询代理给了`#eager_load`，所以只会有一个巨型查询产生。

```ruby
   # 这两个用法是等效的
    User.includes(:addresses).where("addresses.country = ?", "Poland")
    User.eager_load(:addresses).where("addresses.country = ?", "Poland")
```
```sql
    # SELECT 
    # "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4,
    # "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7 
    # FROM "users"
    # LEFT OUTER JOIN "addresses" 
    # ON "addresses"."user_id" = "users"."id" 
    # WHERE (addresses.country = 'Poland')
```
    
　　本例中Rails发现`where`语句用到了预加载表（addresses）中的列（country），故而将`#includes`代理给了`#eager_load`。你也可直接调用`#eager_load`,来到稳定实现这个效果。

　　如果显示调用`#preload`会怎样呢？

```ruby
    User.preload(:addresses).where("addresses.country = ?", "Poland")
    #  SELECT "users".* FROM "users" WHERE (addresses.country = 'Poland')
    #
    #  SQLite3::SQLException: no such column: addresses.country
```
    
　　由于没将`users`表和`addresses`表关联（join）起来，故而抛出了异常。

#### 目标实现了吗

　　如果你再看下这个例子

```ruby
    User.includes(:addresses).where("addresses.country = ?", "Poland")
```
    
　　你可能会问：这些代码初衷是啥？作者这样写想表达啥？我们将用简洁的Rails代码来达成以下目标：

　　1. 找出带波兰地址的用户，且**只**预加载其波兰地址；
　　2. 找出带波兰地址的用户，但预加载其**所有**地址；
　　3. 找出所有的用户，但**只**预加载其波兰地址。

　　你知道我们已实现了其中哪一个目标吗？第一个。下面来看看能否实现后面两个。

#### `#preload`有啥优势

> 当前任务：找出带波兰地址的用户，但预加载其所有地址。即是需要找出那些至少有一个波兰地址的用户，及其所有地址。

　　我们只需要那些有波兰地址的用户，这很简单：`User.joins(:addresses).where("addresses.country = ?", "Poland")`；要预加载地址，再加上`includes(:addresses)`就够了嘛，对吧？

```ruby
    r = User.joins(:addresses).where("addresses.country = ?", "Poland").includes(:addresses)

    r[0]
    #=> #<User id: 1, name: "Robert Pankowecki", email: "robert@example.org", created_at: "2013-12-08 11:26:24", updated_at: "2013-12-08 11:26:24"> 

    r[0].addresses
    # [
    #   #<Address id: 1, user_id: 1, country: "Poland", street: "Rynek", postal_code: "55-555", city: "Wrocław", created_at: "2013-12-08 11:26:50", updated_at: "2013-12-08 11:26:50">
    # ]
```

　　然而，事实并非我们所想的那样：结果中少了了第二个地址，此处它本该出现的。Rails再次探测到 `Where`语句使用了预加载了的表(addresses)，因而底层还是用`#eager_load`来实现`#includes`的。唯一的区别是这里使用的是`INNER JOIN`而非前面的`LEFT JOIN`，但查询结果并非没啥区别。

```sql
    SELECT 
    "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4,
    "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7 
    FROM "users"
    INNER JOIN "addresses" 
    ON "addresses"."user_id" = "users"."id" 
    WHERE (addresses.country = 'Poland')
```

　　难得有机会比Rails还“聪明”，我们可以显示地调用`#preload`而非`#includes`来表明我们的意图：

```ruby
    r = User.joins(:addresses).where("addresses.country = ?", "Poland").preload(:addresses)
```
```sql
    # SELECT "users".* FROM "users"
    # INNER JOIN "addresses" ON "addresses"."user_id" = "users"."id" 
    # WHERE (addresses.country = 'Poland')

    # SELECT "addresses".* FROM "addresses" WHERE "addresses"."user_id" IN (1)

    r[0] 
    # [#<User id: 1, name: "Robert Pankowecki", email: "robert@example.org", created_at: "2013-12-08 11:26:24", updated_at: "2013-12-08 11:26:24">] 

    r[0].addresses
    # [
    #  <Address id: 1, user_id: 1, country: "Poland", street: "Rynek", postal_code: "55-555", city: "Wrocław", created_at: "2013-12-08 11:26:50", updated_at: "2013-12-08 11:26:50">,
    #  <Address id: 3, user_id: 1, country: "France", street: "8 rue Chambiges", postal_code: "75008", city: "Paris", created_at: "2013-12-08 11:36:30", updated_at: "2013-12-08 11:36:30">] 
    # ]
```

　　这正是我们想要的。借助`#preload`，我们不必再将user查询条件和数据预加载混淆在一起了，查询再次变得自然而简洁。

#### 预加载关联数据子集(Preloading subset of association)
> 当前任务：查找所有所有用户，但只预加载其波兰地址。

　
　　说实话，我不喜欢仅预加载关联数据子集，因为程序其他部分还是可能会误认为关联数据已经全加载了（基于该假设做出的决策，自然也是错的）。若获取关联数据子集用于展示，倒还说得过去。

　　我更倾向于直接在关联上加条件：

```ruby
    class User < ActiveRecord::Base
      has_many :addresses
      has_many :polish_addresses, conditions: {country: "Poland"}, class_name: "Address"
    end
```

　　接着，我们便可显示预加载关联数据了：

```ruby
    r = User.preload(:polish_addresses)

    # SELECT "users".* FROM "users" 
    # SELECT "addresses".* FROM "addresses" WHERE "addresses"."country" = 'Poland' AND "addresses"."user_id" IN (1, 2)

    r

    # [
    #   <User id: 1, name: "Robert Pankowecki", email: "robert@example.org", created_at: "2013-12-08 11:26:24", updated_at: "2013-12-08 11:26:24">
    #   <User id: 2, name: "Bob Doe", email: "bob@example.org", created_at: "2013-12-08 11:26:25", updated_at: "2013-12-08 11:26:25">
    # ] 

    r[0].polish_addresses

    # [
    #   #<Address id: 1, user_id: 1, country: "Poland", street: "Rynek", postal_code: "55-555", city: "Wrocław", created_at: "2013-12-08 11:26:50", updated_at: "2013-12-08 11:26:50">
    # ] 

    r[1].polish_addresses
    # []
```

　　或者：

```ruby
    r = User.eager_load(:polish_addresses)
```
```sql
    # SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4, 
    #        "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7
    # FROM "users" 
    # LEFT OUTER JOIN "addresses" 
    # ON "addresses"."user_id" = "users"."id" AND "addresses"."country" = 'Poland'
```
```ruby
    r
    # [
    #   #<User id: 1, name: "Robert Pankowecki", email: "robert@example.org", created_at: "2013-12-08 11:26:24", updated_at: "2013-12-08 11:26:24">,
    #   #<User id: 2, name: "Bob Doe", email: "bob@example.org", created_at: "2013-12-08 11:26:25", updated_at: "2013-12-08 11:26:25">
    # ]

    r[0].polish_addresses
    # [
    #   #<Address id: 1, user_id: 1, country: "Poland", street: "Rynek", postal_code: "55-555", city: "Wrocław", created_at: "2013-12-08 11:26:50", updated_at: "2013-12-08 11:26:50">
    # ]

    r[1].polish_addresses
    # []
```

　　换做是我们，只在运行时才知道要应用的关联条件，我们该怎么做？实话说我不知道，如果你知道，请在评论中告知。

#### 最后一个目标

　　你可能会问：区区一个预加载为啥就这么麻烦？我也不清楚，不过我想大部分`ORM`初衷都是生成单条查询、从单张表中加载数据。有了预加载，情况就复杂多了：它致使我们用**多条件**的查询从**多张表**中去加载**多类数据**。Rails中，我们采用链式API来构建多个查询(比如`#preload`)。

　　若问我想要什么样的API？类似下面这样的：

```ruby
    User.joins(:addresses).where("addresses.country = ?", "Poland").preload do |users|
      users.preload(:addresses).where("addresses.country = ?", "Germany")
      users.preload(:lists) do |lists|
        lists.preload(:tasks).where("tasks.state = ?", "unfinished")
      end
    end
```

　　我希望你能明白我的心思，但这仅是理想，还是让我们回到现实中来吧。

#### Rails 4 中的变化

　　现在我们来说说Rails 4中有关预加载的变化。

```ruby
    class User < ActiveRecord::Base
      has_many :addresses
      has_many :polish_addresses, -> {where(country: "Poland")}, class_name: "Address"
    end
```

　　现在Rails推荐你使用新的`lambda`语法来定义关联条件。这个改进相当棒，因为我不止一次看到这样的错误：关联条件相关代码只在类加载时被解释执行了一次(关联中含有Date.today的代码就会出错)。

　　Rails同样推荐你使用`lambda`语法或`method`语法去定义scope。

```ruby
    # 差，Time.now 会一直停留在class加载的时刻（相对于成了一个“死”值了，而非动态变化的）
    # 并且在开发环境中，这个bug还不容易发现，因为一旦类有修改，开发环境便会自动重新加载代码（掩盖了“死”值的真相）
    scope :from_the_past, where("happens_at <= ?", Time.now)

    # 好
    scope :from_the_past, -> { where("happens_at <= ?", Time.now) }

    # 好
    def self.from_the_past
      where("happens_at <= ?", Time.now)
    end
```

　　无论是动态解释，还是只在一开始解释一次，我们的条件`where(country: "Poland")`都是一样的。但Rails尝试统一两种场景（关联、scope）的语法总是好的，从而避免我们再犯类似的错误。

　　说完了语法上的变化，下面来看看行为上的改变。

```ruby
    User.includes(:addresses)
    #  SELECT"users".* FROM "users"
    #  SELECT "addresses".* FROM "addresses" WHERE "addresses"."user_id" IN (1, 2)

    User.preload(:addresses)
    #  SELECT "users".* FROM "users"
    #  SELECT "addresses".* FROM "addresses" WHERE "addresses"."user_id" IN (1, 2)

    User.eager_load(:addresses)
```
```sql
    #  SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4,
    #         "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7
    #  FROM "users"
    #  LEFT OUTER JOIN "addresses"
    #  ON "addresses"."user_id" = "users"."id"
```
    
　　好吧，看起来还是一样，这倒不奇怪。试着加上先前让我们麻烦不断的查询条件看看：

```ruby
    User.includes(:addresses).where("addresses.country = ?", "Poland")
   # 
    #弃用警告: 貌似你预加载了SQL字符串代码片段中引用的（referenced）表（users或addresses），例如：
    #
    #    Post.includes(:comments).where("comments.title = 'foo'")
    #
    # 目前，Active Record识别出来了字符串中的表（comments），并且知道将其JOIN到整个大查询中，而非单独弄一条查询出来。然而，没有成熟的SQL解析器支持的情况下，这样做是有天然缺陷的。显然我们不会实现SQL解析器，所以将移除此特性。从今以后，引用（reference）字符串中的表时，你得显示告诉Active Record：
    #
    #   Post.includes(:comments).where("comments.title = 'foo'").references(:comments)
    # 
    # 如果你用不着这个隐式join引用表的特性，你可以通过如下设置来完全禁用它：`config.active_record.disable_implicit_join_references = true`.
  #
    # SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4,
    #        "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7
    # FROM "users" 
    # LEFT OUTER JOIN "addresses" ON "addresses"."user_id" = "users"."id" 
    # WHERE (addresses.country = 'Poland')
```

　　咦，看起来真啰嗦！不过建议你还是仔细读一下，因为它把其中情理解释得很清楚。

　　换句话说，Rails不愿像以前那样聪明了——即不再会通过探测`where`条件来决定用哪一种查询策略了，它需要我们的帮助。我们得告诉它某张表有条件查询，就像这样：

```ruby
  User.includes(:addresses).where("addresses.country = ?", "Poland").references(:addresses)
```

　　我很好奇如果我们尝试预加载多张表，但只显示引用（reference）了其中一张表会怎样？

```ruby
    User.includes(:addresses, :places).where("addresses.country = ?", "Poland").references(:addresses)
```
```sql
    #  SELECT "users"."id" AS t0_r0, "users"."name" AS t0_r1, "users"."email" AS t0_r2, "users"."created_at" AS t0_r3, "users"."updated_at" AS t0_r4, 
    #         "addresses"."id" AS t1_r0, "addresses"."user_id" AS t1_r1, "addresses"."country" AS t1_r2, "addresses"."street" AS t1_r3, "addresses"."postal_code" AS t1_r4, "addresses"."city" AS t1_r5, "addresses"."created_at" AS t1_r6, "addresses"."updated_at" AS t1_r7,
    #         "places"."id" AS t2_r0, "places"."user_id" AS t2_r1, "places"."name" AS t2_r2, "places"."created_at" AS t2_r3, "places"."updated_at" AS t2_r4 
    #  FROM "users" 
    #  LEFT OUTER JOIN "addresses" ON "addresses"."user_id" = "users"."id" 
    #  LEFT OUTER JOIN "places" ON "places"."user_id" = "users"."id" 
    #  WHERE (addresses.country = 'Poland')
```

　　我们本以为它会用`#eager_load`策略（通过`LEFT JOIN`）去加载`addresses`，用`#preload`策略（单独的查询）去加载`places`。但是你也看到了，事实并非如此。可能以后这个行为会改变。

　　如果显示使用`#eager_load`去预加载数据，Rails4是不会警告你加`#references`的，生成的查询也一样。

```ruby
   User.eager_load(:addresses).where("addresses.country = ?", "Poland")
```

　　换言之，下面两种写法是等效的：

```ruby
   User.includes(:addresses).where("addresses.country = ?", "Poland").references(:addresses)
   User.eager_load(:addresses).where("addresses.country = ?", "Poland")
```

　　对于`#preload`，行为还是一样（和Rails4以前一样）。

```ruby
    User.preload(:addresses).where("addresses.country = ?", "Poland")
    #  SELECT "users".* FROM "users" WHERE (addresses.country = 'Poland')
    #
    #  SQLite3::SQLException: no such column: addresses.country: SELECT "users".* FROM "users"  WHERE (addresses.country = 'Poland')
```

　　前面提到的其他查询，在Rails4中表现也是一样的：

```ruby
    # 找出带波兰地址的用户，但预加载其所有地址；
    User.joins(:addresses).where("addresses.country = ?", "Poland").preload(:addresses)
    
    #找出所有的用户，但只预加载其波兰地址；
    User.preload(:polish_addresses)
```

　　下面这些方法的文档，在Rails3中已缺失多年了，终于在Rails4补上了：

  - [#includes](http://api.rubyonrails.org/v4.0.1/classes/ActiveRecord/QueryMethods.html#method-i-includes)
  - [#preload](http://api.rubyonrails.org/v4.0.1/classes/ActiveRecord/QueryMethods.html#method-i-preload)
  - [#eager_load](http://api.rubyonrails.org/v4.0.1/classes/ActiveRecord/QueryMethods.html#method-i-eager_load)

#### 小结

　　Rails有3种数据预加载方式：

　　1. #includes
　　2. #preload
　　3. #eager_load

　　`#includes`会根据是否有预加载表相关的查询条件，来决定代理给`#preload`还是`#eager_load`。

　　`#preload`总是会用单独的数据库查询来预加载数据。

　　`#eager_load`会用`LEFT JOIN`联结每个预加载表，从而形成单个巨型查询。

　　Rails4中，如果有预加载表相关查询条件，那么你应该结合使`#references`来使用`#includes`。

#### 更多

　　喜欢本文吗？你也可能对我们的[Rails书籍](http://blog.arkency.com/products)感兴趣：

- [Fearless Refactoring Rails Controllers](http://rails-refactoring.com/?_ga=1.173005729.34765377.1448604207)
- [Rails meets React.js](http://blog.arkency.com/rails-react)
- [Developers Oriented Project Managment](http://blog.arkency.com/developers-oriented-project-management/)
- [Bloging for Busy Programmers](http://blog.arkency.com/blogging/)
- [React.js by Example](http://reactkungfu.com/react-by-example/?_ga=1.247455494.34765377.1448604207)
- [Responsible Rails](http://blog.arkency.com/responsible-rails)
