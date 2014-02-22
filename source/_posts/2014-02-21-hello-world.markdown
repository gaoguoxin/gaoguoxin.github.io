---
layout: post
title: "ActiveSupport::Concern source code analyse"
date: 2014-02-21 02:44:45 +0000
comments: true
categories: Rails
---

1、首先我们来看个例子
``` ruby
module M
  def self.included(base)
    base.extend ClassMethods
    base.send(:include, InstanceMethods)
    scope :disabled, where(:disabled => true)
  end

  module ClassMethods
    #...
  end

  module InstanceMethods
    #...
  end
end
```
这是一个再普通不过的module了，当某个模块(或者类)include它以后，它让目标模块(或者类)去txtend  ClassMethods 这个模块，并去include  InstanceMethods 这个模块，这样的话，目标模块(或者类)在include M 的时候，这个目标模块(或者类)就拥有了ClassMethods这个模块中定义的方法并把这些方法当作类方法，并拥有了InstanceMethods模块的方法，并把这个模块中的方法作为自己的实例方法。如果你不觉得麻烦的话，那么这种方式是可选的或者，干脆直接定义实例方法，而不定义InstanceMethods，因为在include一个module的时候，module里面定义的方法自动就为实例方法。

2、再来看看另外一个例子
``` ruby
module Foo
  def self.included(base)
    base.class_eval do
      def self.method_injected_by_foo
        #...对宿主做某些操作，例如增強功能等等
      end
    end
  end
end

module Bar
  def self.included(base)
    base.method_injected_by_foo #这里调用了Foo这个module定义的method_injected_by_foo方法
  end
end

class Host
  include Foo
  include Bar
  #要想让让Bar被include的时候执行Foo这个module中定义的method_injected_by_foo方法，必须在include  Bar之前先要解决依赖关系，即 include Foo
end
```
上面这个例子对于大多数的使用者来说，就会很麻烦，因为我在include某个模块的时候必须解决他们的依赖关系，要知道我include的模块依赖哪些其他模块，这个对使用者来说有点困难。为了解决这个依赖的问题，貌似我们可以这样做，即将依赖关系放到module内去解决,如下:

```ruby
 module Foo
   def self.included(base)
     base.class_eval do
       def self.method_injected_by_foo
         #...对宿主做某些操作，例如增強功能等等
       end
     end
   end
 end

 module Bar
   include Foo #在这里解决依赖问题
 
   def self.included(base)
     base.send(:do_host_something)
   end
 
 end
 
 class Host
   include Bar # 这里只需要include一个Bar，而不用关心依赖问题
 end
```
以上代码看似可以执行，但是问题是，在Foo被 Bar include的时候，对于Foo来说，此时的base为Bar，而不是 class Host，所以我们没法在Host include的时候，拿到Host中的do_host_something方法，因为它根本就不在Host中

3、为了解决以上讨论的两个问题，ActiveSupport::Concern出现了，对于上面的问题，你只要这样操作就可以了:

对于第一个问题，你可以这样解决:
``` ruby
require 'active_support/concern'
module M
  extend ActiveSupport::Concern

  included do
    scope :disabled, -> { where(disabled: true) }
  end

  module ClassMethods
    #...
  end
end
```

对于第二个问题，你可以这样解决:
``` ruby
    require 'active_support/concern'
    module Foo
      extend ActiveSupport::Concern
      included do
        def self.method_injected_by_foo
          ...
        end
      end
    end
  
    module Bar
      extend ActiveSupport::Concern
      include Foo
  
      included do
        self.method_injected_by_foo
      end
    end
  
    class Host
      include Bar # 只include Bar就可以了，其他的我们不用关心
    end
```

4、接下来，我们看看 active_support/concern 的源码是怎样来实现这些逻辑的

a、首先定义了一个异常类:
``` ruby
class MultipleIncludedBlocks < StandardError
  def initialize
    super "Cannot define multiple 'included' blocks for a Concern"
  end
end
```
b、模块被extend的时候，为base添加一个类实例变量，并赋值为空数组，这个类实例变量是用来记录base模块(或者类)的所有依赖模块的
``` ruby
def self.extended(base)
  base.instance_variable_set(:@_dependencies, [])
end
```
c、重写了ruby的 append_features 方法，这个append_features 是ruby Module中的方法，愿意是当一个module(a)被include 到另一个module(b)的时候，就会调用module  a 中的这个append_features方法，并且把module b 做为参数传给append_features方法，然后将module a中的常量，方法，变量添加到module b 中。

``` ruby
def append_features(base) #此时的base代表module b
  if base.instance_variable_defined?(:@_dependencies)
    base.instance_variable_get(:@_dependencies) << self  #此时的self是module a 
    return false
  else
    return false if base < self 
    @_dependencies.each { |dep| base.send(:include, dep) }  # module b 去 循环include  modulea 的依赖 module
    super  # 这个super 调用的是ruby原来的 append_features 方法
    base.extend const_get(:ClassMethods) if const_defined?(:ClassMethods) #module b 去extend  module a 中定一的 ClassMethods 模块
    base.class_eval(&@_included_block) if instance_variable_defined?(:@_included_block)  # 为module b 动态添加一些内容(block中的内容)
  end
end
```

来看一下这个方法的具体逻辑，首先判断 module b 中是否已经有了类实例变量@_dependencies，如果有的话，那么将被include的module a 追加到这个变量中，如果一个module 中设置了 @_dependencies 这个变量 那么说明 该module 肯定extend 了active_support/concern，如果一个module中没有设置这个变量，可以肯定，该module 没有 extend  active_support/concern ，则该module是最后应用的那个module 。

d、重新定义included 方法，included方法是ruby module中的方法，当一个module a 被另一个module b include的时候，module a 的included 方法就会执行，并把module 作为一个参数 base 传递给该方法，我们可以在这里对base做一些inject，这里对included方法进行了重写。

``` ruby
def included(base = nil, &block)
  if base.nil?
    raise MultipleIncludedBlocks if instance_variable_defined?(:@_included_block)
    @_included_block = block
  else
    super
  end
end
```

当module a 被module include 后，如果这个方法传递了一个base，那么就直接使用ruby module中的included方法，如果我们在include的时候，传递的是一个block，那么就将该block 转化为proc对象，并赋值给 @_included_block 变量，这个变量会在 上边的 append_features方法中使用，这个block一般是用来动态给base 添加一些方法活属性用。例如:

``` ruby
require 'active_support/concern'
module M
  extend ActiveSupport::Concern

  included do  #这里调用included方法，我们只传递了一个block，并且这个block中用来声明一些方法，将来这些方法会成为base的类方法
    scope :disabled, -> { where(disabled: true) }
  end

  module ClassMethods
    #...
  end
end
```

