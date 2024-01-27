---
title: "构建自己的知识库：科研/开发工具链"
date: 2024-1-17T09:31:59Z
lastmod: 2024-1-17T12:56:03Z
---



差生文具多系列

CS方向，个人科研 + 开发工具链分享，主要涉及到文献阅读、代码编辑、笔记等内容
## 文献阅读
### Zotero
集文件管理、阅读、记录为一体，基本可以解决所有问题。
- 支持通过插件进行扩展，如翻译，markdown，笔记等。翻译插件可以自由配置API，选择Google、DeepL等。
- 可以使用官方同步，不过容量有限。但是支持第三方同步，可以使用坚果云等
- 缺点大概是比较吃内存，基本启动起来就会吃1GB+

如果不想配置Zotero的话也可以使用小绿鲸，各方面也都还算不错，同样支持翻译和官方同步，比较烦的是登录需要微信扫码
- [Zotero | Your personal research assistant](https://www.zotero.org/)
- [小绿鲸英文文献阅读器——专注提高SCI阅读效率](https://www.xljsci.com/)

### PDF
目前使用的是Mac上自带的预览，快速且轻量，可以满足日常使用，如果有需要操作PDF，如合并，分割，格式转换等，使用ILovePDF(除了ILovePDF，还有如ILoveIMG等)

win上面可以考虑使用Sumatra PDF，如果学校买了Adobe系列也可以使用Adobe
- [iLovePDF | Online PDF tools for PDF lovers](https://www.ilovepdf.com/)
- [Free PDF Reader - Sumatra PDF](https://www.sumatrapdfreader.org/free-pdf-reader)
## 代码编辑
### Vscode
目前笔者写的最多的是C++/Rust/Go，基本上Vscode一站式解决，vscode的配置文章网上有很多，这里就不详细展开了，放几个我目前在使用的插件：
![C++](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20240121212202.png)

rust基本上一个rust-analyzer就够了，补充一个toml文件支持的：
![rust](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20240121212446.png)

**其他插件**
- Bookmarks：为代码添加标记，方便跳转
- error-lens：更加人性化的高亮显示error
- hex-editor：阅读和编辑二进制文件
- remote-ssh：这个不必多说了，使用服务器必备
- Vim：vscode上的Vim插件
- VsCode Counter: 代码量统计，高级版wc
- Tokyo Night：个人最喜欢的主体
- Cappuccin Icon / Material Icon：图标

### Vim
目前在用Lazy vim，因为有时候写急眼了还是喜欢去摸鼠标，所以对我而言还属于玩具性质，没有将其当作主力来使用。

相同的还有LunarVim，都是属于开箱即用
## 笔记
笔者对笔记软件的基本要求有两个，一是高性能，二是同步。
- 一些笔记软件在字数达到1-2w之后就会出现较为明显的卡顿和延迟，这种就不予考虑
- 同步：同步的方式主要有两种，一是笔记直接云端存储，如notion、语雀、wolai等。另外一种形式是本地存储 + 云端备份同步，代表有obsidian和思源笔记。
针对这两个问题，之前对市面上的主流笔记软件都有体验：
- 高性能：Typora，notion、语雀以及其他云端存储笔记软件，在笔记规模达到1w-2w左右就可以感受到存在延迟和卡顿，因此不考虑作为主力来使用
- 同步：这个比较好解决，可以花钱使用官方存储，也可以自己使用github来解决。
### Obsidian
因此，综合考虑，最终笔者的选择是**obsidian**：
- ob可以选择开启GPU加速，性能有基本保证，可以处理较大规模的笔记
- 笔记管理：ob是一种类似知识库的形式，选择一个目录作为空间，之后所有笔记都管理在这个空间内，虽然不支持像noition那样进行笔记嵌套，但是可以通过文件夹分类管理
- 双链笔记：用于进行知识点的融会贯通
- 丰富的插件生态，基本能够想到的功能都有对应的插件支持
- 外观丰富，可以进行赛博暖暖
![image.png](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20240121215247.png)

#### 同步
ob有官方的同步服务，但是需要收费。由于使用的是md文件存储，因此想要自己同步也非常简单：
- 对于源文件，直接使用githubob也有对应的自动commit 和 push的插件。
- 对于图片，使用 PicGo + 图床，ob 也有对应的 PicGo 插件, 把图片复制到文本当中就会自动上传 + 返回引用链接。使用腾讯云的话，如果只供自己观看，一个月大概只需要几分钱。

除了github之外，还可以直接使用坚果云同步ob的根目录，多一份保险。

[Obsidian - Sharpen your thinking](https://obsidian.md/)

### 思源笔记
在使用ob之前，笔者用了很长一段时间的思源笔记，个人认为在使用体验上各方面都和ob不相上下，似乎是基于electron实现的，但是性能出奇地高，号称百万级别的字数也不会卡顿，比ob还要流畅。并且内存占用只有100MB左右。同时也有丰富的插件系统，甚至还有：
![源神，启动](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20240121220504.png)

不过由于不是使用md存储(可以导出为md，但是不是直接使用md存储)，手动同步起来比较费劲，最终和ob的比较还是败下阵来，只能忍痛割爱了。如果思源愿意使用md进行存储的话，我还是更愿意使用思源。

[SiYuan - Privacy-first personal knowledge management system that supports Markdown, block-level ref, and bidirectional links](https://b3log.org/siyuan/en/)

#### Notion
在上面的方案当中，ob的问题是文件是本地存储的，因此想要把笔记分享给别人比较费劲，可以分享github的链接，不过我设置成了private，所以这个就不行了。因此，在分享上我使用notion作为补充，导入起来也比较方便。

## 其他
### 浏览器
主力是Arc，今年发现的宝藏，无论是其侧边栏的网页管理还是各种小功能都用着极其舒适。
完全兼容chrome生态(毕竟是同一个内核)，可以无缝从chrome切换过来，插件，各类配置，个人收藏，记录的密码都可以全部导入。

更新很频繁，时不时会添加一些新的功能。

很难用几句话就把Arc的优点全部描述清楚，不过个人使用下来的感觉就是极度舒适，最大的问题是只登陆了Mac。
- [产品趋势02期(上)｜挑战Chrome的最强浏览器？Arc究竟牛在哪里 - 知乎](https://zhuanlan.zhihu.com/p/644989671)
- [Arc from The Browser Company](https://arc.net/)
![Arc](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20240121223242.png)

### AI助手
主力是GPT4，如果GPT抽风的话会使用Kimi Chat，各方面都还不错，支持文件上传和联网服务，并且有手机端，在手机上不想挂梯子的时候会用。
[Kimi Chat - 帮你看更大的世界](https://kimi.moonshot.cn)

### 终端
目前是Iterm2和Warp混用，使用体验都不错，Warp在命令补全和提示上效果更好，并且集成了AI，Iterm2就是更加轻量一些，如果想补全的话可以使用Fig。

Iterm2和Warp目前只登陆了Mac，比较遗憾

**Fig**

很好用的补全功能，支持iterm2,vscode和jetbrain的内置终端，不过问题同样是只登陆了Mac。效果如下，绝大多数的命令都有支持。
![image.png](https://pic-bed-1309931445.cos.ap-nanjing.myqcloud.com/blog/20240121221910.png)

### Shell
oh-my-zsh，mac和linux上都是，这个没什么好说的。

### 同步
目前都交给坚果云了，被坚果云握住了命脉
