---

title: homelab系列-本子管理
date: 2022-6-14 10:01:58
permalink: manager_comic_and_doujinshi/
tags:
- nas
- homelab
- comic
---

## 背景

我有一个服役超过7年的android平板`Sony z3 tablet compact`，上面运行着号称android核心竞争力的应用：ehviewer。

长久以来，我都有每天登录ehviewer刷新一下订阅并下载新的漫画本子的习惯。之前我就有想过这个流程能不能优化，一方面每天手动刷新下载比较繁琐，并且若是不及时下载，一些资源在上传之后又会被删除。此外由于ehentai的资源限制，ehviewer下载原图并不方便，通常下载的还是重采样后的缩放图片，在平板上阅览虽然影响不大，若是作为仓鼠收藏则有点过意不去。

而且在长年使用之后，ehviewer中的下载列表变的很长，在翻阅上也有一些不便，大量漫画的管理并不是ehviewer的核心功能，基础的检索、收藏功能也是依托ehentai提供的api实现的。ehentai在功能上虽然没什么可以挑剔的，不过也曾面临过关站风险，而且部分版权物也会被删除，如果有办法做本地管理作为替代和补足，自然也更好。

## 订阅下载

使用到的软件：RSSHub、qBittorrent

本来以为整套流程都需要自己处理，结果调研的时候发现RSSHub+qBittorrent就可以把整个流程完美的解耦后串联了起来。

ehentai原站提供的RSS功能比较单一，无法满足个性化订阅的需要，不过有了RSSHub就比较方便了，这里主要用到的是RSSHub中ehentai的搜索路由。路由参考RSSHub官网的说明，格式为 `/ehentai/search/:params?/:page?/:routeParams?`。可以先在ehentai网站上搜索一次，复制网址后面的搜索参数。这里额外补充一点，其中页码参数`page`是从0开始而非1。以及因为要用于后续下载使用，所以还需要把获取种子地址参数打开。

例如我会订阅所有评分为5分的汉化漫画和同人志，最后的路由为：`/ehentai/search/f_cats=1017&f_search=language%3AChinese&advsearch=1&f_sname=on&f_stags=on&f_sr=on&f_srdd=5/0/bittorrent=1`。

qBittorrent在server端需要额外安装qBittorrent-nox来提供web管理功能，稍微需要注意，qBittorrent-nox的版本不能太低，否则是没有rss订阅功能的。把订阅地址扔到qBittorrent的RSS订阅里，配置好自动下载，整个流程就算完成了。本来还想着对于没有提供种子的漫画需要额外处理一下走http下载，搜了一下近半年的汉化漫画都提供种子文件，决定暂时先不折腾这块了。

当然这样一来下载的漫画就不方便在ehviewer里看了，需要额外的阅读工具，这部分在下面详述。

## 额外的EX

部分资源只在里站exhentai有，所以最好还是配置从里站下载。不过这比表站就要多几步折腾。

RSSHub本身支持检索exhentai，只需要在环境变量中添加`EH_IPB_MEMBER_ID`, `EH_IPB_PASS_HASH`, `EH_SK`, `EH_IGNEOUS`这四个参数，就会自动改成从里站爬取信息。这里有一个小坑，添加上述变量后，访问ehentai和exhentai，会采用用户的个性化配置，可能会对网页元素造成影响，影响爬取。可以在网站的个人设置中，添加一个默认的profile供RSSHub使用。

另外一个麻烦的问题是，exhentai只能在登录后访问，所以输出的exhentai的种子文件地址，在qBittorrent中是无法下载的。搜了一圈qBittorrent好像也没有提供在下载种子文件的时候配置自定义header这么偏门的功能。

RSSHub ehentai路由的作者也许在其他场景下碰到了这个问题，因此提供了一个`EH_IMG_PROXY`参数，用于替换生成的RSS文本中图像的链接为代理服务器地址，从而在RSS阅读器中可以看到exhentai上的封面，不过他可能没想过把这个参数应用在种子下载地址上……

