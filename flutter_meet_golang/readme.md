# 当flutter遇上golang(2020/11/10)


前一段时间偶然发现一个有趣的golang库:[primitive](https://github.com/fogleman/primitive),

它能将一张普通的照片转换成颇具艺术风格的照片，例如：

![](my_avatar.png)

转换后的照片：

![](my_avatar_100.png)

还可以调不同的参数来实现不同的效果。

这个库让我沉迷了一晚上，简直不能自拔，当时我在想，要是有个手机app多好，能随时随地愉快的玩耍，还能和朋友分享。

之前简单学习了flutter，所以打算用flutter来写一个玩玩。

在调研了一番后，发现flutter调用golang库是可行的，于是怀揣着极大的热情开始探索了，想想都激动。

可是整个过程是痛苦不堪的，我差点就放弃了，要不是兴趣驱动，可能真的就无缘了。

因为我比较懒，动手能力差，最讨厌折腾这些环境、参数配置的事情了，我只想安静愉快的写代码。

不过通过这次折腾，也总结了一些经验，那就是专注、不要浮躁和急于求成、解决问题之前先弄清楚背景知识。

这个app花的时间应该能凑个2天整，其中写代码的时间不到2个小时，剩下的时间就是google问题，尝试解决我完全摸不着头脑的问题。。

另外吐槽下，android studio（4.1）在mac上（其他的不清楚）简直弱爆了，几乎达到不可使用的地步，动不动就会无响应，然后等5、6分钟才恢复正常，这样极大影响了我的效率，

每当我尝试新的做法的时候，就会等很久。


## 环境准备

我的电脑： MacBook Pro (Retina, 15-inch, Mid 2015)

go 版本： go version go1.15.3 darwin/amd64

安装flutter 我当前是更新的最新版本 1.24.0-8.0.pre.96。

安装 android studio 我的版本是4.1

安装 android sdk （我忘了以前怎么装得了，好像也是通过android studio装的）

android studio 安装 ndk。
路径: Preferences-> Apperance & Behavior -> System Settings -> Android SDK -> SDK Tools

android 安装 flutter, dart, kotlin插件


## gomobile

如何让flutter调用golang库呢，go有个工具gomobile，可以把golang库生成对应平台可调用的包（我用的安卓，这里只讲安卓，不想折腾）。

因为gomobile要对不同平台暴露接口，它只支持[有限的数据结构](https://godoc.org/golang.org/x/mobile/cmd/gobind#hdr-Type_restrictions)

生成命令：

```shell script
gomobile bind -v -target android -work github.com/fogleman/primitive/primitive
```
生成之前要设置安卓sdk路径的环境变量: `ANDROID_HOME=/Users/echo/Library/Android/sdk`

这里有个坑，gomobile会默认在设置的sdk目录下找ndk-bundle目录，可是我安装的ndk目录是ndk，所以这里我
创建了个软连接: `ln -s ndk/21.3.6528147 ndk-bundle`，`21.3.6528147` 是版本号，你的版本号可能跟我不一样。

过了小坑，前方还有个大坑：


现在go最新版本都1.16了，go module也理所当然会被使用了。

我试了设置GO11MODULE=off，折腾了很久没成功。下面讲的都是使用gomodule的情况。

执行gomobile bind的时候，如果我们不在$GOPATH/src/***下，会报错，说 `no exported names in the package "github.com/fogleman/primitive/primitive"`

'***'为$GOPATH/src/下任何一个目录。这个现象让人大吃一斤，不禁让人脱口而出一句话： 什么jb玩意儿啊啊啊啊

所以我进了`github.com/fogleman/primitive`,再执行，终于，报的错终于长得不一样了，谢天谢地！


很自然地，我发现这个golang库不支持gomobile 绑定，部分报错如下：

```text
gobind/go_primitivemain.go:376:17: cannot use (*proxyprimitive_Shape)(_param_shape_ref) (type *proxyprimitive_Shape) as type primitive.Shape in assignment:
	*proxyprimitive_Shape does not implement primitive.Shape (missing Draw method)
gobind/go_primitivemain.go:1172:7: cannot use (*proxyprimitive_Shape)(_v_ref) (type *proxyprimitive_Shape) as type primitive.Shape in assignment:
	*proxyprimitive_Shape does not implement primitive.Shape (missing Draw method)
```

于是我想着封装了一下，只暴露了一个最简单的接口：

```go
func ConvertImg(path string, outputPath string) (string, error) {****}
```

这里只是为了验证可行性，原库的大量参数去掉了，转为固定值。

然后再 gomobile bind这个封装后的方法，果然成功了。

生成了两个文件： `wrapper-sources.jar`, `wrapper.aar`。


到这里，9*9=81难 应该过了有1/3了。

接下来开始折腾flutter。

## flutter and android studio

flutter创建plugin命令：

`flutter create --org com.echo --template=plugin --platforms android hello`

会生成目录，一级目录如下：


```text
├── CHANGELOG.md
├── LICENSE
├── README.md
├── android
├── example
├── lib
├── hello.iml
├── pubspec.lock
├── pubspec.yaml
└── test
```

example是为了演示这个库怎么使用的，我直接拿来当app使用了。

生成的android目录是用来使用kotlin或者java来实现flutter plugin的。

当我满心欢喜踏上新的征程的时候，发现android目录下的kotlin代码永远找不到那些import的库。

各种查资料，各种试。

难道使用flutter命令创建的项目有问题？(实际上没问题)

我用android studio试试吧，可是按照网上说的 File -> New -> New Flutter Project, 居然没有 `New Flutter Project`选项，

于是又找了资料搞定，nice。

当我点击了 `New Flutter Project`的时候，android studio卡住了，这又是什么神仙问题。

后来我习以为常了，就等它卡就行了，过一会会自己醒过来的，它会假装什么都没发生，继续下一步。

可是还是找不到那些import 的库，网上找的资料都试过了，还是不行，于是自己摸索乱点，感觉是不是把项目点乱套了，又重新创建项目，再次等待卡住醒过来。

后来我就按照flutter官方文档从一个demo项目开始慢慢一步一个脚印跟着操作，结果是要点击目录下的 `android`目录右键，然后点击: flutter -> Open Anroid

Module in Android Studio，然后在新的窗口下编辑android项目，等待它卡几分钟，就好了。

然后导入我们生成的 .aar模块：

file -> New -> New Module,

等待它卡4分钟，选择 import .jar/.aar package，选择之前用gomobile生成的.aar文件。我的叫wrapper.aar(乱取的名字)

编辑 android.hello 的build.gradle: 在dependencies 下添加：

```gradle
    implementation project(":wrapper")
```

再编辑 settings.gradle， 添加：


```gradle

include ':app', ':wrapper'
```

这个时候会弹出窗口说您的gradle发生了变化，点击 `Sync Now`。

再去编辑kotlin文件,更改如下部分：

```kotlin
    // 添加import
    import wrapper.Wrapper;
    。。。
  override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
    if (call.method == "getPlatformVersion") {
      result.success("Android ${android.os.Build.VERSION.RELEASE}")
    } else if (call.method == "convertImg"){
      val path = call.argument<String>("input")
      val outPutPath = call.argument<String>("outputPath")
        try{
          result.success(Wrapper.convertImg(path, outPutPath))
        }catch (e: IllegalArgumentException){
          result.error("BAD_ARGS", e.message!!, null)
        }catch (e:Exception){
          result.error("NATIVE_ERR", e.message!!,null)
        }
    } else {
      result.notImplemented()
    }
  }
```
到这里 android的任务完成了，接下来就轮到flutter和dart的表演了。

进入 hello/lib/hello.dart,给类添加方法：

```dart
  static Future convertImg(Map<String, String> args) async {
    await _channel.invokeMethod('convertImg', args);
  }
```

最终我们使用的时候就直接调用这个方法就ok了。

界面部分就不细说了，相信你们肯定比我玩的溜。

看下效果图吧：

打开app的样子：

![](./flutter1.png)

点击右下角选择图标按钮：

![](./flutter2.png)

展示选择的图片：

![](./flutter3.jpeg)

点击右下角图标按钮，开始生成艺术照：

![](./flutter4.jpeg)

只不过这里又有一个巨坑，一个无法逾越的坑：

当我选择文件开始生成艺术图片的时候，ui就完全卡住了，我设置的圈圈不仅不转，都不出场。。。

那个golang生成图片是cpu密集操作，它和ui在同一个线程，当cpu满转的时候，ui就完全卡住了，导致用户体验极差，

点击生成按钮后，看不到点击的效果，以为没点上呢，然后就是整个app僵住，等待一会突然展示生成的图。


后来搜到isolate，可以将cpu密集操作在新的线程执行，不至于消耗ui线程。

可是鼓捣了半天，研究了半天，发现flutter不支持将methodchannel调用的方法在单独线程执行，Game Over。

就这样吧，累了。







