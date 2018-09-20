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

首先检查了当前路径是否是Podfile文件所在的项目根目录，然后调用`deintegrate_if_different_major_version`

{% highlight ruby %}
    def deintegrate_if_different_major_version
      return unless lockfile
      return if lockfile.cocoapods_version.major == Version.create(VERSION).major
      UI.section('Re-creating CocoaPods due to major version update.') do
        projects = Pathname.glob(config.installation_root + '*.xcodeproj').map { |path| Xcodeproj::Project.open(path) }
        deintegrator = Deintegrator.new
        projects.each do |project|
          config.with_changes(:silent => true) { deintegrator.deintegrate_project(project) }
          project.save if project.dirty?
        end
      end
    end
{% endhighlight %}

检查podfile.lock的写入的cocoapods版本和当前cocoapods版本是否一致，如果不一致将会重塑工程，将除了Podfile、Podfile.lock、Workspace以外的其他关联和依赖全部重置。其中的核心方法`deintegrate_project`来自另一个cocoapods的gem - [cocoapods-deintegrate](https://github.com/CocoaPods/cocoapods-deintegrate)。这里不继续深究了，有兴趣的可以自行查看源码。<br>
<br>
然后调用`Sandbox#prepare`方法建立项目沙盒(Pods)中几个重要的文件夹，主要包括Headers、root、Local Podspecs、Target Support Files等。<br>

{% highlight ruby %}
    def ensure_plugins_are_installed!
      require 'claide/command/plugin_manager'


      loaded_plugins = Command::PluginManager.specifications.map(&:name)

      podfile.plugins.keys.each do |plugin|
        unless loaded_plugins.include? plugin
          raise Informative, "Your Podfile requires that the plugin `#{plugin}` be installed. Please install it and try installation again."
        end
      end
    end
{% endhighlight %}

`ensure_plugins_are_installed!`方法会检查Podfile中通过`plugin`语法应用的插件在你本地RubyGems中是否已经安装，如果没有安装，将会报错。<br>

{% highlight ruby %}
    def run_plugins_pre_install_hooks
      context = PreInstallHooksContext.generate(sandbox, podfile, lockfile)
      HooksManager.run(:pre_install, context, plugins)
    end
{% endhighlight %}

`run_plugins_pre_install_hooks`方法会由sandbox、podfile、lockfile生成`PreInstallHooksContext`类的实例context，然后通过`HooksManager.run`方法遍历通过`HooksManager.register`方法注册`pre_install`切片的所有插件，执行他们注册时传入的block处理块。

{% highlight ruby %}
    def run(name, context, whitelisted_plugins = nil)
        raise ArgumentError, 'Missing name' unless name
        raise ArgumentError, 'Missing options' unless context

        if registrations
          hooks = registrations[name]
          if hooks
            UI.message "- Running #{name.to_s.tr('_', ' ')} hooks" do
              hooks.each do |hook|
                next if whitelisted_plugins && !whitelisted_plugins.key?(hook.plugin_name)
                UI.message "- #{hook.plugin_name} from " \
                           "`#{hook.block.source_location.first}`" do
                  block = hook.block
                  if block.arity > 1
                    user_options = whitelisted_plugins[hook.plugin_name]
                    user_options = user_options.with_indifferent_access if user_options
                    block.call(context, user_options)
                  else
                    block.call(context)
                  end
                end
              end
            end
          end
        end
    end
{% endhighlight %}

这里也顺带提一下制作自己插件过程中，如果想要通过HooksManager来注入不同时机的切片处理的方式。可以像下面这样在诸如pre_install、post_install或post_update等时机加入自定义的处理逻辑。

{% highlight ruby %}
    Pod::HooksManager.register('plugin_name', :post_install)
    do |context|
        # my own handler
    end
    Pod::HooksManager.register('plugin_name', :post_update)
    do |context|
        # my own handler
    end
{% endhighlight %}

至此prepare要做的预备工作就基本结束了，接下来看`resolve_dependencies`方法的实现过程。

{% highlight ruby %}
    def resolve_dependencies
      plugin_sources = run_source_provider_hooks
      analyzer = create_analyzer(plugin_sources)

      UI.section 'Updating local specs repositories' do
        analyzer.update_repositories
      end if repo_update?

      UI.section 'Analyzing dependencies' do
        analyze(analyzer)
        validate_build_configurations
        clean_sandbox
      end
      analyzer
    end
{% endhighlight %}

`run_source_provider_hooks`将遍历注册的所有插件，其中通过`HooksManager.register`方法注册name为`:source_provider`的插件，将会执行对应的处理block，并返回这些sources。<br>
紧接着执行`create_analyzer`方法创建安装分析器。

{% highlight ruby %}
    def create_analyzer(plugin_sources = nil)
      Analyzer.new(sandbox, podfile, lockfile, plugin_sources).tap do |analyzer|
        analyzer.installation_options = installation_options
        analyzer.has_dependencies = has_dependencies?
      end
    end
{% endhighlight %}

以下是Analyzer类的构造方法，

{% highlight ruby %}
    def initialize(sandbox, podfile, lockfile = nil, plugin_sources = nil)
        @sandbox  = sandbox
        @podfile  = podfile
        @lockfile = lockfile
        @plugin_sources = plugin_sources

        @update = false
        @allow_pre_downloads = true
        @has_dependencies = true
        @test_pod_target_analyzer_cache = {}
        @test_pod_target_key = Struct.new(:name, :pod_targets)
        @podfile_dependency_cache = PodfileDependencyCache.from_podfile(podfile)
    end
{% endhighlight %}

以上主要就是将所有预先配置好的配置项丢给Analyzer类，通过Analyzer类来专门负责分析并处理依赖关系。<br>
<br>
紧接着，`resolve_dependencies`的实现源码中出现了我们在终端执行`pod install`时频繁出现的提示语`Updating local specs repositories`，如果我们在执行pod install时附加了`--repo-update`flag，则刚才创建的analyzer实例将执行`update_repositories`方法去更新本地repo仓库的所有pod spec文件。

{% highlight ruby %}
    def update_repositories
        sources.each do |source|
          if source.git?
            config.sources_manager.update(source.name, true)
          else
            UI.message "Skipping `#{source.name}` update because the repository is not a git source repository."
          end
        end
        @specs_updated = true
    end
{% endhighlight %}

这里将获取所有的sources，我们来看`Analyzer#sources`方法，sources包括官方的repo源、Podfile文件中调用`source`方法引入的repo源、还有部分插件引入的源。

{% highlight ruby %}
    def sources
        @sources ||= begin
          sources = podfile.sources
          plugin_sources = @plugin_sources || []

          # Add any sources specified using the :source flag on individual dependencies.
          dependency_sources = @podfile_dependency_cache.podfile_dependencies.map(&:podspec_repo).compact
          all_dependencies_have_sources = dependency_sources.count == @podfile_dependency_cache.podfile_dependencies.count

          if all_dependencies_have_sources
            sources = dependency_sources
          elsif has_dependencies? && sources.empty? && plugin_sources.empty?
            sources = ['https://github.com/CocoaPods/Specs.git']
          else
            sources += dependency_sources
          end

          result = sources.uniq.map do |source_url|
            config.sources_manager.find_or_create_source_with_url(source_url)
          end
          unless plugin_sources.empty?
            result.insert(0, *plugin_sources)
          end
          result
        end
    end
{% endhighlight %}

其中`Source::Manager#find_or_create_source_with_url`方法主要是验证手动添加的sources的url是否是有效的，如果有效，则会直接调用`Command::Repo::Add.parse.run`，相当于执行了`pod repo add`命令。<br>
<br>
让我们回到`Analyzer#update_repositories`方法中，再来窥探一下另一个方法`Pod::Source::Manager#update`。

{% highlight ruby %}
    def update(source_name = nil, show_output = false)
        if source_name
          sources = [git_source_named(source_name)]
        else
          sources =  git_sources
        end

        changed_spec_paths = {}
        sources.each do |source|
          UI.section "Updating spec repo `#{source.name}`" do
            changed_source_paths = source.update(show_output)
            changed_spec_paths[source] = changed_source_paths if changed_source_paths.count > 0
            source.verify_compatibility!
          end
        end
        # Perform search index update operation in background.
        update_search_index_if_needed_in_background(changed_spec_paths)
    end
{% endhighlight %}

其中遍历了所有的source源，提示`Updating spec repo sourcename`的同时执行`Source#update`方法，并在执行更新完毕后，在后台开启了子进程用于更新pod search的索引。来看一下Source#update方法的具体实现。

{% highlight ruby %}
    def update(show_output)
      return [] if unchanged_github_repo?
      prev_commit_hash = git_commit_hash
      update_git_repo(show_output)
      @versions_by_name.clear
      refresh_metadata
      if version = metadata.last_compatible_version(Version.new(CORE_VERSION))
        tag = "v#{version}"
        CoreUI.warn "Using the `#{tag}` tag of the `#{name}` source because " \
          "it is the last version compatible with CocoaPods #{CORE_VERSION}."
        repo_git(['checkout', tag])
      end
      diff_until_commit_hash(prev_commit_hash)
    end
{% endhighlight %}

这个方法中首先检查了当前的source指向的Git远程仓库是否有变化，如果没有变化则直接返回空数组。如果发现远程仓库有新的更新，则会接着调用`update_git_repo`方法。

{% highlight ruby %}
    def update_git_repo(show_output = false)
      repo_git(['checkout', git_tracking_branch])
      output = repo_git(%w(pull --ff-only), :include_error => true)
      CoreUI.puts output if show_output
    end
{% endhighlight %}

这个方法使用到了`repo_git`，而这个方法是使用仓库本地的git配置去执行一些git命令并根据show_output参数选择是否需要输出，这里是执行了git pull更新本地spec源仓库。而`diff_until_commit_hash`则是输出两个commit的`git diff`内容。<br>
<br>
至此，`Analyzer#update_repositories`的调用堆栈大致清晰了。让我们回到`resolve_dependencies`方法，在更新完本地spec源仓库后，开始进行真正的`依赖分析`过程。

{% highlight ruby %}
    UI.section 'Analyzing dependencies' do
        analyze(analyzer)
        validate_build_configurations
        clean_sandbox
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
    * 检查是否需要更新pod source源
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



