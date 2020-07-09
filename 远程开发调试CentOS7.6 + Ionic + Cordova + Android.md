最近项目要写个Android程序，我是打算用Ionic + Cordova + Android 混合开发。主要是想学下前端界面怎样写可以快点和不想用java写android。本片大概会讲完如何用`centos7.6搭一个Ionic远程开发环境`和`远程真机调试Android程序`和`VSCode远程开发`。

本文主要参考ionic的官方文档**https://ionicframework.com/docs**做配置，并且记录一些搭建过程中会遇到的问。

## 搭建CentOS7.6 Ionix 环境	

### 安装nodejs 

参考我之前的安装把**https://www.cnblogs.com/harmful-chan/p/12420720.html**linux的配置都差成差不多，解压好然后配置完环境变量之后`node -v`有输出就行，推荐用长支持版，后面有TLS那种。原文*https://ionicframework.com/docs/intro/environment*

### nodejs 安装 ionic及启动一个最简单工程

运行`npm install -g @ionic/cli`全局安装ionic脚手架工具，全局安装路径在$(NODEJS_HOME)/lib/node_modules下。

然后新建工程`ionic start myApp tabs`运行他会提示是用Angular或vue，vue不熟我是用angular的，选择好之后他会帮你把angular的文件装到myApp/node_modules下，看网速可能有点慢，然后其实就可以`cd myApp && ionic serve --host 0.0.0.0`运行项目，--host是为了指定监听所有网卡进来的请求，默认只能监听localhost的，外网不能访问。这一步一般问题不大，建议把iptables都清空。

原文**https://ionicframework.com/docs/intro/cli**

### ionic远程调试真机设备

ionic混合开发其实是通过capacitor或者cordova这个组件来跟android sdk 进行交互从而可以支持angular或者vue的语法，所以首先我们需要在centos7.6上安装android sdk并远程接入我们的安卓手机 。原文**https://ionicframework.com/docs/developing/android#cordova-setup**

#### 安装open jdk

在centos7.6上epel源维护这我们需要的openjdk`yum list | grep openjdk`可以查看是否有这个包。

