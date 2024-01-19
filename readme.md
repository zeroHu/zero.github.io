# 新增

hexo new {文件名}

# 本地启动

hexo serve

# 发布

hexo clean && hexo d

hexo d 失败了多执行几次。

# 分支解释

1、front 分支是用于写 md 文档的，写博客的分支
2、main 分支是用于发布 md 文档为 html 的线上地址 `https://zerohu.github.io/`的源码的分支 (不用管提交代码，因为 hexo d 的操作会自动将\_config.yml 中的 deploy 下的配置信息发布到 main 分支)
