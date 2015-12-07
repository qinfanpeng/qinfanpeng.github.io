### 判断Rails partial中变量是否定义的正确姿势

假设最初我们引入了一个Rails partial，内含`feature-one`:

```ruby
# app/views/shared/_feature_partial.erb
# ...
<section class='feature-one'>
 This is the feature one
</section>
# ...
```

然后，很多地方都用到了这个partial，类似于这样：

```ruby
<%= render 'shared/feature_partial' %>
# 或这样
<%= render partial: 'shared/feature_partial' %>
```

突然有一天来了个新需求：也需要用到 `feature_partial`，但是不在需要其中的`feature-one`了。于是我们决定对`feature_partial`做如下改动：

```ruby
# app/views/shared/_feature_partial.erb
# ...
<%- need_feature_one  = true unless defined? need_feature_one >
<%- if need_feature_one %>
   <section class='feature-one'>
     This is the feature one
   </section>
<%- end %>
# ...
```
首先，上面的`need_feature_one  = true unless defined? need_feature_one`不能写成这样`need_feature_one ||= true`或这样`need_feature_one = true unless need_feature_one`，这应该无需多言。 现在看起来完美了：对于`feature_partial`以前的引用无需做任何修改（修改最小化），功能便照样完好；对于新引用，若不需要`feature-one`，在引用的时候将`need_feature_one`设置为false即可，类似于下面这样：

```ruby
<%= render 'shared/feature_partial', need_feature_one: false %>
# 或这样
<%= render partial: 'shared/feature_partial', locals: {need_feature_one: false} %>
```

若故事到此结束，那本文就完全没存在的必要了。事实证明，对于`feature_partial`以前的引用，`need_feature_one  = true unless defined? need_feature_one`返回的永远都是`nil`，意外吧？要了解其中原委，还得从两方面说起：

通常而言， Ruby中以问号结尾的方法都返回true/false，如(respond_to?、start_with?等)， 然而`defined?`这货却并非如此，例证如下：

```ruby
>> a = 1
 => 1
>> defined? a
 => "local-variable"
>> defined? b
 => nil
>> defined? nil
 => "nil"
>> defined? String
 => "constant"
>> defined? 1
 => "expression"
>> @c = 2 
>> defined? @c
 => "instance-variable"
```
看起来，`defined?`这货返回的是对应变量的类型，对于没定义的变量返回的是 nil。但这还是解释不了为啥`need_feature_one  = true unless defined? need_feature_one`总返回`nil`阿。

若没提前定义`need_feature_one`，`need_feature_one  = true unless defined? need_feature_one`永远返回`nil`，不妨假设：在进行`defined? need_feature_one`判断时，Ruby已经为这种一行式语法定义了名为`need_feature_one`的变量，默认值为`nil`；所以在进行`defined? need_feature_one`判断时因`need_feature_one`已经定义，所以`need_feature_one  = true`一直没得到执行。下面来窥探一下其编译版本，验证一下我们的想法：

