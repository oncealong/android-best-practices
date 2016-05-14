> 原文链接：https://github.com/futurice/android-best-practices
> 本文是Futurice公司的Android开发人员总结的最佳实践，遵循这些准则可以避免重复制造轮子。如果你对iOS或者WindowsPhone开发感兴趣，那么也请看看[iOS最佳实践](https://github.com/futurice/ios-good-practices)和[Windows客户端开发最佳实践](https://github.com/futurice/win-client-dev-good-practices)。
> 第一版翻译自：http://blog.csdn.net/asce1885 Android开发技术日新月异, Github上也有较大更新, 故对原文有增删

###概要
1. [使用Gradle和推荐的工程结构](#build-system)
2. [把密码和敏感数据存放在gradle.properties文件中](#gradle-configuration)
3.  [使用Jackson或者Gson库来解析JSON数据](#libraries)
4. [不要自己实现HTTP客户端，要使用Volley或者OkHttp库](#networklibs)
5. [避免使用Guava, 使用少量的函数库从而避免超出65k方法数限制. `(注: Guava是Google基于java1.6的类库集合的扩展项目，包括 collections, caching, concurrency libraries, I/O等高质量的 API, 可以使JAVA代码更加优雅和简洁)`](#methodlimitation)
6. [使用Fragments来表示UI界面](activities-and-fragments)
7. [Activities只用来管理Fragments](#activities-and-fragments)
8. [布局XML文件是代码，要组织好它们](#resources)
9. [使用样式文件来避免布局XML文件中属性的重复定义](#styles)
10. [使用多个样式文件避免单一大样式文件的使用](#splitstyles)
11. [保持colors.xml文件简短和不重复，只定义颜色值](#colorsxml)
12. [保持dimens.xml文件不重复，并只定义通用的常量](#dimensxml)
13. [避免ViewGroups层次结构太深](#deephierarchy)
14. [避免在客户端侧处理WebViews，谨防内存泄漏](#webviews)
15. [使用Robolectric作为单元测试的工具，Robotium作为UI测试的工具](#test-frameworks)
16. [使用Genymotion作为你的模拟器](#emulators)
17. [总是使用ProGuard或者DexGuard](#proguard-configuration)
18. [使用SharedPreferences处理简单的数据持久化, 使用ContentProciders处理复杂的数据持久化](#data-storage)
19. [使用Stetho调试你的程序. `(Stetho是Facebook出品的一个强大的Android调试工具，使用该工具你可以在Chrome Developer Tools查看App的布局，网络请求，sqlite，preference，一切都是可视化的操作，无须自己在去使用adb，也不需要root你的设备。)`](#use-stetho)

###Android SDK
把你的Android SDK目录放在电脑的主目录或者其他跟IDE安装目录独立的磁盘位置，某些IDE在安装时就包含了Android SDK，而且可能把它放在跟IDE相同的目录下。当你需要升级（或重新安装）IDE，或者更换IDE时，这种做法是不好的。同样要避免把Android SDK放在另外一个系统层级的目录中，这样当你的IDE在user模式下运行而不是root模式时，将需要sudo权限。

<a name="build-system"></a>
###构建系统
你的默认选择应该是[Gradle](http://tools.android.com/tech-docs/new-build-system)。相比之下，Ant限制更大而且使用起来更繁琐。使用Gradle可以很简单的实现：
1）将你的app编译成不同的版本；
2）实现简单的类似脚本的任务；
3）管理和下载第三方依赖项；
4）自定义密钥库；
5）其他
Google也在积极的开发Android的Gradle插件，以此作为新的标准编译系统。

###工程结构
目前有两个流行的选择：以前的Ant和Eclipse ADT工程结构，以及新的Gradle和Android Studio工程结构。你应该选择新的工程结构，如果你的工程还在使用旧的结构，那么应该立即开始将它迁移到新的结构上面来。

旧的工程结构如下所示：
```
old-structure  
├─ assets  
├─ libs  
├─ res  
├─ src  
│  └─ com/futurice/project  
├─ AndroidManifest.xml  
├─ build.gradle  
├─ project.properties  
└─ proguard-rules.pro  
```
新的工程结构如下所示：
```
new-structure  
├─ library-foobar  
├─ app  
│  ├─ libs  
│  ├─ src  
│  │  ├─ androidTest  
│  │  │  └─ java  
│  │  │     └─ com/futurice/project  
│  │  └─ main  
│  │     ├─ java  
│  │     │  └─ com/futurice/project  
│  │     ├─ res  
│  │     └─ AndroidManifest.xml  
│  ├─ build.gradle  
│  └─ proguard-rules.pro  
├─ build.gradle  
└─ settings.gradle  
```
主要的区别在于新的结构明确的区分源码集合（`main`和`androidTest`），这是从Gradle引入的概念。例如，你可以在源码目录`src`中添加`paid`和`free`两个子目录，分别用来存放付费版和免费版app的源码。

顶层的`app`目录有助于把你的`app`和工程中会引用到的其他库工程（例如`library-foobar`）区分开。`settings.gradle`文件中记录了这些库工程的引用，这样`app/build.gradle`就能够引用到了。

<a name="gradle-configuration"></a>
###Gradle配置

**一般结构**：[参见Google的安卓Gradle指南](http://tools.android.com/tech-docs/new-build-system/user-guide)。
**小任务**：在Gradle中，我们使用tasks而不是脚本（shell，Python，Perl等使用脚本），详细的内容可参见[Gradle文档](http://www.gradle.org/docs/current/userguide/userguide_single.html#N10CBF)。
**密码**：发布release版本时，你需要在app目录下面的`build.gradle`文件中定义`signingConfigs`字段，下面这个配置会出现在版本控制系统中，这是你应该避免的：
```
signingConfigs {  
    release {  
        storeFile file("myapp.keystore")  
        storePassword "password123"  
        keyAlias "thekey"  
        keyPassword "password789"  
    }  
}  
```
你应该创建一个`gradle.properties`文件，该文件不要添加到版本控制系统中，并设置如下：
```
KEYSTORE_PASSWORD=password123  
KEY_PASSWORD=password789  
```
gradle会自动导入这个文件，现在你可以在build.gradle中这样使用：
```
signingConfigs {  
    release {  
        try {  
            storeFile file("myapp.keystore")  
            storePassword KEYSTORE_PASSWORD  
            keyAlias "thekey"  
            keyPassword KEY_PASSWORD  
        }  
        catch (ex) {  
            throw new InvalidUserDataException("You should define KEYSTORE_PASSWORD and KEY_PASSWORD in gradle.properties.")  
        }  
    }  
}  
```
**优先选择Maven依赖而不是导入jar文件。**如果你在工程中显式地包含jar文件，它们会是特定的不可变的版本，例如2.1.1。下载jar包并手动更新是很麻烦的，而这个问题Maven正好帮我们解决了，在Android Gradle构建中也建议这么做。例子如下：
```
dependencies {  
    compile 'com.squareup.okhttp:okhttp:2.2.0'
    compile 'com.squareup.okhttp:okhttp-urlconnection:2.2.0'
}  
```

**避免Maven的动态依赖解析** 避免使用动态版本号, 如`2.1.+`, 因为这可能会导致不同或者不稳定的构建, 也可能会导致不同构建中行为的微妙且难以追踪的差异. 使用静态版本号, 如`2.1.1`有助于创建一个更稳定, 更可预测, 可重现问题的开发环境.

**对不同的发布版本使用不同的打包名字** 对*debug*[构建类型](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Build-Types)使用`applicationIdSuffix`, 从而可以在相同的设备上安装*debug*和*release*的apk(如果你要自定义构建类型, 也这样尝试). 当app在商店发布后, 这会在app的生命周期中格外有价值.
```
android {
    buildTypes {
        debug {
            applicationIdSuffix '.debug'
            versionNameSuffix '-DEBUG'
        }

        release {
            // ...
        }
    }
}
```
使用不同的图标来区别不同的构建类型---例如使用不同的颜色, 或者用一个`debug`标签覆盖图标. Gradle使这些轻而易举: 在默认的project结构上, 只要把`debug`图标放在`app/src/debug/res`, 把`release`图标放在`app/src/release/res`. 你也可以[改变app名字](http://stackoverflow.com/questions/24785270/how-to-change-app-name-per-gradle-build-type)来对应不同的构建类型, 也可以使用上文提到的`versionName`.

###IDE和文本编辑器
**使用任何可以良好应对工程结构的代码编辑器。**代码编辑器是个人喜好的选择，你需要做的是保证你所用的编辑器能够和工程结构以及构建系统良好集成。

当下最受推崇的IDE是Android Studio，因为它是Google开发的，和Gradle耦合最好，默认使用最新的工程结构，已经处于稳定阶段，是为Android开发量身定做的IDE。

当然你也可以使用Eclipse ADT，但你需要配置它才能使用Gradle，因为它默认使用的是旧的工程结构和使用Ant进行构建。你甚至可以使用类似Vim，Sublime Text，Emacs等文本编辑器，这种情况下你需要在命令行中使用Gradle和adb。如果你的Eclipse集成Gradle不可用，你的选择是要么使用命令行编译或者把项目迁移到Android Studio中。Android Studio是最好的选择，因为ADT插件已经被标记为过时了，也就是不会再作后续维护和更新了。

无论你使用哪种方式，需保证的是按照官方的推荐使用新的工程结构和Gradle来构建你的应用，并避免把你特定于编辑器的配置文件加入到版本控制系统中。例如要避免把Ant的build.xml文件添加到版本控制系统中。特别是如果你在Ant中更改了编译配置，不要忘了同步更新`build.gradle`文件。最后一点，要对其他开发人员友好，不要迫使他们修改他们所用编辑器的偏好设置。

<a name="libraries"></a>
###函数库
[**Jackson**](http://wiki.fasterxml.com/JacksonHome)是一个把Java对象转换为JSON字符串或者把JSON字符串转换成Java对象的Java函数库。[Gson](https://github.com/google/gson)也是解决这类问题很流行的选择之一，但我们发现Jackson更加高性能，因为它支持多种可选的处理JSON的方式：流，内存树模型和传统的JSON-POJO数据绑定。尽管如此，Jackson是比Gson更大的函数库，所以需要根据你项目的具体情况，你可能会选择GSON来避免65k方法数限制。其他的选择还有：[Json-smart](http://netplex.github.io/json-smart/)和[Boon JSON](https://github.com/RichardHightower/boon/wiki/Boon-JSON-in-five-minutes)。
<a name="networklibs"></a>
**网络，缓存和图像。**向后端服务器发起网络请求有很多经过实战检验的解决方案，你应该使用这些解决方案而不是自己实现一个。使用[Volley](https://android.googlesource.com/platform/frameworks/volley)或者[Retrofit](http://square.github.io/retrofit/)吧！除了网络请求，Volley还提供了帮助类用于加载和缓存图像。如果你选择Retrofit，那么可以考虑使用[Picasso](http://square.github.io/picasso/)作为加载和缓存图像的函数库，并结合[OkHttp](http://square.github.io/okhttp/)实现高效的HTTP请求。Retrofit，Picasso和OkHttp这三款函数库都是同一家公司开发的，所以它们能够很好的互补。[Volley也能使用OkHttp来实现网络连接](http://stackoverflow.com/questions/24375043/how-to-implement-android-volley-with-okhttp-2-0/24951835#24951835)。

**RxJava**是一个响应式编程的函数库，也就是可以处理异步事件。这是一个强大和有前途的编程范式，但由于它是如此的不同，因此会显得不好理解。在使用这个函数库搭建你的应用的框架时，我们建议你要保持谨慎的态度。我们有几个项目已经使用RxJava来实现，如果你需要帮助可以联系以下这些人：Timo Tuominen, Olli Salonen, Andre Medeiros, Mark Voit, Antti Lammi, Vera Izrailit, Juha Ristolainen。我们已经写了一些博客文章来进行介绍
1. http://blog.futurice.com/tech-pick-of-the-week-rx-for-net-and-rxjava-for-android；
2. http://blog.futurice.com/top-7-tips-for-rxjava-on-android；
3. https://gist.github.com/staltz/868e7e9bc2a7b8c1f754；
4. http://blog.futurice.com/android-development-has-its-own-swift。

如果你之前没有使用RxJava的经验，那么开始时可以仅在网络请求API的响应处使用。如果有经验了，可以将RxJava应用在简单UI事件的处理，例如点击事件或者搜索框中的输入事件。如果你对自己的RxJava技能很自信而且想把RxJava应用到整个项目架构中，那么在代码难以理解的部分要编写Javadocs。要记住对RxJava不熟悉的程序员可能在维护工程的初期会很痛苦。尽你所能帮助他们理解你的代码和RxJava。

[**Retrolambda**](https://github.com/evant/gradle-retrolambda)是兼容在Android中和JDK8之前的Java版本中使用Lambda表达式语法的一个Java函数库。它帮助你的代码保持紧凑和可读，特别是当你使用函数式编程风格时，例如使用RxJava。要使用这个库，需要安装JDK8，在Android Studio工程结构对话框中设置SDK的位置，并设置`JAVA8_HOME`和`JAVA7_HOME`环境变量，然后在工程的build.gradle中设置如下：
```
dependencies {  
    classpath 'me.tatarka:gradle-retrolambda:2.4.+'  
}  
```
接着在各个模块的build.gradle中增加配置如下：
```
apply plugin: 'retrolambda'  
  
android {  
    compileOptions {  
    sourceCompatibility JavaVersion.VERSION_1_8  
    targetCompatibility JavaVersion.VERSION_1_8  
}  
  
retrolambda {  
    jdk System.getenv("JAVA8_HOME")  
    oldJdk System.getenv("JAVA7_HOME")  
    javaVersion JavaVersion.VERSION_1_7  
}  
```
Android Studio提供了支持Java8 lambdas的代码辅助功能。如果你刚接触lambdas，可以参见下面的建议来开始：
1）任何只有一个函数的接口（Interface）是“lambda友好”的，可以被折叠为更紧凑的语法格式；
2）如果对参数或诸如此类的用法还存有怀疑的话，可以编写一个普通的匿名内部类然后让Android Studio帮你把它折叠成lambda表达式的形式。
<a name="methodlimitation"></a>
**要注意dex文件方法数限制的问题，避免使用太多的第三方函数库。**当Android应用打包成一个dex文件时，存在最多65536个引用方法数的限制问题。当超出这个限制时，你将在编译阶段看到fatal error。因此，应该使用尽可能少的第三方函数库，并使用[dex-method-counts](https://github.com/mihaip/dex-method-counts)工具来决定使用哪些函数库的组合以避免不超过该限制。特别要避免使用Guava函数库，因为它包含的方法数超过13k。(`注:现在已经有了方法来解决这个问题, 比如multidex`)
<a name="activities-and-fragments"></a>
###Activities and Fragments
> [`注:因为对activity和fragment优劣的比较一直在持续, 这里和以前的版本改动较大, 但是这里的结论仍不一定是最优结论. 读者需要辩证的看待, 所以先给出最新的翻译, 并把以前的翻译保留, 以便读者辩证思考.`]

关于如何使用Activity和Fragment来最好的组织Android架构, 开发社区和Futurice公司的开发者都还没有一致的看法. Squrare公司甚至有[一个基于Views架构的库](https://github.com/square/mortar), 这直接绕过了Fragments, 但是这种做法在开发社区不值得大范围推广使用.

由于Android API的历史, Fragments可以粗略的认为用来组装成UI. 或者说, Fragments 一般都是和UI相关的.  Activities可以粗略的认为是控制器, 它们用于管理生命周期和状态. 然而, 你可能会看到这些角色各有变种, activities可能用来承担UI角色([提供画面之间的转换](https://developer.android.com/about/versions/lollipop.html)), [fragments可能作为控制器单独使用](http://developer.android.com/guide/components/fragments.html#AddingWithoutUI). 我们只能建议小心使用, 采取明智的决策, 因为fragments-only架构 或者 activities-only架构或者views-only架构都有缺点.下面是一些关于使用时小心什么的建议, 但是还请你们持怀疑态度看待这些想法.

 - 避免大范围的使用[嵌套fragments](https://developer.android.com/about/versions/android-4.2.html#NestedFragments), 这会导致[matryoshka bugs](http://delyan.me/android-s-matryoshka-problem/)。只有在必要的时候（例如在Fragment屏幕中内嵌水平滑动的ViewPager）或者确实是明智的决定时才使用嵌套Fragments。
 - 避免在activities中放太多代码. 只要有可能, 尽量把它们作为轻量级容器, 主要用来负责生命周期的控制和其他重要的Android接口API. 优先选择 single-fragment activity, 把UI相关的代码放在activity的fragment中. 当你需要改变这个fragment让它驻留在分页布局或平板布局时, 这让代码可重用性提高. 避免使用没有对应fragment的activity, 除非你知道你在做一个英明决策.
 - 不要滥用Android系统级别的API, 如严重依赖Intent来进行app内部工作. 这样可能会影响Android系统或其他的应用, 造成bug或者反应滞后. 例如, 众所周知, 如果你的app使用Intent在不同包中进行通信, 当系统重启后, 打开app可能会引发multi-second 滞后. 

>`以前的看法:`
> 在Android中实现UI界面默认应该选择Fragments。Fragments是可复用用户界面，能够在你的应用中很好的组合。我们建议使用Fragments代替Activities来表示用户界面，下面是几点原因：
> 1）多窗口布局的解决方案。最初引入Fragment的原因是为了把手机应用适配到平板电脑屏幕上，这样你可以在平板电脑屏幕上具有A和B两个窗口，而到了手机屏幕上A或者B窗口独占整个手机屏幕。如果你的应用在最开始的时候是基于Fragments实现的，那么以后要适配到其他不同类型的屏幕上面会容易得多。
> 2）不同界面之间的通信。Android
> API没有提供在Activity之间传递复杂数据（例如某些Java对象）的正确方法。对于Fragments而已，你可以使用Activity的实例作为它的子Fragments之间通信的通道。虽然这种方法比Activities之间的通信更好，但你可能愿意考虑Event
> Bus框架，例如使用Otto或者green robot
> EventBus作为更简洁的方法。如果你想避免多添加一个函数库的话，RxJava也可以用于实现Event Bus。
> 3）Fragments足够通用，它不一定需要UI界面。你可以使用没有UI界面的Fragment来完成Activity的后台工作。你可以进一步拓展这个功能，创建一个专门的后台Fragment来管理Activity的子Fragments之间的切换逻辑，这样就不用在Activity中实现这些逻辑了。
> 4）在Fragments里面也可以管理ActionBar。你可以选择创建一个没有UI界面的Fragment专门来管理ActionBar，或者选择让每个Fragments在宿主Activity的ActionBar中添加它们自己的action
> items。更多请参考http://www.grokkingandroid.com/adding-action-items-from-within-fragments/。
> 我们建议不要大量的使用嵌套Fragments，这会导致matryoshka
> bugs。只有在必要的时候（例如在Fragment屏幕中内嵌水平滑动的ViewPager）或者确实是明智的决定时才使用嵌套Fragments。
> 从软件架构的层面看，你的app应该具有一个包含大部分业务相关fragments的顶级activity。当然你也可以有其他辅助activities，这些activities与主activity的具有简单的数据通信，例如通过Intent.setData()或者Intent.setAction()等等。

###Java包结构
Android应用的Java包结构大致类似MVC模式。在Android中[Fragment和Activity实际上是controller类](http://www.informit.com/articles/article.aspx?p=2126865)，另一方面，它们显然也是用户界面的一部分，因此也是View类。

因此，很难严格界定Fragments（或者Activities）是controllers类还是views类。最好把Fragments类单独放在fragments包里面。如果你遵循前面段落的建议的话（只有一个主Activity），Activities可以放在顶层的包里面。如果你计划会存在多个activities，那么就将Activity放在单独的activities包里面。

另一方面，整个包结构看起来很像经典的MVC框架，models包目录存放网络请求API响应经过JSON解析器填充后得到的POJOs对象。views包目录存放自定义的views，notifications，action bar views，widgets等等。Adapters处于灰色地带，介于data和views之间，然而，它们一般需要通过getView()函数导出视图，所以你可以在views包中增加adapters子包。

有的controller类是应用范围的并和Android系统紧密关联，它们可以放在managers包里面。其他的数据处理类，例如“DateUtils”，可以放在utils包里面。与服务器端响应交互的类放在network包里面。
总之，整个包结构从服务器端到用户界面划分如下所示：
```
com.futurice.project  
├─ network  
├─ models  
├─ managers  
├─ utils  
├─ fragments  
└─ views  
   ├─ adapters  
   ├─ actionbar  
   ├─ widgets  
   └─ notifications  
```
<a name="resources"></a>
###资源
命名。遵循以类型作为前缀命名的惯例，即`type_foo_bar.xml`。例子如下：`fragment_contact_details.xml`，`view_primary_button.xml`，`activity_main.xml`。
组织布局XMLs。如果你对如何格式化布局XML文件不大清楚的话，那么下面的惯例或许可以帮你：
1）每行一个属性，缩进四个空格
2）`android:id`始终作为第一个属性
3）`android:layout_****`属性放在头部
4）`style`属性放在尾部
5）结束标签`/>`放在单独一行，便于新增属性或者重新排列属性
6）考虑使用Android Studio中的[Designtime attributes](http://tools.android.com/tips/layout-designtime-attributes)，而不使用android:text硬编码。
```
<?xml version="1.0" encoding="utf-8"?>  
<LinearLayout  
    xmlns:android="http://schemas.android.com/apk/res/android"  
    xmlns:tools="http://schemas.android.com/tools"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:orientation="vertical"  
    >  
  
    <TextView  
        android:id="@+id/name"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        android:layout_alignParentRight="true"  
        android:text="@string/name"  
        style="@style/FancyText"  
        />  
  
    <include layout="@layout/reusable_part" />  
  
</LinearLayout>  
```
一般来说`android:layout_****`属性应该定义在XML布局文件中，而其他的`android:****`属性应该放在样式xml文件中。这条法则也有例外，但一般情况是这样的。这种做法是为了在布局文件中只保留布局属性（位置，留白，大小）和内容属性，而其他外观细节（颜色，填充，字体）放到样式文件中。
例外的有：
- `android:id`必须放在layout文件中
- 在`LinearLayout`中的`android:orientation`一般放在layout文件中更有意思
- `android:text`必须放在layout文件中，因为它定义了内容
- 有时创建通用的style文件并定义`android:layout_width`和`android:layout_height`属性更有意义，但默认情况下这两个属性应该放在layout文件中。
<a name="styles"></a>
**使用styles。**几乎所有工程都需要正确的使用styles，因为它是让view具有相同外观的常见的方法。你的应用的文本内容应该至少具有一个公用的样式，例如：
```
<style name="ContentText">  
    <item name="android:textSize">@dimen/font_normal</item>  
    <item name="android:textColor">@color/basic_black</item>  
</style>  
```
用在TextView上面如下：
```
<TextView  
    android:layout_width="wrap_content"  
    android:layout_height="wrap_content"  
    android:text="@string/price"  
    style="@style/ContentText"  
    />  
```
你可能需要对buttons做类似的工作，但别就此停住。继续把相关的和重复的`android:****`属性分组到公用的style文件中。
<a name="splitstyles"></a>
**把一个大的style文件细分成多个文件。**你没有必要维护单独一个`styles.xml`文件，Android SDK能很好的支持其他文件。styles文件的名字并不一定要是`styles.xml`，起作用的是xml文件里面的`<style>`标签。因此，你的样式文件命名可以是`styles.xml`，`styles_home.xml`，`styles_item_details.xml`，`styles_forms.xml`等。和资源目录名不同（编译系统需要根据资源目录名找到资源），`res/values`里面的文件名可以随意命名。
<a name="colorsxml"></a>
**`colors.xml`是调色板。**在你的colors.xml文件中除了定义颜色名字到RGBA颜色值的映射外，不应该定义其他的东西。不用使用它来定义不同类型按钮的RGBA颜色值。
不要这样做：
```
<resources>  
    <color name="button_foreground">#FFFFFF</color>  
    <color name="button_background">#2A91BD</color>  
    <color name="comment_background_inactive">#5F5F5F</color>  
    <color name="comment_background_active">#939393</color>  
    <color name="comment_foreground">#FFFFFF</color>  
    <color name="comment_foreground_important">#FF9D2F</color>  
    ...  
    <color name="comment_shadow">#323232</color>  
```
这样的使用很容易重复定义相同的RGBA值，这导致如果需要更改一个基本色值时会很麻烦。而且上面的这些定义是和上下文相关的，例如“button”，“comment”，这些应该放到button样式文件中，而不是`colors.xml`文件中。
应该这样做：
```
<resources>  
  
    <!-- grayscale -->  
    <color name="white"     >#FFFFFF</color>  
    <color name="gray_light">#DBDBDB</color>  
    <color name="gray"      >#939393</color>  
    <color name="gray_dark" >#5F5F5F</color>  
    <color name="black"     >#323232</color>  
  
    <!-- basic colors -->  
    <color name="green">#27D34D</color>  
    <color name="blue">#2A91BD</color>  
    <color name="orange">#FF9D2F</color>  
    <color name="red">#FF432F</color>  
  
</resources>  
```
向应用的设计师要以上这些色值定义。命名不需要为颜色名字，如“green”，“blue”等，例如“brand_primary”，“brand_secondary”，“brand_negative”这样的命名也是完全可以接受的。这样来格式化颜色值使得以后如果要修改或者重构颜色时很容易，同时应用中使用了多少种颜色也是一目了然的。对于一个美观的UI，减少使用的颜色种类是很重要的。
<a name="dimensxml"></a>
**dimens.xml文件跟colors.xml文件具有相同的用法。**你应该定义一个典型的间距和字体大小的模版，目的基本上和colors.xml文件一样，好的dimens.xml文件例子如下：
```
<resources>  
  
    <!-- font sizes -->  
    <dimen name="font_larger">22sp</dimen>  
    <dimen name="font_large">18sp</dimen>  
    <dimen name="font_normal">15sp</dimen>  
    <dimen name="font_small">12sp</dimen>  
  
    <!-- typical spacing between two views -->  
    <dimen name="spacing_huge">40dp</dimen>  
    <dimen name="spacing_large">24dp</dimen>  
    <dimen name="spacing_normal">14dp</dimen>  
    <dimen name="spacing_small">10dp</dimen>  
    <dimen name="spacing_tiny">4dp</dimen>  
  
    <!-- typical sizes of views -->  
    <dimen name="button_height_tall">60dp</dimen>  
    <dimen name="button_height_normal">40dp</dimen>  
    <dimen name="button_height_short">32dp</dimen>  
  
</resources>  
```
布局文件的边距和填充应该使用`spacing_****`尺寸定义，而不是使用硬编码（类似字符串硬编码）。这样会带来统一的外观，同时使得组织和修改样式和布局更简单。

**strings.xml**
用像命名空间的文字来给你的strings命名, 不要害怕几个不同的名字有重复的值. 语言如此复杂,  你需要命名空间打破含糊不清的表达方式.
Bad:
```
<string name="network_error">Network error</string>
<string name="call_failed">Call failed</string>
<string name="map_failed">Map loading failed</string>
```
Good:
```
<string name="error.message.network">Network error</string>
<string name="error.message.call">Call failed</string>
<string name="error.message.map">Map loading failed</string>
```
不要用全大写的方式. 坚持普通文本约定(比如首字母大写). 如果你需要用全部大写的方式显示文本, 设置TextView的`textAllCaps`属性.
Bad:
```
<string name="error.message.call">CALL FAILED</string>
```
Good:
```
<string name="error.message.call">Call failed</string>
```
<a name="resources"></a>
**避免过深的views层级。**有时你可能会被诱导在LinearLayout中再增加一层LinearLayout，例如为了完成一组views的排列。这种情况类似如下：
```
<LinearLayout  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    android:orientation="vertical"  
    >  
  
    <RelativeLayout  
        ...  
        >  
  
        <LinearLayout  
            ...  
            >  
  
            <LinearLayout  
                ...  
                >  
  
                <LinearLayout  
                    ...  
                    >  
                </LinearLayout>  
  
            </LinearLayout>  
  
        </LinearLayout>  
  
    </RelativeLayout>  
  
</LinearLayout>  
```
即使你没有很明显的在Layout文件中看到这种情况，但可能最终发生在Java代码中将某个view inflate到另外的view中。
这可能会引起一些问题，你可能会遇到性能问题，因为处理器需要处理很复杂的UI树层级。另一个更严重的问题是可能会引起[StackOverflowError](http://stackoverflow.com/questions/2762924/java-lang-stackoverflow-error-suspected-too-many-views)。
因此，尽量使你的view具有扁平的层级：学习怎样使用[RelativeLayout](https://developer.android.com/guide/topics/ui/layout/relative.html)，怎样[优化布局](http://developer.android.com/training/improving-layouts/optimizing-layout.html)和使用[`<merge>`标签](http://stackoverflow.com/questions/8834898/what-is-the-purpose-of-androids-merge-tag-in-xml-layouts)。
<a name="webviews"></a>
**小心WebView相关的问题。**当你需要展示一个web页面时，例如新闻文章，要避免在客户端侧对HTML进行简化处理，相反，应该向服务器端请求经过简化后的HTML。当WebView持有所在Activity Context引用而不是Application Context引用时，也可能[因WebView导致内存泄漏](http://stackoverflow.com/questions/3130654/memory-leak-in-webview)。要避免在简单文本或者按钮使用WebView，应该使用TextView和Button。
<a name="test-frameworks"></a>
###测试框架
Android SDK的测试框架还很简单，尤其是UI测试相关的。Android Gradle目前实现了一个叫做`connectedAndroidTest`的测试任务，它能够使用[JUnit的Android扩展](http://developer.android.com/reference/android/test/package-summary.html)来运行你创建的JUnit测试用例。这意味着你需要连接设备或者模拟器来运行测试用例，遵循官方帮助指南来进行测试1）http://developer.android.com/tools/testing/testing_android.html；
2）http://developer.android.com/tools/testing/activity_test.html）。

**使用[Robolectric](http://robolectric.org/)来进行单元测试，而不是UI测试。**这是一个为了提高开发速度，专注于提供“独立于设备”的测试框架，尤其适用于models和view models的单元测试。但是Robolectric对于UI测试的支持是不准确和不完善的。使用Robolectric进行动画，对话框等相关的UI元素测试时会遇到问题，你将看不到屏幕相应的UI元素被测试实时操纵，你将类似于在黑暗中行走。

**[Robotium](https://code.google.com/p/robotium/)简化了UI测试。**你不需要Robotium来执行UI测试用例，但它提供了很多帮助工具用来获取和分析views，控制屏幕等，这一点对你可能很有帮助。测试用例很简单，如下所示：
```
solo.sendKey(Solo.MENU);  
solo.clickOnText("More"); // searches for the first occurence of "More" and clicks on it  
solo.clickOnText("Preferences");  
solo.clickOnText("Edit File Extensions");  
Assert.assertTrue(solo.searchText("rtf"));  
```
<a name="emulators"></a>
###模拟器
如果你的工作是开发android app，那么买一个Genymotion模拟器的licence吧。Genymotion模拟器比AVD模拟器具有更快的帧率，而且具有演示app，模拟网络连接质量，GPS位置等工具。它也是用于连接测试的理想工具。使用Genymotion模拟器，你可以模拟很多不同类型的设备，所以购买一个Genymotion模拟器licence比买多个真实设备更划算。
要注意的是：Genymotion模拟器没有移植所有的Google服务，例如Google Play Stoe和Google Maps。你可能需要测试三星特有的API，这时需要购买一台真实的三星设备。

<a name="proguard-configuration"></a>
###Proguard配置
在Android工程中[ProGuard](http://proguard.sourceforge.net/)被用于压缩和混淆打包后的代码。ProGuard的使用可以在工程配置文件中设置。一般情况下当构建一个release版本的apk时，你需要配置Gradle使用ProGuard。
```
buildTypes {  
    debug {  
        minifyEnabled false  
    }  
    release {  
        signingConfig signingConfigs.release  
        minifyEnabled true  
        proguardFiles 'proguard-rules.pro'  
    }  
}  
```
为了决定哪些代码需要保留，哪些代码需要丢弃或者混淆，你需要在你的代码中指定一个或者多个入口点。这些入口点典型的就是具有main函数，applets，midlets，activities等的类。Android框架使用的默认配置文件是`SDK_HOME/tools/proguard/proguard-android.txt`。自定义的工程特有的proguard规则文件定义为`my-project/app/proguard-rules.pro`，将会拼接到默认配置中。

Proguard相关的一个常见问题是在应用启动时出现crash，错误类型是`ClassNotFoundException`或者`NoSuchFieldException`等，即使编译是成功的，原因不过如下两种：
1）ProGuard把需要用到的类，枚举，方法，变量或者注解等给移除了；
2）ProGuard把相应的类，枚举或者变量名给混淆（重命名）了，但调用者还是使用它原来的名字，例如Java反射的情况。

检查`app/build/outputs/proguard/release/usage.txt`文件看出问题的对象是否被移除了；检查`app/build/outputs/proguard/release/mapping.txt`文件看出问题的对象是否被混淆了。为了防止ProGuard把需要的类或者类成员移除了，需要在ProGuard配置文件中增加keep选项：
```
-keep class com.futurice.project.MyClass { *; }  
```
为了防止ProGuard把需要的类或者类变量混淆了，要增加keepnames选项：
```
-keepnames class com.futurice.project.MyClass { *; }  
```
一些例子可以从[ProGuard配置模版](https://github.com/futurice/android-best-practices/blob/master/templates/rx-architecture/app/proguard-rules.pro)上面找到，更多例子参见[ProGuard官方例子](http://proguard.sourceforge.net/#manual/examples.html)。

**在你的项目早期，执行一个release构建来检查ProGuard规则是否正确的保持了不需要移除或者混淆的东西。**当你增加新的函数库，也要执行新的Release构建并在设备上测试生成的apk来确保没有问题。不要等到你的app要发布1.0版本了才想到要执行一个release构建，这时你可能会得到及其不愉快的惊喜，并花一段时间来修复这些问题。

**贴士：**保存每个你发布给最终用户的apk包对应的mapping.txt文件。保存mapping.txt的原因在于当你的用户上传混淆过的crash日志时，你可以很容易的进行调试。

**DexGuard：**如果你需要能对你发布的代码进行优化，尤其是混淆的核心工具的话，可以考虑[DexGuard](http://www.saikoa.com/dexguard)，这是由ProGuard同一团队发布的商业软件。它还可以很容易的对分割Dex文件以解决65k函数个数限制问题。

 <a name="data-storage"></a>
###数据存储
####SharedPreferences
如果你需要保存简单的标志, 并且你的程序是单进程的, SharedPreferences就足够使用了. 这是个很好的默认选择.

当如下情况时, 你可能不想使用SharedPreferences:
- 性能: 你的数据很复杂,或者数据量很大.
- 多个进程可以访问数据: 你有窗体小部件或者远程服务, 它们运行在自己的进程中, 并且需要同步数据.

####ContentProviders
当SharedPreferences满足不了你, 你应该考虑使用平台标准ContentProviders, 它更快并且是进程安全的.

使用ContentProviders唯一的问题是有大量的样板文件代码来建立ContentProviders, 但是没有好的入门指南. 但是通过使用库,如[Schematic](https://github.com/SimonVT/schematic), 是可以直接生成ContentProviders的, 这大大降低了工作成本.

你还需要亲自写一些解析代码来从Sqlite的列中读取数据, 或存到Sqlite中. 可以使用Gson来序列化数据对象, 并且只保留结果字符串. 通过这种方式, 你损失了一些性能, 但是你不必对实体类的所有域声明一行以读取它.
####使用ORM框架
我们一般不推荐使用对象关系映射库, 除非你有非一般复杂的数据, 并且你迫切需求ORM框架的帮助. 使用ORM会让代码变得复杂, 并且需要学习如何使用. 如果你确定需要使用ORM, 你应该关注它是否是进程安全的, 令人惊讶的是, 很多已有的ORM解决方案不是进程安全的.
 <a name="use-stetho"></a>
###使用Stetho
[Stetho](http://facebook.github.io/stetho/)是Facebook发布的Android 应用调试桥, 和Chrome桌面浏览器的开发者工具集成使用. 通过Stetho, 你可以轻松检查你的应用, 尤其是网络方面的问题. Stetho也让你更方便的检查,编辑SQLite数据库和shared preferences. 但是, 你应该确保Stetho只会在debug模式下启用, 不要存在于release版本中. 

####致谢
Antti Lammi, Joni Karppinen, Peter Tackage, Timo Tuominen, Vera Izrailit, Vihtori Mäntylä, Mark Voit, Andre Medeiros, Paul Houghton and other Futurice developers for sharing their knowledge on Android development.

###License
Futurice Oy Creative Commons Attribution 4.0 International (CC BY 4.0)
