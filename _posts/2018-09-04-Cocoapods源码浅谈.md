---
layout:     post
title:      Cocoapods源码浅谈
date:       2018-09-04
author:     开不了口的猫
header-img: img/icon_cocoapods_reading_bg.jpeg
catalog: true
tags:
    - Cocoapods
    - Ruby
    - 阅读笔记
---

# 第一部分：CLI指令
### 核心类-Pod::Command
Pod模块中的Command类是Cocoapods组件中最核心的指令基类，它继承于CLAide模块的Command类。我们熟悉的pod命令的所有子命令都是Pod::Command类的子类。<br><br>
例如，当我们在终端里调用`pod lib lint`时，内部的调用顺序为以下三个步骤：
1. 首先会调用到Pod模块中Command类的`run`方法，接收参数列表`['lib', 'lint']`
2. 调用CLAide模块的Command类的`run`方法，解析参数
3. CLAide模块的Command类通过查找已记录的所有Command类继承者列表，找到对应的Lint类，执行子类的`validate!`和`run`方法

说一下细节。先来看Pod模块Command类的`run`方法，

{% highlight ruby %}
def self.run(argv)
    help! 'You cannot run CocoaPods as root.' if Process.uid == 0

    verify_minimum_git_version!
    verify_xcode_license_approved!

    super(argv)
ensure
    UI.print_warnings
end
{% endhighlight %}

主要就是校验了一下Git和XCode的兼容性，然后调用了`super`即CLAide::Command类的run方法，将argv参数列表传递过去。<br>
<br>
再来看CLAide::Command的`run`方法，

{% highlight ruby %}
def self.run(argv = [])
    plugin_prefixes.each do |plugin_prefix|
    PluginManager.load_plugins(plugin_prefix)
    end

    argv = ARGV.coerce(argv)
    command = parse(argv)
    ANSI.disabled = !command.ansi_output?
    unless command.handle_root_options(argv)
    command.validate!
    command.run
    end
rescue Object => exception
    handle_exception(command, exception)
end
{% endhighlight %}

首先加载了Cocoapods插件列表中的所有插件(这一步我们暂时先忽略)，`ARGV.coerce`方法确保了传递进来的参数会被强转成ARGV类的实例，紧接着会调用`parse`方法，

{% highlight ruby %}
def self.parse(argv)
    argv = ARGV.coerce(argv)
    cmd = argv.arguments.first
    if cmd && subcommand = find_subcommand(cmd)
    argv.shift_argument
    subcommand.parse(argv)
    elsif abstract_command? && default_subcommand
    load_default_subcommand(argv)
    else
    new(argv)
    end
end
{% endhighlight %}

首先取出传递进来的参数列表的第一个参数，然后通过`find_subcommand`方法去查找是否存在与这个参数名称相等的Command子类，其中`CLAide#command`方法是将诸如Pod::Command::Lib类解析成`lib`。

{% highlight ruby %}
def self.find_subcommand(name)
    subcommands_for_command_lookup.find { |sc| sc.command == name }
end
{% endhighlight %}

那么已记录的subcommands是哪里来的呢？我们来看一下CLAide::Command类实现的另一个方法`self.inherited`，

{% highlight ruby %}
def self.inherited(subcommand)
    subcommands << subcommand
end
{% endhighlight %}

这其实是Ruby原生API中的方法，覆写了这个方法的实现后，如果存在别的子类直接继承了这个类，那么这个方法就会自动被调用，这里很明显，如果有子类直接继承了CLAide::Command类，那么就会被添加到当前这个类的subcommands类数组属性中，同理可得，Pod::Command和Pod::Command::Lib也是如此。好了，让我们再回到`parse`方法的探索中，当我们一旦找到了下一级pod命令参数，就会调用

{% highlight ruby %}
argv.shift_argument
subcommand.parse(argv)
{% endhighlight %}

`ARGV#shift_argument`是用来删除参数列表中的首个元素，然后将剩余的参数列表继续分发到之前找到的Pod::Command子类，递归调用parse方法，最后仅当`argv.arguments.first`为空时，返回最终子类的实例。然后parse结束，返回给了command变量，紧接着判断argv中的flag，例如`--allow-warnning`又或者是`--version`等，检测除了`--version`，都会直接调用这个子类实例的`validate!`和`run`方法。

{% highlight ruby %}
unless command.handle_root_options(argv)
    command.validate!
    command.run
end
{% endhighlight %}

至此，Command核心类的命令派发流程就结束了。

### pod install
直接看`Pod::Command::Install`类的run方法：

{% highlight ruby %}
def run
    verify_podfile_exists!
    installer = installer_for_config
    installer.repo_update = repo_update?(:default => false)
    installer.update = false
    installer.install!
end
{% endhighlight %}

第一步，通过调用`Pod::Command#verify_podfile_exists!`方法验证podfile是否存在，这个方法会在很多地方用到。

{% highlight ruby %}
def verify_podfile_exists!
    unless config.podfile
        raise Informative, "No `Podfile' found in the project directory."
    end
end
{% endhighlight %}

其中config是Config类的单例。

{% highlight ruby %}
def self.instance
    @instance ||= new
end
{% endhighlight %}

然后podfile是config这个单例的属性，同样是懒加载的。

{% highlight ruby %}
def podfile
    @podfile ||= Podfile.from_file(podfile_path) if podfile_path
end
attr_writer :podfile
{% endhighlight %}

这里会调用到`Podifle.from_file`方法，

{% highlight ruby %}
def self.from_file(path)
    path = Pathname.new(path)
        unless path.exist?
        raise Informative, "No Podfile exists at path `#{path}`."
    end

    case path.extname
    when '', '.podfile'
        Podfile.from_ruby(path)
    when '.yaml'
        Podfile.from_yaml(path)
    else
        raise Informative, "Unsupported Podfile format `#{path}`."
    end
