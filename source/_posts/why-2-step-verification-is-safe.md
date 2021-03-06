title: Google两步登录的安全性分析与保护措施
tags:
  - Google
  - 安全
id: 1369
categories:
  - 胡说八道
date: 2011-03-04 06:30:10
---

前两天写了篇关于[Google两步登录的介绍文章](http://chensd.com/2011-03/google-2-steps-verification.html)，在[推上](http://twitter.com/#!/Woo997/status/42749365391081472)和[cnBeta](http://www.cnbeta.com/articles/136126.htm)上都有不少人质疑其安全性，主要是因为对电信运营商的不信任。两步登录用了两天多了，今天早上也专门试了试它的一些情况，在这里刚好总结一下。

设置为两步登录后，完成第二步登录的数字码有三种途径可以获得，一是通过手机上的程序生成，二是通过设定的手机接收短信，三是设置时生成的备用数字验证码。此外，利用程序特有密码（Application-specific Password）也是一个访问Google账户中数据的方法，下面分别从这四方面来分别分析它的安全性。

需要说明的是，**对于设置了两步登录的账号，单独得到其账户密码和数字验证码都是没有用的**，而且，通过程序生成和短信接收得到的六位数字验证码还有一个时效限制。<!--more-->

## 一、手机程序生成数字码的安全性

六位数字码每30秒就改变一次，具体哪一秒改变，不同的设备不一样，时间是数字码生成中的一个很重要的变量，无论是是关闭程序还是如何，只要是30秒内，生成的数字码是不变的。但这并不意味着数字码被猜中的机率为百万分之一，因为这个数字码是有“有效期”的，时间从两步登录中第一步登录数据的提交开始计算，从那开始，五分钟内生成的共计10个数字验证码中任意一个都可以辅助完成登录，也就是说，数字码验证被突破的机率应该是十万分之一。这一点也是用户无法控制的，保密工作的重点不在这一步。

Android平台上用于生成六位数字码的程序Google Authenticator，只有一个权限要求，即控制振动器——成功的在添加一个账号后，将振动以示提醒。在断开手机所有的网络后，仍然可以完成程序的安装，并可正常的添加账号用以进行两步登录，亦可正常的生成数字码进行登录，这意味着，在整个过程中，Google Authentication没有往Google的服务器传送任何数据，也即与手机本身的各项信息无关，只是用某种算法进行了一个计算。

为了验证生成的数字码的确与设备无关，在设置两步登录时，同时使用两台手机扫描了那个QR码图，结果两台手机都可以辅助完成两步登录。而且，两台手机生成的数字码也是一致的。这丙一次证明了数码验证码的生成与设备无关。

其实，在设置两步登录中生成的用于扫描的QR码所加密的内容是一下如下的字符串：
> otpauth://totp/xxxxx@gmail.com?secret=5dytxxxxxxxdc3tb
其关键内容有两个，一个是用户名，另外一个就是secret后的密钥，每一个账号的密钥是不变的。除了用QR码扫描的方式在Google Authenticatior添加账号外，也可以手动添加，手动添加时需要输入的，也是这两个信息。

现在的问题是，生成的数字码与用户名是否有关系，为了验证，在Google Authentication中手动添加一个账号，用户名随便，密钥使用上面的密钥。结果表明，只要密钥一样，无论用户名是什么，都生成一样的数字码。这意味着，六位数字码的生成只与那个secret密钥和时间相关，保护secret密钥就成为了保密的重心，好在Google Authentication中无法查看secret密钥。要想查看secret密钥，只有在登录进Google账户后。

### 保护措施

这一步的保密有两个，一是保证secret密钥不外泄，二是保证用于生成数字码的设备不要被他人获取。主要为以下几点

*   **不在任何地方粘贴含有secret的QR码图**
*   在不使用Google服务后，一定要注意点击右上角的**登出（Sign Out）**
*   **在Google Authentication中添加数个账号，用户名随意（可以为任意字符，不限于邮箱格式），密钥为任意16位字符数字组合**，以使即使获得这个设备，也要花精力去分辨到底哪个六位数字码为真
*   遇到突发情况时，尽量**立即卸载设备上的Google Authentication程序**
[caption id="attachment_1376" align="aligncenter" width="240" caption="在Google Authenticator应用中添加多个干扰的无用账号，名称随意"][![](/upfile/2011/03/Android_Google_Authentication_UI.png "Android_Google_Authentication_UI")](/upfile/2011/03/Android_Google_Authentication_UI.png)[/caption]

## 二、手机短信接收验证码的安全性

这个是最为话柄的一个方法，因为需要使用短信接收六位数字码，而朝内电信运营商个个都有不良记录，并且这一步无法跳过，设置完成后，一旦删除备用手机号码，两步登录也会自动失效，这一步的确有些不人性化。

不过，陌生人在登录到你的账号之前，只知道备用手机号的最后两位，虽然两步登录设置过程中，Google推荐你用值得信任的人的号码作为备用短信接收号码，但如果你真用了你熟悉的人的号码，可能又是一个安全隐患。

### 保护措施

最好的办法是，**在网上找一个朋友，确保只有自己知道这个人与自己的关系，然后借用其手机号为备用号码**。这样，就算最了解你的人，依然无法弄清楚备用短信接收号码是多少，**如果你可以找到一个手机尾号与你某个熟人的号码一致的网友，那就更具迷惑性了**。而且，两步登录的第二步有超时时间，大约是五到十分钟，我相信，在这个时间段时做一次全国范围的短信搜索，还是非常有难度的。

[caption id="attachment_1377" align="aligncenter" width="350" caption="两步登录会提示备用手机号码的末两位"][![](/upfile/2011/03/3-way-to-login-google-accounts-with-2-steps-verification.png "3-way-to-login-google-accounts-with-2-steps-verification")](/upfile/2011/03/3-way-to-login-google-accounts-with-2-steps-verification.png)[/caption]

## 三、备用数字验证码

这个方法是在以上两个方法都无法登录的时候用的，Google提供了10个八位数字验证码，每个号码可以使用一次，Google推荐你打印出来，不过这可不是个什么好办法。如何保护好它成了最麻烦的一点。

### 保护措施

为了确保这10个八位数字验证码的安全性，建议按如下几步进行操作

*   1、**将数字码录入到一个文本文件，并将进行基础性编码**，例如，将所有出现3的地方换成5，将所有出现5的地方换成3
*   2、使用截图软件将第一步的结果进行截图，得到一个图片
*   3、**使用winrar等工具，将这个图片压缩，并设置密码**，一定要设置一个与Google账户不同的密码，得到的压缩包文件名一定要随意，如“新建 文本文档.rar”
*   5、将这个图片传到可以永久保存的网盘上，并将其共享（注意不要选择如115等有共享期限限制的网盘），记下共享地址
*   6、找一个可以自定义的缩短网址工具，将第五步的共享地址进行缩短，这个网址一定要记牢。[http://snipurl.com](http://snipurl.com)就是个可以进行自定义的网址缩短工具，如缩短的最终结果为[http://snipurl.com/wodemimazaizheli](http://snipurl.com/wodemimazaizheli)
[caption id="attachment_1378" align="aligncenter" width="200" caption="备用八位数字验证码提供了十次机会"][![](/upfile/2011/03/used-bakup-verification-codes.png "used-bakup-verification-codes")](/upfile/2011/03/used-bakup-verification-codes.png)[/caption]

这样，其它方法都无法登录，或者登录途径被破坏后，就可以打开[http://snipurl.com/wodemimazaizheli](http://snipurl.com/wodemimazaizheli)，然后下载备用验证八位数字码图——可别忘了你还对它进行了基础性的编码。

## 四、真正的危险：程序特有密码（Application-specific password）

因为有些程序是不支持两步登录的，比如桌面版Gtalk、Empathy、Picasa，甚至Chrome浏览器的同步功能，为了使这些程序能够正常的通过Google账户验证，需要为其生成一个程序特有密码，而这个密码，却可以用在任何地方，虽然不能用它来登录Google账户和Gmail，但却可以来完成邮件客户端的验证，再使用邮件客户端来收发邮件。

一旦生成了程序特有密码，在Google账户中便不能再查看密码，只能够删除。虽然诸如Gtalk和Emparty等软件会用星号来显示密码，但想看到星号隐藏的密码并不困难。这也使得程序特有密码成了两步登录的薄弱环节，因此建议，最好使用两个Google账户，一个用于邮件等重要事务，另外一个账号专门用于不能进行两步登录的场合。