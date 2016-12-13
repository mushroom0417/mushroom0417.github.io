<!DOCTYPE html>

<meta charset="utf-8">



<meta http-equiv="X-UA-Compatible" content="chrome=1">



<meta name="description" content="Mushroom0417.GitHub.io : mushroom0417's blog">



<link rel="stylesheet" type="text/css" media="screen" href="stylesheets/stylesheet.css">



<title>Mushroom0417.GitHub.io</title>

apksigner是Android Build Tools从24.0.3版本引入的一个新的apk文件签名工具，用它可以替代以往一直在使用的jarsigner，而且只要apk文件的AndroidManifest声明支持的sdk范围包含24或者以上（就是安卓7.0或者以上），那么工具会自动为apk文件打入新的签名格式，名为[**APK Signature Scheme v2**](https://source.android.com/security/apksigning/v2.html)。经测试该签名方式在安卓7.0或者以上的系统上能大大的提高安卓的速度（这里可能存疑，不确定是否是新的签名格式导致的）。而且使用旧有的签名方法我这边测试到部分apk文件无法在安卓7.0的机子上安装，一般都是提示哪个文件size不对，或者签名对不上，然后我使用apksigner重新签名之后就可以了。根据[这篇文章](https://source.android.com/security/apksigning/v2.html)，谷歌说到新系统是兼容旧的V1的签名格式的（没有V2的签名就用V1的方式来验证），但是我的的确确遇到了安装包只有V1签名的只能在7.0以下机子安装无法在7.0或者以上机子安装的情况，谷歌了一大轮也没找到结果，所以为了确保我们的安装包能够正常的安装在各个版本的机子上，我们还是有必要对我们的包打上两个版本的签名的（这里如果是使用**Android Stuidio**来打包的话就可以忽略了，只要把gradle配置文件的**buildToolsVersion**设置到**24.0.3**或者以上的话（文章发表当前最新的版本为**25.0.1**），IDE会自动为我们打上兼容7.0的签名）。下面就将我迁移新的签名工具遇到的坑和解决方法分析给大家。

apksigner签名的命令
```shell
$ apksigner sign --ks <keystore file path> --ks-key-alias <alias> --ks-pass pass:<keystore password> --key-pass pass:<key password> --out <output apk path> <input apk path>
```
* ks 对应build.gradle里面的SigningConfig里面的**storeFile**
* ks-key-alias 对应build.gradle里面的SigningConfig里面的**keyAlias**
* ks-pass 对应build.gradle里面的SigningConfig里面的**storePassword**
* key-pass 对应build.gradle里面的SigningConfig里面的**keyPassword**

apksigner具体的参数定义和使用方法可以参考谷歌的[这篇文章](https://developer.android.com/studio/command-line/apksigner.html)
其中我再提下其中2个需要注意下的参数，一个是ks-type一个是ks-provider-name，这两个字段可以通过
```shell
$ keytool -list -v -keystore *.keystore
```
这个命令来查看到，一般我们的keystore文件的type都是JKS的，apksigner的这个type的默认值也是JKS，所以如果是JKS的话这个参数可以不填。

要验证签名是否成功，我们可以使用以下这个命令来验证
```shell
$ apksigner verify -v apk文件路径
```
如果出现类似这样的提示就代表签名是兼容7.0以下和以上的。如果v2 scheme为false即代表在7.0上安装时是用旧的验证签名的方式。
```shell
$ Verifies
$ Verified using v1 scheme (JAR signing): true
$ Verified using v2 scheme (APK Signature Scheme v2): true
$ Number of signers: 1
```
但是第一个坑即将来临，你安装上诉签好名并对齐后的应用的时候就会提示
```shell
$ SHA-256 digest of contents did not verify
```
谷歌了一通，发现这个提示，[原文链接](https://developer.android.com/studio/command-line/zipalign.html)
>Caution: You must use zipalign at one of two specific points in the app-building process, depending on which app-signing tool you use:
* If you use **apksigner**, zipalign must only be performed ***before*** the APK file has been signed. If you sign your APK using apksigner and make further changes to the APK, its signature is invalidated.
* If you use **jarsigner**, zipalign must only be performed ***after*** the APK file has been signed.

因为我们对齐应用都是在签名之后，但是新的打签方式有点不一样，如果这样做的话就会破坏掉apk的信息，导致验签失败，所以我们需要把对齐的步骤提前到签名之前。
在高高兴兴的以为已经解决了问题的时候，我修改了打签对齐的流程然后安装包，结果仍然提示这个……然后我到处谷歌百度，无奈关于apksigner使用的文章几乎没有，只能自己硬着头皮去尝试找出解决方案。在几乎要放弃的时候，我突然想起最新的sdk build tools好像可以更新了，于是我打开SDK Manager将其更新到25.0.1，并将命令行apksigner的执行转到25.0.1的目录去，重启下terminal，然后重新签名、安装。然后居然安装成功了……好吧，可能是新版本修复了这个问题，可以谷歌上没有人讨论过……可能他们都不用像我这样苦逼的只能用命令行来打包文件。
至此，打包工具的更新已完结。顺利的使得apk的签名兼容到所有的版本。

参考文章：
* apksigner工具介绍 https://developer.android.com/studio/command-line/apksigner.html
* 签名介绍 https://developer.android.com/studio/publish/app-signing.html
* zipalign工具介绍 https://developer.android.com/studio/command-line/zipalign.html
* 新签名格式介绍 https://source.android.com/security/apksigning/v2.html
* keytool工具使用方法介绍http://janrone.com/2015/12/29/%E6%9F%A5%E7%9C%8Bapk%E7%AD%BE%E5%90%8D-%E5%92%8C-keystore-%E7%9A%84%E4%BF%A1%E6%81%AF/

<!-- HEADER -->

 <div id="header_wrap" class="outer">
  <header class="inner">
  <a id="forkme_banner" href="https://github.com/mushroom0417">View on GitHub</a>
  <h1 id="project_title">Mushroom0417.GitHub.io</h1>
  <h2 id="project_tagline">mushroom0417's blog</h2>
</header>
</div>

<!-- MAIN CONTENT -->

 <div id="main_content_wrap" class="outer">
  <section id="main_content" class="inner">
  <h3><a id="welcome-to-github-pages" class="anchor" href="#welcome-to-github-pages" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link">
</span></a>Welcome to GitHub Pages.</h3>
  <p>apksigner &gt;= 24.0.3\. Author your page content here <a href="https://guides.github.com/features/mastering-markdown/">using GitHub Flavored Markdown</a>, select a template crafted by a designer, and publish. After your page is generated, you can check out the new <code>gh-pages</code> branch locally. If you’re using GitHub Desktop, simply sync your repository and you’ll see the new branch.</p>
  <h3><a id="designer-templates" class="anchor" href="#designer-templates" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link">
</span></a>Designer Templates</h3>
  <p>We’ve crafted some handsome templates for you to use. Go ahead and click 'Continue to layouts' to browse through them. You can easily go back to edit your page before publishing. After publishing your page, you can revisit the page generator and switch to another theme. Your Page content will be preserved.</p>
  <h3><a id="creating-pages-manually" class="anchor" href="#creating-pages-manually" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link">
</span></a>Creating pages manually</h3>
  <p>If you prefer to not use the automatic generator, push a branch named <code>gh-pages</code> to your repository to create a page manually. In addition to supporting regular HTML content, GitHub Pages support Jekyll, a simple, blog aware static site generator. Jekyll makes it easy to create site-wide headers and footers without having to copy them across every page. It also offers intelligent blog support and other advanced templating features.</p>
  <h3><a id="authors-and-contributors" class="anchor" href="#authors-and-contributors" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link">
</span></a>Authors and Contributors</h3>
  <p>You can @mention a GitHub username to generate a link to their profile. The resulting <code>&lt;a&gt;</code> element will link to the contributor’s GitHub Profile. For example: In 2007, Chris Wanstrath (<a href="https://github.com/defunkt" class="user-mention">@defunkt</a>), PJ Hyett (<a href="https://github.com/pjhyett" class="user-mention">@pjhyett</a>), and Tom Preston-Werner (<a href="https://github.com/mojombo" class="user-mention">@mojombo</a>) founded GitHub.</p>
  <h3><a id="support-or-contact" class="anchor" href="#support-or-contact" aria-hidden="true"><span aria-hidden="true" class="octicon octicon-link">
</span></a>Support or Contact</h3>
  <p>Having trouble with Pages? Check out our <a href="https://help.github.com/pages">documentation</a> or <a href="https://github.com/contact">contact support</a> and we’ll help you sort it out.</p>
</section>
</div>

<!-- FOOTER -->

 <div id="footer_wrap" class="outer">
  <footer class="inner">
  <p>Published with <a href="https://pages.github.com">GitHub Pages</a></p>
</footer>
</div>
