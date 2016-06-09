### Inverse_of 的坑

Rails 4.1以后，大多数情况下它都能自动帮我们加上`inverse_of`。大概也是由于这个原因，现在我们少有提到`inverse_of`了，但是某些情况下Rails不会自动帮我们加的，需要我们自己留意：

```ruby
# activerecord-4.2.5.1/lib/active_record/reflection.rb:553
VALID_AUTOMATIC_INVERSE_MACROS = [:has_many, :has_one, :belongs_to]
INVALID_AUTOMATIC_INVERSE_OPTIONS = [:conditions, :through, :polymorphic, :foreign_key]

def can_find_inverse_of_automatically?(reflection)
  reflection.options[:inverse_of] != false &&
    VALID_AUTOMATIC_INVERSE_MACROS.include?(reflection.macro) &&
    !INVALID_AUTOMATIC_INVERSE_OPTIONS.any? { |opt| reflection.options[opt] } &&
    !reflection.scope
end
```

下面先来温习一下`inverse_of`相关作用。
#### Memory Optimization Associated Object

```ruby
class Dungeon < ActiveRecord::Base
  has_many :traps
end

class Trap < ActiveRecord::Base
  belongs_to :dungeon, foreign_key: :my_dungeon_id
end    
```
这里之所以要加上`foreign_key: :my_dungeon_id`是为了阻止 `Rails`自动帮我们加上`inverse_of`。

```ruby
dungeon = Dungeon.first
trap = dungeon.traps.first

trap.dungeon # 此处还会去查询数据库（有点不合理，对吧）

trap.dungeon == dungeon     # => true
trap.dungeon.equal? dungeon # => false（说明它俩虽然内容相同，但在内存中却不是同一个对象）

dungeon.level == trap.dungeon.level # => true
dungeon.level = 10
dungeon.level == trap.dungeon.level # => false（也有点不合理，对吧）
```

下面我们移除`foreign_key: :my_dungeon_id`，如此一来 `Rails`便会自动帮我们加上`inverse_of`的：

```ruby
class Trap < ActiveRecord::Base
  belongs_to :dungeon
end    

dungeon = Dungeon.first
trap = dungeon.traps.first

trap.dungeon # 此处不会再去查询数据库了

trap.dungeon == dungeon     # => true
trap.dungeon.equal? dungeon # => true（这才合情理嘛）

dungeon.level == trap.dungeon.level # => true
dungeon.level = 10
dungeon.level == trap.dungeon.level # => true
```

#### Creating associated objects across a has_many :through

```ruby
class Post < ActiveRecord::Base 
  has_many :taggings 
  has_many :tags, :through => :taggings 
end

class Tag < ActiveRecord::Base 
  has_many :taggings 
  has_many :posts, :through => :taggings 
end

class Tagging < ActiveRecord::Base 
  belongs_to :tag, foreign_key: :my_tag_id
  belongs_to :post
end
```

同样加`foreign_key: :my_tag_id`是为了避免Rails自动帮我们加上`reverse_of`。

```ruby
post = Post.first
tag = post.tags.build name: "ruby"
tag.save

tag.taggings # => []
tag.posts    # => []
```
造成最后两行返回空的本质原因是没保存对应的关联的数据`tagging`。现在如果移除`foreign_key: :my_tag_id`，就相当于加上了`reverse_of: :taggings`（Rails自动加的）变成了下面这样：

```ruby
class Tagging < ActiveRecord::Base 
  belongs_to :tag, reverse_of: :taggings
  belongs_to :post, reverse_of: :taggings
end

post = Post.first
tag = post.tags.build name: 'ruby'
tag.save

tag.taggings # => [... ]
tag.posts    # => [post]
```
说明Rails在保存关联数据时，需要知道反向的关联关系。

#### Inverse_of in mongoid
初步看起来`mongoid`中的`reverse_of`只是起到自定义关联名称的作用，并不具备上面提到的那些功效：

```ruby
class Lush
  include Mongoid::Document
  has_many :whiskeys, class_name: "Drink", inverse_of: :alcoholic
end

class Drink
  include Mongoid::Document
  belongs_to :alcoholic, class_name: 'Lush', inverse_of: :whiskeys
end

alcoholic = Lush.first
whiskey = alcoholic.whiskeys.first

whiskey.alcoholic == alcoholic     # => true
whiskey.alcoholic.equal? alcoholic # => false（对此貌似没有好的方式可以解决）

# 会导致 N+1 问题
alcoholic.whiskeys.each { |whiskey| whiskey.some_method_which_use_alcoholic }

# 不太优雅的解决方案一（会再查一次alcoholic，但不会有N+1）
alcoholic.whiskeys.includes(:alcoholic).each { |whiskey| whiskey.some_method_which_use_alcoholic }

# 不太优雅的解决方案二（不会再查询alcoholic）
alcoholic.whiskeys.each do |whiskey|
  whiskey.alcoholic = alcoholic
  whiskey.some_method_which_use_alcoholic 
end

```
#### 参考资料
1. activerecord-4.2.5.1/lib/active_record/associations.rb
2. https://www.viget.com/articles/exploring-the-inverse-of-option-on-rails-model-associations




