---
title: 使用Terraform管理云服务
date: 2022-07-13 09:18:42
permalink: terraform_for_cloud_service/
tags:
- cloud service
- IaC
---

## 前言其一

本篇博客写作之时，本站的域名已经从 <https://blog.goatman.me> 切换到 <https://blog.xdsoar.net>，原因是为了将站点迁移到国内，只好更换了可以在国内备案的域名。原域名已经做了301重定向处理，不过还是建议看到这里的读者更新一下地址。

使用国内的对象存储服务来做静态站点托管有一个奇怪的矛盾。做静态站点托管，我需要有一个已备案的域名；要备案域名，我需要购买云主机；如果我购买了云主机，那么为什么我还需要对象存储做静态站点托管？

对象存储做静态站点是我特别喜欢的公有云服务，价格低廉又安全可靠，但为此单独买一台云主机着实让我受不了。直到最近发现阿里云购买函数服务包也可以享受代备案，此事才算了解。

## 前言其二

这个博客的最早的几篇博文，写的就是当初在s3上部署的流程，在那之后的几个实验项目我也都是按照当时博文里写的流程来做发布。不过最近几年里，我也在思考能不能更进一步，让整个流程以更顺畅的方式进行。CI工具已经打通了从代码到制品的全过程，代码到服务的最后一公里，或许就要靠IaC工具来实现了。

去年我花了不少时间研究aws的`CloudFormation`，老实说看的我云里雾里。折腾了半天也没搞明白，想了想这东西可能真的不适合个人使用。直到熟悉IaC的朋友给我推荐了`Terraform`才算找到了条可行的路子。

本身迁站是个很繁琐的事情，结合Terraform来做，本是可以省不少事的。不过实际使用下来，terraform与阿里云的衔接还是有不少坑，所以实际并没有那么顺利。好在这种事情折腾一次，后面的项目都可以复用，还算是比较便利。

## 关于Terraform

Terraform是HashiCorp创建的IaC工具，在IaC领域目前姑且算是较为成熟的产品。从我短暂的使用体验上来说，算不上优秀，只能说是够用的程度，当然这可能也和我使用的是阿里云的provider有关。因为阿里云的配套设施对比aws确实算不上成熟，一些配置项在控制台页面可以配置，但是在terraform中无法配置。我本来以为是provider没有实现，想着实在不行就提pr加一下，都拉了代码准备动手了，结果阿里云的API本身就没有提供这块的配置项……

因为目前我没有使用Terraform来管理aws，所以aws的provider姑且不评价。不过主观评估，这类第三方工具通常是很难做到完全覆盖第一方产品的功能的。

从动机上考虑，云服务厂商恐怕更希望提供独有的无可替代的服务。而目前除了aws s3成了对象存储的事实标准以外，各类云服务其实并没有特别统一的标准。因此即使Terraform声称支持众多云服务提供商，也不可能做到一次编写，到处运行的效果，想要把已有的aws上的服务，改一改provider就迁移到阿里云上，依旧是做不到的。

那么IaC的作用是什么呢？我个人认为是为基础设施提供更好的运维方式。特别是采用声明式的管理方式，可以确保基础设施的状态是受控的，可感知的。当然前提是能够完全通过IaC工具进行基础设施管理，使它成为单一事实来源。

