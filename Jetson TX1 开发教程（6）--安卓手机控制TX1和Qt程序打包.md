<h2><font color=#0099FF>前言</font></h2> 

如果身边没有显示器，那我们我们应该如何控制TX1？方法应该比较多，这里只说我最喜欢的方法，juiceSSH+VNC Viewer，**既有命令模式，也有图形模式**。有人可能问，之前的博客已经说过VNC Viewer了啊，就用这个岂不是可以？但是如果所在现场没有WiFi那就没法连接。我们想到，安卓手机还可以开热点啊，让手机热点充当局域网，还需要juiceSSH的帮助，一图胜千言，效果图如下。

![这里写图片描述](http://img.blog.csdn.net/20170722223908424?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

<h2><font color=#0099FF>配置步骤</font></h2> 

**配置过程依然是要使用HDMI显示器的，配置好后就可以单独使用手机进行控制了。**先做一些准备工作：

工具：安卓手机，Jetson TX1

软件：安卓手机需要安装juiceSSH和VNC Viewer，国内豌豆荚好像有，网络好的同学也可使用[APKMirror](http://www.apkmirror.com/) 。

TX1需要安装VNC server，执行这个命令`sudo apt-get install tightvncserver`

用安卓手机开启便携式热点，用户名密码可以改的简单点，然后让TX1连接到该热点上，为了自动连接到这个网络，可以让TX1忘记所有其他WiFi。

在TX1上执行`ifconfig` 查看网络连接情况，如下图所示，主要查看wlan0的Ip地址，比如这里就是192.168.43.165。

![这里写图片描述](http://img.blog.csdn.net/20170722222051440?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

打开手机上的juiceSSH软件，点击闪电按钮，快速建立一个连接，类型就是SSH，填入上述IP地址，点击确定；然后徐州呢新建一个账户来登录IP，填写相关信息，昵称随便写，用户名写TX1的登录用户名“ubuntu”，密码也填写“ubuntu”，点击确定。第一次连接需要接受主机认证，一会就连接成功，此时可以在手机上的远程终端操作TX1。

![这里写图片描述](http://img.blog.csdn.net/20170722222114833?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

现在可以使用TX1的命令行了，如果还想看TX1桌面，那就要借助VNC，在命令模式之下输入`vncserver`，首次使用它会提示设置一个密码，设完密码后，提示“Would you like to enter a view-only password？”，为了方便，我选择了no。此时终端提示我们已经开启了vncserver，而且还给出了一个桌面号，这里的桌面号应该默认为1，如下图所示。

![这里写图片描述](http://img.blog.csdn.net/20170722222137060?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

打开VNC Viewer，新建一个连接，IP地址填写刚才的那个，用户名填写“ubuntu”，我这里并不需要填写端口号，不过知道一下，端口号是5901（如果桌面号为2，那端口号就是5902）。接下来点击确定就能看到TX1的桌面了。

![这里写图片描述](http://img.blog.csdn.net/20170722222434499?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![这里写图片描述](http://img.blog.csdn.net/20170722222710655?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSmVzc2VfTXg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
（这个是手机截屏，不是显示器）

细看一下，分辨率真是感人，应该只有800x600，手机好歹也是1080p的，好多东西显示不全。**这个暂时没找到方法**，TX1中`/etc/default/` ，没有发现grub文件，不能用这个方法修改分辨率。但是，如果同时连着显示器，分辨率又回到1080p，真是尴尬。

<h2><font color=#0099FF>Qt程序的简单打包</font></h2> 

博主使用手机控制TX1的时候就想啊，运行Qt里面写的caffe程序还是不太方便。打开qt，然后选择项目，再运行，在小小的手机上确实麻烦。如果将软件打包，就能使用juiceSSH中的命令行执行，无需进入图像界面。

查了一下，打包主要分为**静态打包**和**动态打包**，类似于Windows中的单一安装包和便携版。静态打包比较麻烦，学起来较为困难。而动态打包则是将可执行文件和依赖库放一起，方便简易，而且程序如有变动，直接替换可执行文件就行。

动态打包分为以下几步：

- 在主目录下新建一个空文件夹，用于存放所需文件，比如我的文件夹叫做portable。
- 将测试程序caffe-test使用release模式编译，**注意一定要是release模式**。
- 打开编译的文件夹，将得到的可执行文件“caffe-test”复制到新建的portable文件夹中。
- 编写ldd.sh脚本，放置在portable文件夹中，脚本的作用是批量复制程序所需依赖库到指定文件夹，脚本内容如下：

```shell
#!/bin/sh  
  
exe="caffe-test" # 程序名称 
des="/home/mx/portable" # 指定文件夹  
  
deplist=$(ldd $exe | awk  '{if (match($3,"/")){ printf("%s "),$3 } }')  
cp $deplist $des 
```

`./ldd.sh`运行脚本，可以发现portable文件夹中瞬间多出几十个.so文件，这些都是程序用到的依赖库。

- 编写caffe-test.sh脚本，该脚本一定要和可执行程序同名，并且和刚才的依赖库放在同一文件夹下，脚本内容如下：

```shell
#!/bin/sh
appname=`basename $0 | sed s,\.sh$,,`

dirname=`dirname $0`
tmp="${dirname#?}"

if [ "${dirname%$tmp}" != "/" ]; then
dirname=$PWD/$dirname
fi
LD_LIBRARY_PATH=$dirname
export LD_LIBRARY_PATH
$dirname/$appname "$@"
```

这个脚本作用是链接函数库并运行可执行文件，内容来源于http://doc.qt.io/qt-5/linux-deployment.html，请勿做任何修改。

- 修改脚本权限，使用如下命令：

```shell
sudo chmod +x caffe-test.sh
```

- 运行脚本`./caffe-test.sh` ，一般来说程序可以正常跑起来，但是也有可能报错，比如会有缺少`libcaffe.so.1.0.0-rc3`的错误，这时候检查一下文件夹，果然没有复制到这个库，那我直接去caffe文件夹把这个`libcaffe.so.1.0.0-rc3`库拷过来就完了，再次运行脚本，错误消失。

参考文献：
http://blog.lxx1.com/2403
http://blog.csdn.net/hjl_1991/article/details/50365927
