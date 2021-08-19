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
- 主版本号,eg:v3 v2
  - 按语义版本2.0的规范,此仓库至少包含两个不兼容的主版本
  - go.mod需要表明自己引用的是哪个版本,不带主版本号表示用v1
  - 主版本号后缀的规则是:如果新旧两个包都使用同一个import路径,则新包必定向后兼容旧包
    - 这也是为什么需要将v2/v3等添加到import路径中,主要是方便区分不兼容的多个版本
  - v0是不稳定的,v1是最有可能做到向后兼容的版本,所以主版本号不需要v0和v1
    - 特例:gopkg.in需要一直带主版本号,eg:gopkg.in/pion/webrtc.v3
    - 后缀不是/,而是.
    - gopkg.in是一个路由站,针对github仓库实现的
  - 利用主版本号,可在一次编译中,同时使用到不同的主版本
- 包是如何添加到module的(下面是解析过程)
  - 这里的解析过程特指:使用go命令+package path的解析
  - (01)用package path的前缀(即包对应的module)去build list中查找
    - build list解释:
      - 是整个module图
      - 是所有依赖module最小版本的集合
  - (02)在build list中找到了pakcage前缀对应的module
    - 如果找到了,go命令会进一步检查package是否存在这个module中
      - package带的子目录(如果package带子目录)是否存在module中
      - 是否有go源码存在于子目录中
        - go build的约束不适用于这儿
    - 如果确认了module提供要找的package,就使用这个module
    - 如果module不提供要找的package;或有多个module提供要找的package,go命令会报错
      - flag `-mod=mod`会让go命令试图去找个新的module,并更新对应的go.mod/go.sum
        - go get/go mod tidy 默认就带了`-mod=mod` flag
  - (03)go命令要为package找个新module(即在build list中未能找到提供package的module)
    - 第一步要确认代理信息(goproxy环境变量)
      - goproxy是由代理url列表和direct/off组成,用逗号隔开
        - 代理url都用goproxy协议通信
        - direct表示直接和版本控制系统(eg:git)通信
        - off表示不应该尝试通信
        - goprivate/gonoproxy两个环境变量也对此处的行为有影响
    - go命令会针对代理信息来找module(如果失败,就对下一个代理信息查找)
      - 具体做法是将package path的各个前缀拿去代理处请求(并行的)
      - 失败的标志是所有请求都失败了(404/410);或module中并不提供指定的package
      - 成功的标识有两点:
        - 一是有请求成功
        - 二是go命令会对所有成功的请求,下载对应的module,并在module中找到了package
      - 如果有多个请求成功了,最长路径的module就是要找的
  - (04)如果go命令为package找到了新的moudle,则更新主moudle go.mod的require指令
  - 通过由package找module的规则,确保了一点:未来加载了一个相同的package,这个package和老package一样,都会使用同一个module
  - 如果一个已解析的包并没有被主module直接导入,则需要添加注释 // indirect
    - 有两种情况会发生这种情况:
      - 直接依赖A未启用go module,那么A依赖的BCD就会在主moudle中出现,并添加 // indirect
      - 直接依赖A启用了go module,但A依赖的D未添加到A的依赖中,也需要在主module中依赖D,并添加 // indirect
    - 当所有go项目都迁移到go module后,就不会出现间接依赖的注释了
    - go mod why -m 依赖 可查找依赖链
    - go mod why -m all 可查找所有依赖链