end
{% endhighlight %}

这里主要针对Podifle的几种命名形式`CocoaPods.podfile.yaml`、`CocoaPods.podfile`、`Podfile`进行分别的解析，不过一般pod是通过`pod lib create`来生成的话，默认名称就是Podfile，那我们就以Podfile为例，来看一下`Podfile.from_ruby`方法。

{% highlight ruby %}
    def self.from_ruby(path, contents = nil)
      contents ||= File.open(path, 'r:utf-8', &:read)

      # Work around for Rubinius incomplete encoding in 1.9 mode
      if contents.respond_to?(:encoding) && contents.encoding.name != 'UTF-8'
        contents.encode!('UTF-8')
      end

      if contents.tr!('“”‘’‛', %(""'''))
        # Changes have been made
        CoreUI.warn "Smart quotes were detected and ignored in your #{path.basename}. " \
                    'To avoid issues in the future, you should not use ' \
                    'TextEdit for editing it. If you are not using TextEdit, ' \
                    'you should turn off smart quotes in your editor of choice.'
      end

      podfile = Podfile.new(path) do
        # rubocop:disable Lint/RescueException
        begin
          # rubocop:disable Eval
          eval(contents, nil, path.to_s)
          # rubocop:enable Eval
        rescue Exception => e
          message = "Invalid `#{path.basename}` file: #{e.message}"
          raise DSLError.new(message, path, e, contents)
        end
        # rubocop:enable Lint/RescueException
      end
      podfile
    end
{% endhighlight %}

首先读取了Podfile文件的内容，然后将Podfile中的一些因为编辑器导致的不规范的单引号或双引号进行自动修正，然后通过Ruby的`eval`方法执行这些内容，这里会涉及到DSL(领域专属语言)的概念，不熟悉的朋友可以先阅读一下[这篇](https://draveness.me/dsl)文章。<br>
`Pod::Podfile::DSL`模块定义了Podfile语法中所有的方法的实现，在Podfile类中Mix-in了DSL模块，并且在Podfile的`initialize`方法中通过`instance_eval`执行了这个block，将Podfile文件中调用的方法执行一遍，DSL模块预先定义的方法这里就不赘述了，大多都会执行

{% highlight ruby %}
def set_hash_value(key, value)
    unless HASH_KEYS.include?(key)
        raise StandardError, "Unsupported hash key `#{key}`"
    end
    internal_hash[key] = value
end
{% endhighlight %}

这个方法，将Podfile中的配置项缓存在Config的podfile实例的internal_hash中。<br>
<br>
然后再看执行pod install时调用的第二个方法`Pod::Command#installer_for_config`

{% highlight ruby %}
installer = installer_for_config
{% endhighlight %}

{% highlight ruby %}
def installer_for_config
    Installer.new(config.sandbox, config.podfile, config.lockfile)
end
{% endhighlight %}

通过由config单例懒加载自动生成的Sandbox、Podfile、Lockfile配置类实例生成`Installer`类实例，这个实例是执行核心方法`install!`的真正的对象。<br>
紧接着判断用户在终端执行pod install时是否携带了`--repo-update`标识，这将决定在pod install执行之前是否会先去执行`pod repo update`操作。

{% highlight ruby %}
installer.repo_update = repo_update?(:default => false)
{% endhighlight %}

pod install的内部实现步骤其实和pod update非常相似。所以update属性被用来区分正在进行的是两者中哪一种操作。

{% highlight ruby %}
installer.update = false
{% endhighlight %}

最后让我们来分析一下pod install中的核心调用 —— `installer.install!`。

{% highlight ruby %}
    def install!
      prepare
      resolve_dependencies
      download_dependencies
      validate_targets
      generate_pods_project
      if installation_options.integrate_targets?
        integrate_user_project
      else
        UI.section 'Skipping User Project Integration'
      end
      perform_post_install_actions
    end
{% endhighlight %}

先来分析prepare方法的实现。

{% highlight ruby %}
    def prepare
      # Raise if pwd is inside Pods
      if Dir.pwd.start_with?(sandbox.root.to_path)
        message = 'Command should be run from a directory outside Pods directory.'
        message << "\n\n\tCurrent directory is #{UI.path(Pathname.pwd)}\n"
        raise Informative, message
      end
      UI.message 'Preparing' do
        deintegrate_if_different_major_version
        sandbox.prepare
        ensure_plugins_are_installed!
        run_plugins_pre_install_hooks
      end
    end
{% endhighlight %}

最后总结一下install!方法主要要干的几件事：
1. prepare 准备工作
    * 检查安装目录,必须在项目根目录
    * 检查Podfile.lock文件cocoapods版本,如果主版本不一样, 会重新集成cocoapods
    * 创建安装目录Pods及子目录
    * 检查Podfile中的plugin插件都已经安装并加载
    * 加载插件
2. resolve_dependencies 解决依赖
    * 检查是否需要更新podsource源
    * 如果Podfile中有删除的库, 进行清理文件
3. download_dependencies 下载依赖库
    * 下载各个pod库
    * 执行Podfile中pre_install钩子方法
4. validate_targets 验证target和pod正确
5. generate_pods_project 生成'Pods/Pods.xcodeproj工程
    * 调用Podfile中post_install钩子方法
    * 生成Pods工程
    * 生成Podfile.lock文件和Manifest.lock文件
6. integrate_user_project 集成
    * 创建.xcworkspace文件
    * 集成Target
    * 警告检查
    * 保存.xcworkspace文件到目录
7. 调用plugin的post_install钩子方法

### pod update



