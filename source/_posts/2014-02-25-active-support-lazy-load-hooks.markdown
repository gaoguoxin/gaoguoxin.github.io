---
layout: post
title: "active_support_lazy_load_hooks"
date: 2014-02-25 08:42:32 +0000
comments: true
categories: Rails
---

在rails的源码中经常看到有ActiveSupport.on_load这样的代码出现，于是找到代码的定义处(active_support/lazy_load_hooks.rb)中，里面的代码很简单，但是为我们提供的功能却很强大，下面我们来看一下:

``` ruby

@load_hooks = Hash.new { |h,k| h[k] = [] } #定义一个空的hash,用来存放还没有被执行的blocks
@loaded = Hash.new { |h,k| h[k] = [] } #定义一个空hash,这个空hash主要用来存放已经被加载的载体类。

def self.on_load(name, options = {}, &block)#这个方法是见到最多的，很多我们需要将来执行的代码块儿都是通过这个方法存储的
  @loaded[name].each do |base|
    execute_hook(base, options, block)
  end

  @load_hooks[name] << [block, options] #每次调用该方法时，该方法后面block要执行的动作都被暂时保存在这个变量中
end

def self.execute_hook(base, options, block) #这个方法主要是用来执行存储的代码用
  if options[:yield]
    block.call(base)
  else
    base.instance_eval(&block)
  end
end

def self.run_load_hooks(name, base = Object) #执行开始的地方，当某个文件require了这个module之后，并切调用了这个方法之后，执行开始
  @loaded[name] << base
  @load_hooks[name].each do |hook, options|
    execute_hook(base, options, hook)
  end
end

```

下面来看一下这个azy_load_hooks用在什么地方，我们打开active_record/railtie.rb这个文件，我们会看见很多类似的代码:
<!-- more -->

``` ruby

ActiveSupport.on_load(:active_record) do
  filename = File.join(app.config.paths["db"].first, "schema_cache.dump")

  if File.file?(filename)
    cache = Marshal.load File.binread filename
    if cache.version == ActiveRecord::Migrator.current_version
      self.connection.schema_cache = cache
    else
      warn "Ignoring db/schema_cache.dump because it has expired. The current schema version is #{ActiveRecord::Migrator.current_version}, but the one in the cache is #{cache.version}."
    end
  end
end

```

方法后的block是我们真正要做的额外的操作这些操作可能会延长我们的app的启动的过程，所以，我们在这里使用on_load将这些要执行的方法暂时存放在一个变量里，只有在程序需要的时候才去执行这些额外的操作，这样的话，我们的app启动的就会更快

最后的执行是在active_record/base.rb文件中，代码的最下面执行到了run_load_hooks方法:

``` ruby 

ActiveSupport.run_load_hooks(:active_record, Base) #调用run_load_hooks，额外的操作开始执行。

```

如果你打开rails其他的railtie的话，你会发现也有azy_load_blocks方法，他们起到的作用都是一样的，即，提高我们应用启动的速度
