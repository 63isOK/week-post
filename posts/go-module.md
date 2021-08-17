# go mod 学习

主要分以下几部分来学习:

- go module [ref](https://golang.google.cn/ref/mod)
- 开发过程中会遇到的所有问题
- 网友笔记,这部分主要是查漏补缺

## go module reference

go module用来管理依赖的,先了解定义.

### module/package/version的定义

- module, 一些列包,特征是她们一起发布,带版本
  - 可从仓库或module代理服务直接下载
  - module由module path标识,也就是go.mod里的一行
  - module的根目录,就是包含go.mod的目录
  - main mudle的目录,就是go命令执行目录
- package,同一目录下的源文件集合,特征是一起编译
  - module内的包
  - package path = module path + (相对于module根的)子目录
- module path, 模块路径,module的标准名称
  - 在go.mod中由module指定声明
  - module path是模块内所有包的package path的前缀
  - module path通常会包含一个仓库的根目录
    - 如果主版本号>=2,还需要包含主版本后缀,这个是必须的
      - 两个意思
        - 小于版本2的,都不需要带主版本后缀, 1.9.9版本也不需要带
        - eg: module github.com/pion/webrtc/v3 主版本后缀是v3
    - 其次大部分module都定义在仓库根目录,所以module path和仓库地址可以保持一致
    - 如果module不再仓库的根目录,那么具体的子目录就是module path
      - 此时具体的子目录是不需要带主版本后缀,至于module path要不要带"v2"就看版本号是否等于2
  - 反过来,如果module path以(eg:"v2")结尾,那么v2就有可能是
    - 主版本后缀
    - 子目录中本身就带v2目录
- version, 标识了module的一个不可变的快照
  - 可能是发布版本, eg: v1.2.3
  - 可能是预发布版本, eg: v1.2.3-beta4, 或是 v1.2.3-pre
  - 版本号以v开头,后面接的是"语义版本2.0"
    - 在语义版本2.0中,分主版本.次版本.修订版本-预发布+build元信息
    - 在版本比较时,build元信息是被忽略的
    - go.mod可以出现build元信息,+incompatible表示非module项目的主版本到了2或以上
    - 什么样的项目才不会使用module?
      - 不再维护的项目
      - 云厂商的sdk:一直希望开发者用最新版本,破坏性变更较多;开发者也必须用最新的
  - go命令会自动将"tag/branch/commit号"转换成对应的标准版本号
    - 首先会移除build信息(+incompatible除外)
    - 预发布版本的信息会替换成"提交号+时间戳"
- 预发布版本
  - 写法类似于 v0.0.0-20191109021931-daa7c04131f5
  - 有3部分:
    - vX0.0.0或vX.Y.Z-0;如果是没有tag,是vX.0.0
    - 时间戳,年月日时分秒
    - 提交号,git的前12位,svn的0填充的提交号
  - vX.0.0-yyyymmddhhmmss-abcdefabcdef 适用于没有基础版本,只有X符合主版本号
  - vX.Y.Z-pre.0.yyyymmddhhmmss-abcdefabcdef 基础版本是vX.Y.Z-pre
  - vX.Y.(Z+1)-0.yyyymmddhhmmss-abcdefabcdef 基础版本是vX.Y.Z
    - 基础版本是 v1.2.3, 则下一个预发布版本是v1.2.4-0.20191109021931-daa7c04131f5
  - 多个预发布版本可以指向同一提交,只是他们的基础版本不一样:只有在预发布之后对低版本进行tag操作,才会有这种情况
  - 整个预发布版本的逻辑保证了:预发布的版本比所有版本高,但比未来所有版本低
  - 相同基础版本的预发布版本,在比较时按时间顺序比较
  - 依赖时间戳是防止基于预发布机制做处的攻击行为
  - 预发布版本会由工具自动维护,不需要手动编写
