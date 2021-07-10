---
title: 快速发布组件到mavenCentral
tags:
  - mavenCentral
  - gradle
categories: 其他
date: 2021-07-03 13:55:40
---


## 前言

在很久之前写过一篇[发布开源库到jcenter](https://blog.csdn.net/Rain_9155/article/details/90948189)的文章，但不幸的是几个月前Jfrog发布了终止Bintray服务的[声明](https://jfrog.com/blog/into-the-sunset-bintray-jcenter-gocenter-and-chartcenter/)，声明的大概意思是说2021年3月31号之后Jcenter仓库将不再接收用户的组件提交，同时将Jcenter设为只读代码仓库，无限期地提供现有组件供用户下载，也就是说目前Jcenter仓库的状态是你无法再提交组件的更新，但你可以继续下载你以前托管的组件版本，所以现在你要做的就是把你的组件的新版本发布到其他仓库，例如Jitpack和MavenCentral，我曾经写过一篇[快速发布开源库到jitpack](https://blog.csdn.net/Rain_9155/article/details/90516026)的文章，本篇文章的内容是教你如何发布组件到MavenCentral仓库，同时写了一个发布脚本简化发布过程，使用这个脚本之前先看一下本篇文章的前期准备内容。

脚本地址：[MavenPublishScript](https://github.com/rain9155/MavenPublishScript)

## 前期准备

在讲解之前，有必要介绍一下Sonatype 、OSSRH、MavenCentral之间的关系，[Sonatype](https://www.sonatype.com/)是一间公司，它运营着[MavenCentral仓库](https://search.maven.org/)，我们想要发布组件到MavenCentral，必须要通过Sonatype的OSSRH，OSSRH即OSS Repository Hosting，对象存储仓库托管，Sonatype使用[Nexus Repository Manager](https://s01.oss.sonatype.org/#welcome)为组件提供存储库托管服务，我们发布组件时要先发布到Sonatype OSSRH上，然后才能同步到MavenCentral仓库，就好像我们之前使用Jcenter时，要先发布到Jfrog Bintray上，然后才能同步到Jcenter仓库。

### 1、注册Sonatype账号

首先你需要注册一个Sonatype Jira账号，Sonatype使用Jira管理你的groupId申请过程，注册地址如下：

[Sonatype Jira - Sign up](https://issues.sonatype.org/secure/Signup!default.jspa)

点开注册地址后，如图：

{% asset_img mavencentral1.png mavencentral %}

按要求填入你的邮箱地址、姓名、用户名、密码即可，其中用户名最好不要有中文，记住你的用户名和密码，会在后面用到。

### 2、申请groupId

我们平时使用托管在远程仓库的组件时都是通过它的GAV坐标来定位的，GAV即groupId、artifactId、version，其中groupId你可以理解为你自己在Sonatype OSSRH创建的仓库，groupId就是你仓库的名称，申请groupId就是在Sonatype OSSRH申请创建属于你的仓库，我们后面发布组件时要先发布到Sonatype OSSRH上名为groupId的仓库，然后才能同步到MavenCentral仓库。

还有申请的groupId并不是顺便填的，按照Sonatype的要求，groupId必须要是一个**域名的反写**，所以你要拥有一个域名，当你申请groupId时，Sonatype会用某种方式让你证明你是这个域名的所有者。

如果你拥有某个域名，例如example.com域名，你可以使用任何以com.example开头的groupId，例如com.example.test1、com.example.test2等，如果你没有自己的域名也没关系，Sonatype支持代码托管平台的Pages网站域名，例如Github，你可以在你的Github账号上开启你的Pages服务，这样你就拥有了一个与Github账号关联的个人域名，格式为{username}.github.io，例如我的Github Pages网站就是rain9155.github.io，很多人都是利用这种托管在三方平台的网站搭建自己的博客网站，除了Github，Sonatype还支持GitLab、Gitee等，下面表格列出这些常用的代码托管平台Pages服务开启的官方教程地址，和开启后对应的域名和相应的groupId：

| 代码托管平台 | Pages服务开启教程地址                                   | 域名                 | groupId              |
| ------------ | ------------------------------------------------------- | -------------------- | -------------------- |
| Github       | https://pages.github.com/                               | {username}.github.io | io.github.{username} |
| Gitee        | https://gitee.com/help/articles/4136                    | {username}.gitee.io  | io.gitee.{username}  |
| GitLab       | https://about.gitlab.com/stages-devops-lifecycle/pages/ | {username}.gitlab.io | io.gitlab.{username} |

下面就以我的Github Pages网站域名rain9155.github.io为例，申请名为io.github.rain9155的groupId，首先打开Sonatype Jira网站, 地址如下：

[Sonatype Jira - Dashboard](https://issues.sonatype.org/secure/Dashboard.jspa)

首次进入需要你登陆，输入你刚才注册的Sonatype Jira用户名和密码登陆，然后就进入Sonatype Jira网站首页，然后点击导航栏的**Create**按钮，此时会弹出一个弹窗，会让你填一些申请groupId时需要的信息，其中Project选择Community Support - Open Source Project Repository Hosting (OSSRH)，Issue Type选择New Project，Summary填一个标题，Group Id就填你要申请的groupId，Project URL随便填一个你的组件仓库地址，SCM url也是随便填一个你的组件仓库版本控制地址，如下：

{% asset_img mavencentral2.png mavencentral %}

最后点击create按钮，它会创建一个issue，issue名称格式为OSSRH-{taskId}，如我这里为OSSRH-69596，你可以在All Projects面板中看到它，然后接下来就是等待Sonatype Jira的邮件通知，邮件会发送到你注册账号时填写的邮箱，它的第一封邮件会叫你在你的Github中创建一个名为OSSRH-{taskId}的空仓库，从而证明你是groupId对应域名的拥有者，当你创建之后，你需要到OSSRH-{taskId}下的comment面板中回复，当你回复后，它又会再发一封邮件给你，告诉你groupId已经申请完毕，此时你可以发布组件到Sonatype OSSRH中，如何发布请看后面的内容，当你发布后，你需要在Sonatype OSSRH中把你的组件同步到MavenCentral后才可以通过GAV引用它，如何同步请看后面的内容，当你同步后，你需要再次到OSSRH-{taskId}下的comment面板中回复，然后Sonatype OSSRH才会为你激活组件同步到MavenCentral的程序，整个交流过程可以参考我[OSSRH-69596](https://issues.sonatype.org/browse/OSSRH-69596)中的comment面板，如下：

{% asset_img mavencentral3.png mavencentral %}

回复的内容是什么不重要，只要你回复了就行，上述就是申请groupId和激活MavenCentral同步的整个流程，只需要在第一次发布组件时进行一次就行，以后发布组件时不需要再进行上面的过程，直接使用该groupId就行。

### 3、生成gpg签名信息

Sonatype要求发布到MavenCentral仓库的组件中的每个文件都需要通过[gpg](https://www.gnupg.org/)签名，gpg是一个命令行工具，提供了对数据的签名和加密能力，支持主流的签名和加密算法，并提供了密钥管理系统，要使用gpg签名，我们必须先在电脑上安装gpg，然后使用gpg生成签名需要的密钥对。

我们首先来安装gpg，对于mac电脑，直接通过Homebrew安装就行，在命令行执行：

```bash
$ brew install gpg
```

对于window电脑，我们可以下载gpg的执行文件安装，下载地址如下：

[gpg download](https://gpg4win.org/download.html)

安装完成后，在命令行输入`gpg --version` 输出gpg的版本信息表示安装完成，如下：

```bash
$ gpg --version
gpg (GnuPG) 2.2.27
libgcrypt 1.8.7
Copyright (C) 2021 g10 Code GmbH
License GNU GPL-3.0-or-later <https://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Home: C:/Users/HY/AppData/Roaming/gnupg
Supported algorithms:
Pubkey: RSA, ELG, DSA, ECDH, ECDSA, EDDSA
Cipher: IDEA, 3DES, CAST5, BLOWFISH, AES, AES192, AES256, TWOFISH,
        CAMELLIA128, CAMELLIA192, CAMELLIA256
Hash: SHA1, RIPEMD160, SHA256, SHA384, SHA512, SHA224
Compression: Uncompressed, ZIP, ZLIB, BZIP2
```

然后在命令行输入`gpg --full-generate-key` 进行密钥对生成，这个命令会让你一步步选择生成密钥对需要的信息，如下：

```bash
$ gpg --full-generate-key
gpg (GnuPG) 2.2.27; Copyright (C) 2021 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
  (14) Existing key from card
Your selection? 4
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072)
Requested keysize is 3072 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0)
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: rain9155
Email address: jianyu9155@gmail.com
Comment:
You selected this USER-ID:
    "rain9155 <jianyu9155@gmail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key DF7B4B4D1A32FB02 marked as ultimately trusted
gpg: revocation certificate stored as 'C:/Users/HY/AppData/Roaming/gnupg/openpgp-revocs.d\9E02D0E104E3D3517B5B0E54DF7B4B4D1A32FB02.rev'
public and secret key created and signed.

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
pub   rsa3072 2021-07-06 [SC]
      9E02D0E104E3D3517B5B0E54DF7B4B4D1A32FB02
uid                      rain9155 <jianyu9155@gmail.com>
```

首先它会让你选择算法，我这里选择了RSA，仅用于签名，然后会让你输入密钥长度，默认是3072bits，然后会让你选择密钥的过期时间，我这里选择了永久有效，建议大家也选择永久有效，然后确认信息后会让你输入姓名、邮箱、注释来作为密钥的唯一标识，这里我生成的标识为"rain9155 \<jianyu9155@gmail.com\>"，最后确认后会弹出一个弹窗让你输入一个密码来保护你的私钥，记住你输入的密码后面会用到，点击确认后看到public and secret key created and signed这句话说明密钥对生成完毕，我们可以通过`gpg --list-keys`列出生成的公钥信息，通过`gpg --list-secret-keys`列出生成的私钥信息，但其实列出的信息是类似的，我们以`gpg --list-keys`为例，如下：

```bash
$ gpg --list-keys
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   2  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 2u
C:/Users/HY/AppData/Roaming/gnupg/pubring.kbx
---------------------------------------------
pub   rsa3072 2021-07-06 [SC]
      9E02D0E104E3D3517B5B0E54DF7B4B4D1A32FB02
uid           [ultimate] rain9155 <jianyu9155@gmail.com>
```

列出的信息首先有公钥的文件位置，这里为C:/Users/HY/AppData/Roaming/gnupg/pubring.kbx，接着是pub rsa3072 2021-07-06 [SC]，表示使用RSA算法，公钥长度为3072bits，创建日期为2021-07-06，接着是长长的一串id值9E02D0E104E3D3517B5B0E54DF7B4B4D1A32FB02，它表示公钥的id，我们使用时只需要用到它的后八位就行，所以我们后面使用这个公钥时只需要使用1A32FB02就行，最后是公钥的唯一标识uid，根据我们前面输入的姓名、邮箱、注释生成。

我们签名只需要用到私钥，而公钥需要上传到公钥服务器，这样我们用私钥签名的文件上传到MavenCenral后，它才能使用公钥验证这个文件，这里我选择了keyserver.ubuntu.com这个公钥服务器，上传公钥的命令格式为`gpg --keyserver 公钥服务器地址 --send-keys 公钥id`，如下我把刚刚生成的公钥1A32FB02上传到公钥服务器：

```bash
$ gpg --keyserver keyserver.ubuntu.com --send-keys 1A32FB02
gpg: sending key DF7B4B4D1A32FB02 to hkp://keyserver.ubuntu.com
```

没有错误提示就表示上传成功，我们还可以使用`gpg --keyserver 公钥服务器地址 --recv-keys 公钥id`从公钥服务器导入公钥到本地从而验证公钥是否上传成功，如下：

```bash
$ gpg --keyserver keyserver.ubuntu.com --recv-keys 1A32FB02
gpg: key DF7B4B4D1A32FB02: "rain9155 <jianyu9155@gmail.com>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
```

可以看到导入成功，表示公钥已经成功上传到公钥服务器。

还有最后一步，把私钥导出到一个文件中，这样我们发布组件时才能通过引用这个文件，从而使用私钥进行签名，命令格式为`gpg -o 导出的文件路径 --export-secret-key 私钥id`，执行导出私钥命令会弹出一个弹窗让你输入私钥密码，输入正确后才能成功导出，如下把私钥导出到名为1A32FB02.gpg的文件中，私钥id和公钥id是一样的：

```bash
$ gpg -o D:/File/keystore/1A32FB02.gpg --export-secret-key 1A32FB02
```

现在已经成功把私钥导出D:/File/keystore/1A32FB02.gpg中，记住你导出的私钥文件路径，后面会用到。

## 发布到OSSRH

现在万事俱备，可以发布组件了，发布组件使用`maven-publish`插件，我们主要发布的内容有组件的aar或jar文件、sources文件、javadoc文件，pom文件，还有这些文件的签名文件，以.asc为后缀，签名要使用`signing`插件，为了简化发布过程，我已经把发布过程编写成了一个gradle脚本 - [publication.gradle](https://github.com/rain9155/MavenPublishScript/blob/main/script/publication.gradle)，使用时只需要apply进来，然后填写发布的基本信息，执行发布任务就可以自动生成发布所需的所有文件并发布到OSSRH中，下面简单介绍如何使用这个gradle脚本。

首先在你的组件的build.gradle中apply该脚本：
```groovy
apply from: 'https://raw.githubusercontent.com/rain9155/MavenPublishScript/main/script/publication.gradle'
```
接下来准备好你在前面注册的Sonatype账号、申请好的groupId、生成好的gpg私钥信息，然后在组件的gradle.properties(没有则创建)中添加组件信息，其中GAV坐标是必填信息，其他是可选信息：
```groovy
# GAV坐标
GROUPID=申请好的groupId，例如io.github.rain9155
ARTIFACTID=组件名，例如mavenpublishscript
VERSION=组件版本，例如1.0.0，版本加SNAPSHOT后缀可发布到maven远程snapshot地址，如1.0.0-SNAPSHOT

# 可选信息
DESCRIPTION=gradle发布组件到MavenCentral脚本，支持aar和jar发布
URL=https://github.com/rain9155/MavenPublishScript
LICENSENAME=The Apache License, Version 2.0
LICENSEURL=http://www.apache.org/licenses/LICENSE-2.0.txt
DEVELOPERNAME=rain9155
DEVELOPEREMAIL=jianyu9155@gmail.com
# scm信息，格式参考http://maven.apache.org/scm/scm-url-format.html
SCMURL=https://github.com/rain9155/MavenPublishScript/tree/master
SCMCONNECTION=scm:git:git://github.com/rain9155/MavenPublishScript.git
SCMDEVELOPERCONNECTION=scm:git:ssh://github.com:rain9155/MavenPublishScript.git
# 如果发布的是android组件，为true时，支持根据flavor动态生成组件名称，规则为ARTIFACTID-{flavorName}，默认为false
APPENDFLAVORNAME=true
```
然后在项目根目录的local.properties(没有则创建)中添加gpg签名信息和ossrh账号信息，记得要把local.properties从你的版本控制中移除，避免泄漏你的签名信息和账号信息：
```groovy
# gpg签名信息
signing.keyId=你的密钥id，例如1A32FB02
signing.password=你的私钥密码
signing.secretKeyRingFile=你的私钥文件路径，例如D:/File/keystore/1A32FB02.gpg

# ossrh账号信息
ossrh.username=你的ossrh账号名，即Sonatype账号用户名
ossrh.password=你的ossrh账号密码，即Sonatype账号密码
```
最后在命令行执行gradle任务发布组件，如果你发布的是android组件(aar)，执行的任务的名称格式为`publish{flavorName}AndroidlibPublicationToMavenRepository`或`publish{flavorName}AndroidlibPublicationToMavenLocal`，其中flavorName为android组件的flavor的名称，首字母大写，没有则不填flavorName，如果你发布的是java组件(jar)，执行的任务名称为`publishJavalibPublicationToMavenRepository`或`publishJavalibPublicationToMavenLocal`：
```bash
//发布android组件到maven本地仓库
gradle publishAndroidlibPublicationToMavenLocal

//发布android组件到maven远程release或snapshot仓库
gradle publishAndroidlibPublicationToMavenRepository

//假设android组件含有flavorName为china，发布china版本的android组件到maven本地仓库
gradle publishChinaAndroidlibPublicationToMavenLocal

//假设android组件含有flavorName为oversea，发布oversea版本的android组件到maven远程release或snapshot仓库
gradle publishOverseaAndroidlibPublicationToMavenRepository

//发布java组件到maven本地仓库
gradle publishJavalibPublicationToMavenLocal

//发布java组件到maven远程release或snapshot仓库
gradle publishJavalibPublicationToMavenRepository

//发布所有android组件和java组件到maven本地仓库
gradle publishToMavenLocal

//发布所有android组件和java组件到maven远程release或snapshot仓库
gradle publishAllPublicationsToMavenRepository

//发布所有android组件和java组件到maven本地仓库和maven远程release或snapshot仓库
gradle publish
```

上述发布任务根据组件发布的是android组件(aar)还是java组件(jar)来执行，同时发布android组件时支持flavor发布，publishXXToMavenRepository任务表示发布到远程仓库OSSRH中，publishXXToMavenLocal任务表示发布到maven本地仓库，maven本地仓库路径为{用户目录}/.m2/repository/，也可以在AS左侧的gradle面板中查看这些任务，如下：

{% asset_img mavencentral4.png mavencentral %}

选中合适的任务双击执行就行，如果你执行的是publishXXToMavenRepository任务，任务执行成功后你就可以到[OSSRH](https://s01.oss.sonatype.org/#welcome)查看你发布上去的组件，首次进入需要登陆，输入你的Sonatype用户名和密码登陆，然后点击左下侧的**Staging Repository**就可以看到你发布到OSSRH的组件，如下：

{% asset_img mavencentral5.png mavencentral %}

在Staging Repository的面板中可以浏览你发布的组件，选中它，然后点击下面的content面板就可以查看组件发布的详细文件。

## 同步到MavenCentral

组件发布到OSSRH后，还不能通过GAV引用它，还要同步到MavenCentral后才能通过GAV引用它，首先要在OSSRH上**Close**组件，让它从open状态变为close状态，如下：

{% asset_img mavencentral6.png mavencentral %}

当你点击Close按钮后，会弹出一个弹窗叫你确认，确认后Close按钮会变灰，此时下方的Activity面板会显示close过程中进行的规则校验，例如校验你发布的文件是否有签名、是否含有javadoc文件、是否含有sources文件，pom文件是否合格等，当所有校验通过后你才可以同步组件到MavenCentral，当校验通过后你会收到Sonatype发送的邮件，此时你可以点击上方的**Release**按钮同步组件到MavenCentral，如下：

{% asset_img mavencentral7.png mavencentral %}

当你点击Release按钮后，会弹出一个弹窗叫你确认，确认后Release按钮会变灰，如果你是第一次发布组件，在**申请groupId**时我讲过，你还要到OSSRH-{taskid}的comment面板下回复它一次OSSRH才会为你激活同步程序，激活同步程序后大概十几分钟你的组件就会被同步到MavenCentral，这时你就可以在[MavenCentral网站](https://search.maven.org/)搜索到你的组件，并可以在gradle或maven等构建工具中通过GAV引用它，激活同步程序后，你下次Release组件时就会自动同步到MavenCentral而无需再做其他操作。

## 结语

发布到MavenCentral的步骤相比于发布到Jitpack要复杂的多，Jitpack只需要一个代码托管仓库账号，而MavenCentral需要准备Sonatype账号、个人域名、签名信息，但是MavenCentral比Jitpack成熟得多，它目前是最大的java组件和其他开源组件的托管仓库，也是很多构建系统如Maven的默认存储仓库，如何选择就看个人爱好了。

以上就是本文的所有内容！

参考资料：

[JCenter 服务更新](https://developer.android.google.cn/studio/build/jcenter-migration?hl=zh_cn)

[The Central Repository Documentation](https://central.sonatype.org/publish/publish-guide/)