Terraform的官方文档比较冗长，为了快速上手，我参考的是[阿里云的文档](https://help.aliyun.com/document_detail/111911.html))。下面我以本次博客部署为例，讲述使用Terraform管理基础设施的全过程。

## 开始之前

开始之前，需要做如下准备：

* 安装Terraform命令行客户端
* 创建阿里云用户对应的`access key`，并在RAM中赋权。
* 购买一个域名

其中域名或许也可以通过Terraform创建，不过域名毕竟不像其他资源那么随意，所以我没有使用Terraform管理。第二步原则应该创建一个IaC专用的用户并设计权限范围，个人使用并没有那么讲究，我就直接使用了实名用户。

准备好之后就可以使用Terraform进行资源创建了。这里再啰嗦两句，使用Terraform的过程中会产生一些记录资源状态的文件(tfstate)，又或者是创建的用户密钥文件，如果有团队协作的需求，使用对象存储是个好办法，别忘了**注意访问权限**。个人使用来说，放到一个私人的git仓库当然也是可以的，同样要注意**不能放到公开仓库里**。

至于一些可复用的配置模板或者是分享给别人用的参考配置之类的，可能gist是个不错的选择。

## 配置对象存储与授权

首先厘清一个静态博客站点需要用到哪些云服务

* 一个oss bucket用于存储静态站点的文件
* 一个dns记录，作为博客域名
* 一个用户，以及对应的RAM，用于CI部署制品到OSS
* （可选）一个用于加速访问的CDN
* （可选）一个SSL证书，支持https访问

前三步基础配置其实很简单，参考如下

```hcl
provider "alicloud" {
  alias  = "hz-prod"
  region = "cn-hangzhou"
}

resource "alicloud_oss_bucket" "bucket-blog-log" {
  provider = alicloud.hz-prod
  bucket   = <bucket_name_for_log>
  acl      = "private"
  force_destroy = true
}

resource "alicloud_oss_bucket" "bucket-blog" {
  provider = alicloud.hz-prod

  bucket = <bucket_name_for_blog>
  acl    = "public-read"
  force_destroy = true

  # 配置CDN加速
  transfer_acceleration {
    enabled = true
  } 
  # 静态网站的默认首页和404页面
  website {
    index_document = "index.html"
    error_document = "error.html"
  }
  # 访问日志的存储路径
  logging {
    target_bucket = alicloud_oss_bucket.bucket-blog-log.id
    target_prefix = "log/"
  }

  # 防盗链设置
  referer_config {
    allow_empty = true
    referers    = ["http://<blog_url>","https://<blog_url>"]
  }
}


resource "alicloud_ram_user" "blog_user" {
  name         = "blog_deploy"
  display_name = "blog_deploy"
  mobile       = "86-18688866888"
  comments     = "devops user for blog"
  force        = true                   
}

resource "alicloud_ram_access_key" "ak" {
  user_name   = alicloud_ram_user.blog_user.name
  secret_file = "<storage_path>/accesskey.txt"
  # 保存AccessKey的文件名
}

resource "alicloud_ram_policy" "blog_policy" {
  policy_name        = "blog_oss_policy"
  policy_document    = <<EOF
  {
    "Statement": [
      {
        "Action": [
          "oss:*"
        ],
        "Effect": "Allow",
        "Resource": [
          "acs:oss:*:*:<bucket_name_for_blog>",
          "acs:oss:*:*:<bucket_name_for_blog>/*",
          "acs:oss:*:*:<bucket_name_for_log>",
          "acs:oss:*:*:<bucket_name_for_log>/*"
        ]
      }
    ],
      "Version": "1"
  }
  EOF
  description = "privilege for deploy to blog bucket"
  force       = true
}

resource "alicloud_ram_user_policy_attachment" "blog_attach" {
  policy_name = alicloud_ram_policy.blog_policy.policy_name
  policy_type = alicloud_ram_policy.blog_policy.type
  user_name   = alicloud_ram_user.blog_user.name
}

# 如果不需要使用独立cdn, 则将dns解析到oss专用的加速域名上
resource "alicloud_alidns_record" "blog_record" {
  domain_name = "<domain>"
  rr          = "<sub_domain>"
  type        = "CNAME"
  value       = "${alicloud_oss_bucket.bucket-blog.id}.oss-accelerate.aliyuncs.com"
  remark      = "blog domain"
  status      = "ENABLE"
}
```

整个文件中需要使用者自己定义的参数都以`<custom parameter>`的方式标出，应该说还是挺通用的。如果想新建一个别的站点，只要把整个文件复制一份，并填写相应的参数就好了，然后一条`terraform apply`就可以创建出来。

不过很遗憾的是，做完上面这些步骤之后，还有两项设置，只能在控制台进行。分别是设置bucket可以访问的域名，以及设置子页面的首页，即访问`aaa.com/bb/`时展示`aaa.com/bb/index.html`的内容。由于这两项配置甚至连阿里云的API都没有提供，目前看来是很难实现了。我能想到的就是在hcl中声明，当bucket被创建的时候，打印一条消息，提醒需要手动执行的这两个步骤了……

除了这两项做不了的配置，整体过程应该说还是很顺畅的，配置完成后，本地也有了访问对象存储的密钥，先本地做一趟实验把命令跑通，再修改博客对应的流水线脚本，整个迁移过程就算完成了。

## 配置SSL证书

其实按上面的流程走完，OSS内建的加速方案也已经开启了，原则上就只有配置SSL证书这一件事情要单独做，偏偏这件事情做起来又不是那么方便。经过前面对OSS相关API的摸排，我确信虽然控制台上提供了配置SSL证书的步骤，但是这个过程无法通过Terraform完成。如果觉得麻烦，这步也可以在控制台完成。

这一环节的另一复杂性在于，如果不想用阿里云的免费证书，使用诸如`acme.sh`之类的工具申请的免费证书只有3个月的有效期，为了能让证书自动更新，那么还需要定时作业来更新证书。简而言之，不是很有必要，而且很复杂，如果真的想干，那么就接着往下吧……

首先是再创建一个用户，并赋予更新dns record权限，用于申请证书。

```hcl
provider "alicloud" {
  alias  = "hz-prod"
  region = "cn-hangzhou"
}


resource "alicloud_ram_user" "ssl_user" {
  name         = "sslUser"
  display_name = "sslUser"
  mobile       = "86-18688866888"
  comments     = "for get ssl cert"
  force        = true
}

resource "alicloud_ram_access_key" "ak" {
  user_name   = alicloud_ram_user.ssl_user.name
  secret_file = "accesskey.txt" # 保存AccessKey的文件名
}

resource "alicloud_ram_policy" "ssl_policy" {
  policy_name     = "ssl_policy"
  policy_document = <<EOF
  {
    "Statement": [
      {
            "Action": "alidns:*",
            "Resource": "*",
            "Effect": "Allow"
        }
    ],
      "Version": "1"
  }
  EOF
  description     = "privilege for ssl"
  force           = true
}

resource "alicloud_ram_user_policy_attachment" "ssl_attach" {
  policy_name = alicloud_ram_policy.ssl_policy.policy_name
  policy_type = alicloud_ram_policy.ssl_policy.type
  user_name   = alicloud_ram_user.ssl_user.name
}
```

上面的配置里没有一个变量，属于拿来即用的简单配置。不过这里我分配的权限偏大了一点，给予了账户下DNS解析服务的所有权限，追求权限分配最小化的话可以更细化一点。

关于`acme.sh`的使用这里就不展开叙述了，网上相关资料很多，用**acme.sh alidns**做关键词就能搜到很多，比如[这篇](https://f-e-d.club/topic/use-acme-sh-deployment-let-s-encrypt-by-ali-cloud-dns-generic-domain-https-authentication.article))，你会发现这篇文章里占了较大篇幅的用户权限配置部分，在用了Terraform以后可以完全被上面的配置实现，有没有感受到一点IaC带来的便利？

这样就拿到了免费的证书，可以开始正式配置了。

由于CDN的配置中可以指定，所以可以通过重新配置CDN并指定SSL证书实现https。如果要进行以下配置，记得先在之前配置bucket的文件中，去掉关于dns记录的配置，并再次`terraform apply`，当然如果你想使用一个不一样的域名用于CDN的话，也是可以的。具体如下

```hcl
provider "alicloud" {
  alias  = "hz-prod"
  region = "cn-hangzhou"
}

resource "alicloud_cdn_domain_new" "domain" {
  domain_name = "<cdn_blog_domain>"
  cdn_type    = "web"
  scope       = "global"
  sources {
    content  = "<oss_bucket_url>"
    type     = "oss"
  }
  certificate_config {
    server_certificate = <ssl_cert>
    private_key = <ssl_key>
    cert_name = "terra_ssl_cert"
    cert_type = "upload"
  }
}

resource "alicloud_alidns_record" "blog_record" {
  domain_name = "<domain>"
  rr          = "<sub_domain>"
  type        = "CNAME"
  value       = "${alicloud_cdn_domain_new.domain.cname}"
  remark      = "blog domain"
  status      = "ENABLE"
}
```

这里有一个小细节，<oss_bucket_url>这个参数理论上可以从上一个关于oss bucket的配置中拿到。不过这个CDN的配置我单独拆了一个文件，原因是这个CDN的实现并不理想。如果按照上面的配置做一次`terraform apply`是没有问题，但是如果要做修改的话（其实相当于destroy再create），失败的可能性相当大。这主要是因为CDN这个资源的一致性较差，destroy后需要等待一段时间后才能使用重新创建。

而因为SSL证书有效期的问题，每隔几个月，就要手动destroy，等待一段时间后再apply。当然因为整个流程其实就是几个命令的事情，加一个cron任务，自动执行也不是多难的事情，倒也还算可以接受。另一个瑕疵点在于，删除了dns记录和cdn后，一段时间内这个域名就无法访问了。从可用性角度考虑，更新时采用AB两个CDN实例进行蓝绿切换会更合适。不过我这只是一个简单的博客站点，就不额外考虑这么多了。

## 后续

前面写了好几篇关于家用服务器（home server）的内容，但其实公有云也有很多实用又实惠的用法。但是记录过程要一页一页的截图，一些实验性质的操作做完再重复一遍也很麻烦，有了IaC工具以后确实方便了很多。后续我会把这些有趣的尝试记录下来，也算是一个系列吧。