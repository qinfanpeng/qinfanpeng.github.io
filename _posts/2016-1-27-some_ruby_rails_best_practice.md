---
layout: post
title:  "Rails & Ruby Best Practice"
date:   2016-1-27 12:55:19 +0800
categories: jekyll update
---

> 其中部分示例稍老，但其思想仍然适用

## Setup
```ruby
gem install rails_best_practices

rails_best_practices .
```

## default_scope is Evil
```ruby
class Post
  default_scope where(published: true).order("created_at desc")
end
```
you can't override default scope

```ruby 
Post.order("updated_at desc").limit(10) # =>
# Post Load (17.3ms)  SELECT `posts`.* FROM `posts` WHERE `posts`.`published` = 1 ORDER BY created_at desc, updated_at desc LIMIT 10

Post.unscoped.order("updated_at desc").limit(10) # =>
# Post Load (1.9ms)  SELECT `posts`.* FROM `posts` ORDER BY updated_at desc LIMIT 10
``` 
default_scope will affect your model initialization

```ruby
Post.new # =>
<Post id: nil, published: true, title: nil, created_at: nil, updated_at: nil, user_id: nil>
```

## Not Rescue Exception
```ruby
loop do
  begin
    sleep 1
    eval "djsakru3924r9eiuorwju3498 += 5u84fior8u8t4ruyf8ihiure"
  rescue Exception => e
    puts "I refuse to fail or be stopped!"
  end
end
```

### Ruby Exception Class Hierarchy
```ruby
Exception
  NoMemoryError
  ScriptError
    LoadError
    NotImplementedError
    SyntaxError
  SignalException
    Interrupt
      Timeout::Error    # < ruby 1.9.2
  StandardError         # caught by rescue (default if no type was specified)
    ArgumentError
    IOError
      EOFError
    IndexError
    LocalJumpError
    NameError
      NoMethodError
    RangeError
      FloatDomainError
    RegexpError
    RuntimeError
      Timeout::Error    # >= ruby 1.9.2
    SecurityError
    SocketError
    SystemCallError
    SystemStackError
    ThreadError
    TypeError
    ZeroDivisionError
  SystemExit
  fatal
```

## Rescue StandardError, Not Exception
```ruby
begin
  # iceberg!
rescue => e
  # lifeboats
end

# which is equivalent to:
begin
  # iceberg!
rescue StandardError => e
  # lifeboats
end
```

## Rescue Exception for Logging/Reporting 
```ruby
begin
  # iceberg?
rescue Exception => e
  # do some logging
  raise e  # not enough lifeboats ;)
end
```

## Needless Deep Nesting
```ruby
# config/routes.rb
resources :auctions do
  resources :bids do
    resources :comments 
  end
end
```
```ruby
edit_auction_bid_comment_path(@auction, @bid, @comment)
```
- As we already have the resource_id parameter in the URLs for `#show`, `#edit`, `#update` and `#destroy` 
- So only need to nest `#index`, `#new` and `#create` under the resource

## Rewrite Deep Nesting
```ruby
resources :auctions do
  resources :bids, only: [:index, :new, :create]
end

resources :bids, only: [:show, :edit, :update, :destroy]
```

Or

```ruby
resources :auctions, shallow: true do 
  resources :bids do
    resources :comments 
  end
end
```
Or

```ruby
resources :auctions do 
   resources :bids
end

resources :bids do 
  resources :comments
end

resources :comments
```

## Shallow Routes
```ruby
bid_comments GET    /bids/:bid_id/comments(.:format)
             POST   /bids/:bid_id/comments(.:format)
new_bid_comment GET /bids/:bid_id/comments/new(.:format)
   edit_comment GET /comments/:id/edit(.:format)
        comment GET /comments/:id(.:format)
             PATCH  /comments/:id(.:format)
             PUT    /comments/:id(.:format)
             DELETE /comments/:id(.:format)
auction_bids GET    /auctions/:auction_id/bids(.:format)
             POST   /auctions/:auction_id/bids(.:format)
new_auction_bid GET /auctions/:auction_id/bids/new(.:format)
       edit_bid GET /bids/:id/edit(.:format)
            bid GET /bids/:id(.:format)
             PATCH  /bids/:id(.:format)
             PUT    /bids/:id(.:format)
             DELETE /bids/:id(.:format)
    auctions GET    /auctions(.:format)
             POST   /auctions(.:format)
    new_auction GET /auctions/new(.:format)
   edit_auction GET /auctions/:id/edit(.:format)
        auction GET /auctions/:id(.:format)
     ...
```

## Replace Instance Variable With Local Variable

### Downside of using instance variables in partials

```ruby
class PostsController < ApplicationController
  def show
    @item = ...
  end
end
# sidebar partial use @item directly
<%= render partial: "sidebar" %>
```

The downside of using instance variables in partials is that you **create a dependency** in the partial to something outside the partial's scope (coupling). This makes the partial **harder to reuse**, and can force changes in several parts of the application when you want to make a change in one part. 

Partials that use instance variables in partial:

1. **Discourage reuse**, as they can only easily be reused in actions that set up instance variables with the same name and data 
2. Shotgun Surgy，Must change when the instance variable in any controller that uses the partial changes either the instance variable name or its type or data structure, *and vice versa*.


