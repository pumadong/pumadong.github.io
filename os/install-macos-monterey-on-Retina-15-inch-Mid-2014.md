# 前言

2014年中期的15寸MacPro，闲置已久，偶尔用来上下网。

根据[官方说明](https://support.apple.com/zh-cn/111980)，可以支持到Big Sur版本（MacOS 11），过去尝试升级到此版本，并没有成功。

最近有需求，于是拿出两天时间(2025/03/05-03/06)尝试进行系统升级。

# 为什么进行升级

1. 当前版本Sierra 10.12.6存在的问题。
   - Chrome版本最高只能到90，很多插件已经不再支持这个版本。
   -  科学上网全局代理客户端（e.g. ClashX），已经不支持这个版本。导致一些功能（比如Google全页翻译，Homebrew安装软件）用不了。
   - 其他很多软件在低版本运行，新特性无法使用。
2. etc.

# 升级准备

1. U盘2个。
   1. 1个8G（Yosemite 10 bootable installer），这个U盘是过去制作的，因为当时通过联网安装，一直失败，于是制作了这个启动盘并一直保存。制作方法参考这个网址，[Create a bootable installer for macOS](https://support.apple.com/en-us/101578)。也可以用下面提到的[Open Core Legacy Patcher](https://dortania.github.io/OpenCore-Legacy-Patcher/)进行制作。
   2. 1个32G（Monterey 12 bootable installer），用[Open Core Legacy Patcher](https://dortania.github.io/OpenCore-Legacy-Patcher/)进行制作。实操过程中，这个U盘制作了11-15的所有启动安装盘，最终用了Monterey12。
2. 移动硬盘1个，重装之前使用Time Machine进行备份。开辟1个100G分区，存储官网下载的10.12.6/11/12/13/14/15 PKG文件（PKG文件会在制作bootable installer时用到）。

# 安装系统

1. 用8G U盘（Yosemite 10.10.5 bootable installer），安装系统。

   这个MacOS 10也是这个机器的最初版本。安装过程很顺利。

2. [官网](https://support.apple.com/en-us/102662)下载并升级到Sierra 10.12.6，直接安装PKG到/Applications下进行安装即可，这个过程也很顺利。

3. [官网](https://support.apple.com/en-us/102662)下载并升级到Sierra 10.12.6，直接安装PKG到/Applications下进行安装即可，这个过程也很顺利。

4. 使用[Open Core Legacy Patcher](https://dortania.github.io/OpenCore-Legacy-Patcher/)制作Monterey 12 bootable installer。制作方法参考：[Install macOS on Unsupported Macs - Catalina/ Big Sur/ Monetery/ Ventura and Sonoma](https://medium.com/future-drafted/install-macos-on-unsupported-macs-catalina-big-sur-monetery-ventura-and-sonoma-a9003cf5ad75)。

# 安装应用

1. Alfred/iTerm/Sublime/搜狗输入法/[ClashX](https://clashx.org/)，Google搜索安装即可，smoothly！

2. 安装[Home-brew](https://brew.sh/)。

   - 官网不再技术支持Monterey 12以及通过OpenCore升级的版本。但是依然可以安装最新的4.4.23版本。

   - 使用官方提供的bash脚本，是会报一些Connection方面的错误的，需要全局代理。另外，Mac终端默认是不会用代理的，需要在终端执行命令：

     > export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
     >
     > 也可以把这一行加到~/.bash_profile中

     这个命令是在通过ClashX的Copy Shell Command拷贝得来。如下：

     ![clash-copy-shell](images/clash-copy-shell.png)

     我执行完这个命令，通过智能选择速度最快的香港代理服务器，依然不能安装，后来通过手工更改VPN为一个美国代理点解决。

     ![clash-download-brew-2](images/clash-download-brew-2.png)

   - brew 清理缓存

     以运行 `$ brew cleanup` 来清除这些陈旧的缓存文件让它们不再占用你的硬盘空间——然而 `cleanup` 的常规策略是仅清除超过 120 的缓存，而不是“所有”缓存，所以你可能需要使用 `--prune=all` 选项来清除所有缓存：

     ```bash
     brew cleanup --prune=all
     ```

3. 安装Typo。比较好用的所见即所得MarkDown编辑器。
   最后一个免费版本是：[0.11.18](https://zahui.fan/posts/64b52e0d/)。官网：[https://typora.io/](https://typora.io/)。

4. 配置GitHub。

   - Git Client

     > ```shell
     > ssh-keygen -t ed25519 -C "your_email@example.com"
     > ```
     > ```shell
     > $ pbcopy < ~/.ssh/id_ed25519.pub
     > # Copies the contents of the id_ed25519.pub file to your clipboard
     > ```

   - https://github.com/

     [SSH and GPG keys](https://github.com/settings/keys) -> New SSH key -> Copy SSH Key to the key textbook -> Click Add SSH Key button

   - Verifying

     > ssh -T git@github.com

     ```You may see a warning like this
     > The authenticity of host 'github.com (IP ADDRESS)' can't be established.
     > ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
     > Are you sure you want to continue connecting (yes/no)?
     ```

   - 注意：

     - Git仓库地址使用SSH方式拉取。
     ```shell
     git clone git@github.com:pumadong/pumadong.github.io.git
     ```
     - 如果之前是用https方式拉取的，需要修改远程仓库地址。有如下3种方式，任选其一。
     
       ```shell
       # 1.修改 git remote origin set-url [url]
       # 2.先删后加 git remote rm origin; git remote add origin [url]
       # 3.直接修改config文件，把项目地址换成新的。vim .git/config
       ```

   参考：

   1. [GitHub Docs](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)。
   2. [GitHub不再支持密码验证解决方案](https://cloud.tencent.com/developer/article/1861466)。

5. brew install ruby@3.3。

   会碰到openssl3.4.1因为test通不过停止安装的问题

   ![clash-copy-shell](images/clash-download-brew-3.png)

   

   下载源码自行编译安装可以解决这个问题：

   ```bash
   curl -O https://www.openssl.org/source/openssl-3.4.1.tar.gz
   
   tar xzfv openssl-3.4.1.tar.gz
   
   cd openssl-3.4.1
   
   # 不喜欢用 perl, 直接 ./config 干净利落
   ./config \
     --prefix=/usr/local/Cellar/openssl@3/3.4.1 \
     --openssldir=/usr/local/openssl@3 \
     --libdir=lib \
     no-ssl3 \
     no-ssl3-method \
     no-zlib \
     darwin64-x86_64-cc \
     enable-ec_nistp_64_gcc_128
   
   make
   
   sudo make install MANDIR=/usr/local/Cellar/openssl@3/3.4.1/share/man MANSUFFIX=ssl
   ```

   新开终端, 验证确认：openssl --version

   homebrew 链接 openssl，让 brew 正确识别, 后续安装 python@3.10 分析依赖, 自然就会跳过 openssl@3 安装

   ```
   brew link openssl@3
   ```

   以上解决办法参考这篇文章：[老旧Mac 以 homebrew 安装 openssl@3 失败的问题及解决](https://www.zfkun.com/old-mac-openssl-install.html)。

   

   此外，虽然以上办法安装完成openssl@3，但是遇到了llvm19无法安装的问题，在build的时候过不去。临时放弃3.3这个版本。

6. brew install ruby@3.1

   ![clash-copy-shell](images/clash-download-brew-4.png)

   

   安装过程顺利，需要手工把安装路径加到~/.bash_profile中。

   安装ruby(>=2.7)的目的是为了安装jekyll，这个blog生成工具可以和github.io很好的配合。

7. 安装jekyll

   直接用gem install jekyll后，安装成功，但是找不到位置执行jekyll。

   找到这个解决办法：[https://jekyllrb.com/docs/troubleshooting/#installation-problems](https://jekyllrb.com/docs/troubleshooting/#installation-problems)。

   `gem install -n /usr/local/bin jekyll`

   Quickstart测试一下：[https://jekyllrb.com/docs/](https://jekyllrb.com/docs/)。不错，很顺畅！！！

   下面要进行jekyll和github.io的集成工作了。

# 注意问题

1. High Sierra 10.13.6这个版本的下载地址非常少。如果从App Store下载，是个Stub版本(只有几十M，不是10多G)，不能用来制作bootable installer，尝试从网上找到的资源，比如[Archive](https://archive.org/details/macOS.High.Sierra.10.13.6)，又Verify失败，所以我就放弃了这个版本。

   当时之所以计划安装这个版本，是以为想从Sierra 10.12.6直接到Sonoma 14，Open Coare Legacy Patcher提示需要这个版本。后来直接从Sierra到Big Sur，就放弃这个版本了。

2. Monterey 12的安装过程，错误日志也提示过那个不兼容问题并安装失败，但是重试了两次，就成功了。

3. Ventura 13/Sonoma 14/Sequoia 15，都尝试多次，但是安装失败。开始安装后，字体马上变得特别小，几乎看不清，安装过程中有以下错误情况：

   - 可能和无线WIFI存在某种兼容问题，第一步选择网络时，识别网络并加入成功，后续安装过程中出现WIFI断网（实际WIFI是正常的）。

   - 有错误日志：no compatibility bundle on this version of macos. will assume compatible，然后安装失败。Google是有说Date问题，有说U盘问题。我确认过系统时间，是对的，U盘也多次制作，没有成功。

# 后记

- 沉下心来研究，问题总能一个一个解决！
