---
title: python项目流水线
date: 2019-12-31 08:04:37
permalink: python_project_pipeline
---

依旧是上次的[python 项目](/python_project), 周末抽时间把流水线建了, 顺便加了一个badge. python项目的流水线相对简单, 对于编译流水线来说, 其实只要跑一下测试就好了. 

这里我用的测试框架是pytest, 以azure流水线为例, yaml内容如下

```yaml
trigger:
- master
- dev
- azure

pool:
  vmImage: 'ubuntu-latest'
strategy:
  matrix:
    Python36:
      python.version: '3.6'
    Python37:
      python.version: '3.7'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '$(python.version)'
  displayName: 'Use Python $(python.version)'

- script: |
    python -m pip install --upgrade pip
    cd taskcommander
    pip install .
  displayName: 'Install dependencies'

- script: |
    pip install pytest-azurepipelines
    pip install taskcommander/.[test]
    pytest --cov=./taskcommander/
  displayName: 'pytest'

- script: |
    pip install codecov && codecov  -t $(CODECOV_TOKEN)
  displayName: 'upload code coverage'
```

azure对于python项目支持多版本测试, 写一套流水线用多个python版本运行, 如上面的yaml中就写了3.6和3.7两个版本, 2.7因为一开始就没想着支持, 3.5因为对于**type hint**的支持不太一样, 因此这两个版本就去掉了.

步骤是非常简单的三步

1. `pip install`安装python包(其实不需要发布的话这步骤也可以去掉, 不过我觉得安装也算是一个测试环节吧)
2. `pip install`安装pytest以及测试需要的包, 并运行测试
3. 将代码覆盖率(code coverage)上传到codecov.

每个步骤都很简单, 就不展开讲了. 唯一需要注意的是上传codecov中用了一个环境变量. 这是因为流水线的配置yaml一般是写到代码库(github)上的, 一般不推荐把诸如用户/密码等敏感信息写到代码库里去, 因此用环境变量的做法更合适一些. 而且azure pipeline支持定义密码类环境变量, 对于此类变量, 在pipeline的输出日志里也会隐藏, 提供更好的保密性.

配置好以后, 每次提交代码到`master`、`dev`分支就会自动触发流水线运行, 如果运行报错, 也会有邮件提醒. 另外, 配置了codecov以后, 创建pr时也会自动提醒合并代码会带来的覆盖率变化.

最后是给项目readme加上codecov的badge, 个人感觉算是展示项目一个比较实用的badge. 在readme里加入类似以下内容, 就能正常展示badge了.

```markdown
[![codecov](https://codecov.io/gh/xdsoar/TaskCommander/branch/dev/graph/badge.svg)](https://codecov.io/gh/xdsoar/TaskCommander)
```

每个项目的badge可以在登录codecov后, 在项目的settings->badge中找到, 不需要自己敲.