## Replace Instance Variable With Local Variable

### Instead, pass locals to the partials

```ruby
<%= render 'reusable_partial', item: @item %>
```
the partial only references item and not @item.

```ruby
<%= render 'reusable_partial', :item => @other_object.item %>
```
Also, this can be reused in contexts where there is no @item:

```ruby
<%= render 'reusable_partial', :item => @duck %>
```

Now, because the partial only references item and not @item, the action that renders the view that renders the reusable_partial is free to change without affecting the reusable_partial and the other actions/views that render it: 

If my @duck changes in the future and no longer quacks like reusable_partial expects it to (the object's interface changes), I can also use an adapter to pass in the kind of item that reusable_partial expects: 

@duck changes in the future, I can also use an adapter.

```ruby
<%= render 'reusable_partial', :item => itemlike_duck(@duck) %>
```

## Use Scope Access
Check the permission by comparing the owner of object with current_user

```ruby
class PostsController < ApplicationController
  def edit
    @post = Post.find(params[:id])
    if @post.user != current_user
      flash[:warning] = 'Access denied'
      redirect_to posts_url
    end
  end
end
```
Just use scope access to make permission check simpler.

```ruby
class PostsController < ApplicationController
  def edit
    # raise RecordNotFound exception (404 error) if not found
    @post = current_user.posts.find(params[:id])
  end
end
```

## The Law of Demeter
Code Smell

```ruby
class Invoice < ActiveRecord::Base
  belongs_to :user
end

<%= @invoice.user.name %>
<%= @invoice.user.address %>
<%= @invoice.user.cellphone %>
```
Refactor

```ruby
class Invoice < ActiveRecord::Base
  belongs_to :user
  delegate :name, :address, :cellphone, :to => :user, :prefix => true
end

<%= @invoice.user_name %>
<%= @invoice.user_address %>
<%= @invoice.user_cellphone %>
```

## Keep Finders on Their Own Model
Bad Smell

```ruby
class Post < ActiveRecord::Base
  has_many :comments

  def find_valid_comments
    self.comment.find(:all, :conditions => {:is_spam => false},
                            :limit => 10)
  end
end

class Comment < ActiveRecord::Base
  belongs_to :post
end

class CommentsController < ApplicationController
  def index
    @comments = @post.find_valid_comments
  end
end
```
```ruby
class Post < ActiveRecord::Base
  has_many :comments
end

class Comment < ActiveRecord::Base
  belongs_to :post

  scope :only_valid, -> { is_spam: false }
  scope :limit, lambda { |size| { :limit => size } }
end

class CommentsController < ApplicationController
  def index
    @comments = @post.comments.only_valid.limit(10)
  end
end
```

## Use query attribute
Code Smell

```ruby
<% if @user.login.blank? %>
  <%= link_to 'login', new_session_path %>
<% end %>

<% if @user.login.present? %>
  <%= @user.login %>
<% end %>
```
Refactor

```ruby
<% unless @user.login? %>
  <%= link_to 'login', new_session_path %>
<% end %>

<% if @user.login? %>
  <%= @user.login %>
<% end %>
```

## Simplify Render in Views
Before

```ruby
<%= render :partial => 'sidebar' %>
<%= render :partial => 'shared/sidebar', :locals => { :parent => post } %>
```
After

```ruby 
<%= render 'sidebar' %>
<%= render 'shared/sidebar', :parent => post %>
```

## About The Safe Navigation Operator(&.)

```ruby
if account && account.owner && account.owner.address
...
end
```
Isn't identical to

```ruby
if account.try(:owner).try(:address)
...
end
```
Isn't identical to

```ruby
if account&.owner&.address
...
end
```

### There're similar when something is nil

```ruby
account = Account.new(owner: nil) # account without an owner

account.owner.address
# => NoMethodError: undefined method `address' for nil:NilClass

account && account.owner && account.owner.address
# => nil

account.try(:owner).try(:address)
# => nil

account&.owner&.address
# => nil
```

### &. operator **only skips nil** but **recognizes false**! 

```ruby
account = Account.new(owner: false)

account.owner.address
# => NoMethodError: undefined method `address' for false:FalseClass `

account && account.owner && account.owner.address
# => false

account.try(:owner).try(:address)
# => nil

account&.owner&.address
# => undefined method `address' for false:FalseClass`
```

### When use `try`, whatch out misspelling!

```ruby
account = Account.new(owner: Object.new)

account.owner.address
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>

account && account.owner && account.owner.address
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>`

account.try(:owner).try(:address)
# => nil

account&.owner&.address
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>`
```

There's a stricter version of try - `try!` for you:

```ruby
account.try!(:owner).try!(:address)
# => NoMethodError: undefined method `address' for #<Object:0x00559996b5bde8>`
```

## References

1. [http://thepugautomatic.com/2013/05/locals/](http://thepugautomatic.com/2013/05/locals/)
2. [http://stackoverflow.com/questions/2503838/rails-should-partials-be-aware-of-instance-variables](http://stackoverflow.com/questions/2503838/rails-should-partials-be-aware-of-instance-variables)
3. [http://nithinbekal.com/posts/rails-shallow-nesting/](http://nithinbekal.com/posts/rails-shallow-nesting/)