这个思路倒是可行，在代理服务中手动配置Cookie到header中，应该可以解决exhentai无法访问的问题，遂尝试在nginx上添加一项代理配置

```nginx
server {
listen       80;
server_name  exhentai.<HOMELAB_DOMAIN>;
location / {
proxy_pass https://exhentai.org;
proxy_set_header   Host        exhentai.org;
proxy_ssl_server_name on;
proxy_set_header   X-Real-IP   $remote_addr;
proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header   Cookie      <EXHENTAI_COOKIE>;
    }
}
```

测试了一下使用代理地址下载种子文件，没有问题，看来方法可行。

理论上可以把server_name直接配置成exhentai.org，然后在hosts或者自定义的dns中把这个域名指向homelab的内网ip，这样不需要修改RSS输出结果就能让qBittorrent通过反向代理下载到种子文件。不过exhentai的下载地址使用了https，虽然不清楚qBittorrent和ehentai在保障https通信安全上做的如何，使用自签证书或许可以绕过去，不过我也不太想干这种左右互搏的事情。

所以为了让qBittorrent订阅到的地址指向代理服务器，本来最后一步应该是给RSSHub提个PR，把代理地址应用到替换种子地址上，不过这样多少要费些时日。临时性的又想了个法子，反正是要做反向代理的，索性在nginx的反向代理里加一项配置，把RSSHub的响应体中的exhentai域名直接替换成代理的域名。修改的配置也很简单：

```nginx
server {
listen       80;
server_name  rsshub.<HOMELAB_DOMAIN>;
location / {
proxy_pass http://<HOMELAB_DOMAIN>:<RSSHUB_PORT>;
proxy_set_header   Host    $host;
proxy_set_header   X-Real-IP   $remote_addr;
proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
sub_filter_once    off;
sub_filter_types   application/xml;
sub_filter "https://exhentai.org/" "http://exhentai.<HOMELAB_DOMAIN>/";
    }
}
```

## 管理阅览

我之前试过用**calibre**来做管理，calibre也有从ehentai[获取元数据的插件]([yingziwu/doujinshi_metadata_plugins: the calibre metadata plugins for doujinshi (github.com)](https://github.com/yingziwu/doujinshi_metadata_plugins))，把ehviewer上的漫画导进去之后，先编辑好基础的名称和作者信息，然后用插件从e站获取余下的tag等信息。整个管理流程是没有问题，不过体验也说不上来有多好，另外calibre基本只能解决管理的问题，阅览的需求需要另外想办法。

本以为没有更好的方案就只能把这两步分开，结果意外发现[Lanraragi](https://github.com/Difegue/LANraragi)用起来还不错。在管理功能上，比起calibre更适合做漫画和同人志的管理，也有插件从ehentai获取元数据。

此外在阅览方面也做的不错，网页端的功能算不上很完善但是够用，而且非常难得的有tachiyomi插件，移动端的体验也比较好。

因为Lanraragi和tachiyomi插件都是最近才开始用，暂时还没有什么大问题，后续折腾的时候碰到什么问题再单独更新吧。因为元数据都在本地，后期即使要迁移到其他管理方案上，也具有一定可行性。

唯一比较麻烦的是这个应用的后端是Perl写的，看起来有点头疼，希望后续不会有需要钻进去改代码的时候。

使用中的一个小插曲，下载的漫画我是放在nas上通过smb挂载到运行Lanraragi的虚拟机上，本想着数据库也直接存在nas上方便备份，结果应用一启动硬盘吵的像破锣似的，但是又没有大量读写。我本以为是因为频繁需要扫描文件变更来做更新，感叹以后漫画本子只能存ssd了，后来看了日志发现是因为smb权限问题导致Redis AOF持久化失败了疯狂重试导致的，算是给我刚服役不久的两块机械盘留了条活路。