```ruby
puts RubyVM::InstructionSequence.compile("need_feature_one = true unless defined? need_feature_one").disassemble
== disasm: <RubyVM::InstructionSequence:<compiled>@<compiled>>==========
local table (size: 2, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 2] need_feature_one
0000 trace            1                                               (   1)
0002 putnil                            
0003 putobject        "local-variable" # defined? need_feature_one 返回 "local-variable"，进一步说明此时 need_feature_one已定义
0005 swap
0006 pop
0007 branchunless     12               # 若defined?条件为false，则跳转至第12行
0009 putnil
0010 leave
0011 pop
0012 putobject        true             # 第12行在此
0014 dup
0015 setlocal_OP__WC__0 2
0017 leave
=> nil
```
抛开其他细节不谈，这里出现了“local-variable”，由前面对`defined?`的特性的了解可知，这是已定义本地变量的输出，说明在进行`unless defined? need_feature_one`判断时`need_feature_one`已定义，因而可以佐证我们的假设。关于Ruby编译、解析相关知识详见：[Ruby 是如何解释运行程序的](https://ruby-china.org/topics/28040)，欲知上面每条指令（YARV instruction）的意思，请期待《Ruby原理剖析》。

下面再来看看其词法解析版本：

```ruby
require 'ripper'
require 'pp'
    
pp Ripper.sexp("need_feature_one = true unless defined? need_feature_one")

[:program,
 # unless修饰符
 [[:unless_mod,
   # defined? 条件判断，注意这里的:var_ref，可认为是变量引用，大致可说明此时need_feature_one已经定义。
   [:defined, [:var_ref, [:@ident, "need_feature_one", [1, 40]]]],
   [:assign,
    [:var_field, [:@ident, "need_feature_one", [1, 0]]],
    # 将need_feature_one赋值为true。
    [:var_ref, [:@kw, "true", [1, 19]]]]]]]
=> [:program, [[:unless_mod, [:defined, [:var_ref, [:@ident, "need_feature_one", [1, 40]]]], [:assign, [:var_field, [:@ident, "need_feature_one", [1, 0]]], [:var_ref, [:@kw, "true", [1, 19]]]]]]]
```

留意上面第一个:var_ref，下面还会用作对比。

事实证明，下面这样是可以修复上面这个问题：

```ruby
<%- unless defined? need_feature_one %>
 need_feature_one = true
<%- end %>
```
对应编译版本如下：

```ruby
puts RubyVM::InstructionSequence.compile("unless defined? need_feature_one \n need_feature_one = true \n end").disassemble
== disasm: <RubyVM::InstructionSequence:<compiled>@<compiled>>==========
== catch table
| catch type: rescue st: 0003 ed: 0008 sp: 0000 cont: 0010
== disasm: <RubyVM::InstructionSequence:defined guard in <compiled>@<compiled>>
0000 putnil
0001 leave
|------------------------------------------------------------------------
local table (size: 2, argc: 0 [opts: 0, rest: -1, post: 0, block: -1, kw: -1@-1, kwrest: -1])
[ 2] need_feature_one
0000 trace            1                                               (   1)
0002 putnil
0003 putself
0004 defined          17, :need_feature_one, true
0008 swap
0009 pop
0010 branchunless     15
0012 putnil
0013 leave
0014 pop
0015 trace            1                                               (   2)
0017 putobject        true
0019 dup
0020 setlocal_OP__WC__0 2
0022 leave
=> nil
```
对比发现，这个编译版本不再有先前那样的“local-variable”了，说明在进行`unless defined? need_feature_one`判断时，`need_feature_one`尚未定义。

对应词法解析版本如下：

```ruby
require 'ripper'
require 'pp'

pp Ripper.sexp("unless defined? need_feature_one \n need_feature_one = true \n end")

[:program,
 [[:unless,
   # defined?条件判断，注意此处是:vcall，而非先前的:var_ref，vcall代表Ruby还未识别出这里调用的need_feature_one是一个变量还是一个方法；而var_ref表明此时已经识别出是一个变量。大致可说明此时need_feature_one尚未定义。
   [:defined, [:vcall, [:@ident, "need_feature_one", [1, 16]]]],
   [[:assign,
     [:var_field, [:@ident, "need_feature_one", [2, 1]]],
     [:var_ref, [:@kw, "true", [2, 20]]]]],
   nil]]]
=> [:program, [[:unless, [:defined, [:vcall, [:@ident, "need_feature_one", [1, 16]]]], [[:assign, [:var_field, [:@ident, "need_feature_one", [2, 1]]], [:var_ref, [:@kw, "true", [2, 20]]]]], nil]]]
```

但Ruby/Rails如此优雅的存在，肯定有更优雅的解决方案。比如下面这样：

```ruby
# 还行
need_feature_one = true unless local_assigns.has_key? :need_feature_one

# 好
need_feature_one = local_assigns.fetch(:need_feature_one, true)
# 或
need_feature_one = local_assigns.fetch(:need_feature_one) { true }

# 更好
# ... 等你来发现。
```

关于`fetch`两种用法的区别，可查阅[《Confident Ruby》](http://book.douban.com/subject/25751838/)。

#### 小结
看起来，带条件修饰符的变量赋值语句， Ruby都会为其先赋值为nil，然后进行条件判断，若条件满足，再进行第二次赋值操作。

#### 参考资料
- [《Ruby Under a Microscope》](http://book.douban.com/subject/24718740/)
- [YARV: Instruction Table](http://www.atdot.net/yarv/insnstbl.html)
- [How to disassemble Ruby code into RubyVM opcodes/Instruction Sequences](http://rehanjaffer.com/how-to-disassemble-ruby-code-into-rubyvm-yarv-opcodes-instruction-sequences/)
- http://mla.n-z.jp/~w3ml/w3ml.cgi/ruby-changes/msg/20450