![image-20200604150135444](http://img.hfzs.store/myblog/img/image-20200604150135444.png)

我们需要1.8的openjdk和openjdk-devel，-devel带了一些调试用的工具，如果没装，后面回报**Javac is not found**之类的错误。安装`yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel`。好之之后`java -version 和 javac -version`有输出即可。yum 安装会帮我们把环境变量ue配好，比解压缩方便一丢丢。

### 安装 android sdk

我们的服务器没有界面嘛，所以选用命令行安装的方式。先下载工具` wget https://dl.google.com/android/repository/commandlinetools-linux-6514223_latest.zip ` 然后解压~/下。里面就只有一个tools目录，里面包含了我们需要的**sdkmanager**，我们主要就是用它来下载和管理像**Android SDK Tools（android_sdk/cmdline-tools/）sdkmamager这些管理工具**，**Android SDK Build Tools（android_sdk/build-tools/）打包apk用的工具**，**Android SDK Platform Tools（android_sdk/platform-tools/）adb这种交互工具**，**Android Emulator（android_sdk/emulator/）模拟器**这些工具用的。原文**https://developer.android.com/studio/command-line/**

接着我们检查sdkmamager是否正常使用`$(解压路径)/tools/sdkmamager --version`查看时候有报错。运行时会有如下报错信息

```shell
...
Warning: Could not create settings
java.lang.IllegalArgumentException
    at com.android.sdklib.tool.sdkmanager.SdkManagerCliSettings.<init>(SdkManagerCliSettings.java:428)
    at com.android.sdklib.tool.sdkmanager.SdkManagerCliSettings.createSettings(SdkManagerCliSettings.java:152)
    at com.android.sdklib.tool.sdkmanager.SdkManagerCliSettings.createSettings(SdkManagerCliSettings.java:134)
    at com.android.sdklib.tool.sdkmanager.SdkManagerCli.main(SdkManagerCli.java:57)
    at com.android.sdklib.tool.sdkmanager.SdkManagerCli.main(SdkManagerCli.java:48)
...
```

原因是我们解压.zip的目录有问题和没有配置环境变量导致的。参考**https://stackoverflow.com/questions/60440509/android-command-line-tools-sdkmanager-always-shows-warning-could-not-create-se**

首先，我们新建一个放sdk所有工具的目录，把解压得到的tools放到工具目录下的cmdline-tools目录下`mkdir -p /root/android/cmdline-tools && mv -f ~/tools /root/android/cmdline-tools`。然后我们配置环境变量`vi /etc/profile`追以下信息，把sdkmaanger可以命令行调用

```shell
$ export ANDROID_SDK_ROOT=$HOME/Library/Android/sdk
$ export PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools/tools/bin
```

`source /etc/profile`刷新一下环境变量就可以shell调用啦，接下来我们要安装一系列的工具`sdkmanager "build-tools;29.0.3"`安装完这个之后他会自动帮我们把其他也装上，没有的话也可以手动安装。版本我是选最新版的。蓝色框哪四个是必须的，platforms;根据你的安卓手机版本来

![image-20200604154527556](http://img.hfzs.store/myblog/img/image-20200604154527556.png)

接下来，我们把新安装好的文件的目录也加进环境变量中。然后`vi /etc/profile`刷新。

```shell
# avdmanager, sdkmanager
export PATH=$PATH:$ANDROID_SDK_ROOT/tools/bin
# adb, logcat
export PATH=$PATH:$ANDROID_SDK_ROOT/platform-tools
# emulator
export PATH=$PATH:$ANDROID_SDK_ROOT/emulator
```

现在android sdk就安装好了，但我们真机远程调试嘛，接下来是特殊步骤。

### 真机远程调试

我们要借助adb这个工具官方说明**https://developer.android.com/studio/command-line/adb**。adb分为三个部件，

![image-20200604155219238](http://img.hfzs.store/myblog/img/image-20200604155219238.png)

client是我们本地的命令行adb程序，daemon是运行在我们手机里自带的守护进程，server一般和client一起也是运行在本地的，也就是我们的centos7.6服务器。

但其中，我们有一个很重要的问题，adb daemon也就是手机端他不会自动监听外部端口，需要我们手动开启监听。所以接下来有两套操作方法，已root和为root参考**https://blog.csdn.net/u012785382/article/details/79171782**未root 重启还原（我就是这种）

在本地电脑安装adb（参考上面）**注意是本地电脑不是服务器电脑**。然后手机插上数据线，数据线另一头插进本地电脑蓝色的usb口，手机启动usb调试模式（不懂百度把，机型不一样方法不一样）。然后命令行运行`adb devices`查看你手机是否已经连上。（有一个device，没有的话想办法令这里检测得到再往下）

![image-20200604160409854](http://img.hfzs.store/myblog/img/image-20200604160409854.png)

接下来`adb tcpip 5555`启动监听5555端口。然后就可以拔掉usb，手机接入同一网段wifi，连接`adb connect 192.168.31.196:5555`ip改为自己手机的。然后`adb devices`会看到已经连接的手机。

![image-20200604160850997](http://img.hfzs.store/myblog/img/image-20200604160850997.png)

到这里，android的操作就完成了。

接下来我们要装gradle参考**https://gradle.org/install/**也就解压而已

然后就可以用ionic直接把应用放到手机上了，执行`npm i -g cordova  native-run`和android交互用的组件，最后`ionic cordova run android --target=192.168.1.227:5555 --verbose`指定我们的远程设备运行。

![dwa1354d13wa135dwa3](http://img.hfzs.store/myblog/img/dwa1354d13wa135dwa3.gif)

如果遇到**PANIC: Broken AVD system path. Check your ANDROID_SDK_ROOT value, AVD 错误 **在~/.android新建一个avd目录`mkdir -p ~/.android/avd`