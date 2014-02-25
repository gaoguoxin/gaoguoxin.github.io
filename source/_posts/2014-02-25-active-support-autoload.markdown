---
layout: post
title: "active_support_autoload"
date: 2014-02-25 05:43:38 +0000
comments: true
categories: Rails
---

在看rails源码的时候发现很多地方用到了autoload，意识到这个autoload是整个框架的基础方法，于是有时间看了遍里面的代码，关键的方法都定义在autoload.rb(active_support/dependencies/autoload.rb)下面我们来看下这个文件里面都做了些什么事:

``` ruby

def self.extended(base)
  base.class_eval do
    @_autoloads = {} #hash 盛放该module需要一次性require的文件,每个元素的key为大写的module的名字，value为该module对应的文件(包含路径)
    @_under_path = nil #字符串，代表需要load的某个模块的路径
    @_at_path = nil    #字符串，代表需要load的某个模块的路径
    @_eager_autoload = false #标识位
  end
end

```

以上代码，很明显，给base(某个可能extend ActiveSupport::Autoload的模块)声明了几个实例变量。
<!--more-->
再来看看autoload方法,这个方法是我们在rails源码中看到最多的
``` ruby

def autoload(const_name, path = @_at_path)
  unless path
    full = [name, @_under_path, const_name.to_s].compact.join("::")#生成一个用::隔开的字符串
    path = Inflector.underscore(full)#生成要加载模块的路径 例如:ActiveSupport #=> active_support
  end

  if @_eager_autoload
    @_autoloads[const_name] = path
  end

  super const_name, path #调用ruby的autoload方法，这个方法是一个懒加载，即只有用到模块中的方法或者变量的时候，才会去require该文件
end

```

接下来的几个方法:

``` ruby
def autoload_under(path)
  @_under_path, old_path = path, @_under_path #设置了@_under_path变量
  yield
ensure
  @_under_path = old_path
end

def autoload_at(path)
  @_at_path, old_path = path, @_at_path#设置了@_at_path变量
  yield
ensure
  @_at_path = old_path
end

def eager_autoload #这个方法用来一次性require所有所需的文件
  old_eager, @_eager_autoload = @_eager_autoload, true#设置了@_eager_autoload变量为true
  yield
ensure
  @_eager_autoload = old_eager
end
```

以上这三个方法都是后面跟着block的，所以这三个函数都yield了这个block，并且设置了base模块的几个基本的实例变量，这些block中的内容，如果你才看看rails源码的话，你会发现，基本上都是在里面调用了autoload方法:

``` ruby
autoload_under "renderer" do
  autoload :Renderer
  autoload :AbstractRenderer
  autoload :PartialRenderer
  autoload :TemplateRenderer
  autoload :StreamingTemplateRenderer
end

autoload_at "action_view/template/resolver" do
  autoload :Resolver
  autoload :PathResolver
  autoload :OptimizedFileSystemResolver
  autoload :FallbackFileSystemResolver
end

eager_autoload do
  autoload :Base
  autoload :Context
  autoload :CompiledTemplates, "action_view/context"
  autoload :Digestor
  autoload :Helpers
  autoload :LookupContext
  autoload :Layouts
  autoload :PathSet
  autoload :RecordIdentifier
  autoload :Rendering
  autoload :RoutingUrlFor
  autoload :Template
  autoload :ViewPaths
end
 
```

这个时候我们回过头来在来看 autoload方法，里面有这么一句:

``` ruby

if @_eager_autoload
  @_autoloads[const_name] = path
end

```
也就是说，只要我们调用了eager_autoload这个方法，我们就会设置@_eager_autoload值为true，并且我们还设置了@_autoloads这个hash，这个hash就是用来盛放我们将来需要一次性require的文件的路径


再来看看最重要的一个方法:

``` ruby

def eager_load!
  @_autoloads.values.each { |file| require file }
end

```

这个方法是我们最终实现饥饿加载的最终代码所在，即一次性require我们所需的所有的module文件。

这个autoload模块就是用来方便我们require我们所需要的module文件的,如果一个module extend了这个autoload模块的话，那么他就可以利用autoload这个方法来加载同一个module 名命名的文件加下的其他模块文件而不用制定文件的路径。还可以通过eager_autoload方法来指定这个module需要批量require 的module的名字，然后通过 eager_load!方法就可以实现一次性require多个文件，这个方法的好处是避免了我们在文件的开头写很多的require，如下面的例子所示:

``` ruby

module MyLib
  extend ActiveSupport::Autoload

  autoload :Model

  eager_autoload do
    autoload :Cache
  end
end

Then your library can be eager loaded by simply calling:

MyLib.eager_load!

```





