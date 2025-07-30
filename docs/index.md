使用 MkDocs
# Welcome to MkDocs

For full documentation visit [mkdocs.org](https://www.mkdocs.org).

## Commands

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

## Project layout

    mkdocs.yml    # The configuration file.
    docs/
        index.md  # The documentation homepage.
        ...       # Other markdown pages, images and other files.

## 使用技巧与最佳实践
### 1. 手动添加特殊项目
在自动生成的导航中手动添加特殊链接：
* [Java 官方文档](https://docs.oracle.com/en/java/)
* [Spring 框架指南](https://spring.io/guides)

### 2. 添加图标增强可读性
使用表情符号作为分类图标：
## 🧠 知识体系
## ⚙️ 运维部署
## 💡 面试技巧

### 3. 创建最近更新板块
## 🆕 最近更新
* [2023-11-15] [Spring Boot 3 新特性](knowledge/java/spring-boot-3.md)
* [2023-11-10] [Redis 集群配置](knowledge/database/redis-cluster.md)


### 4. 条件性显示内容（高级）
在 Markdown 中使用 HTML 注释控制显示：
```
<!-- if version > 1.2 -->
* [新特性介绍](features/new-in-v1.2.md)
<!-- endif -->
```


### 常见问题解决
#### Q: 导航未更新怎么办？
```
# 强制重新生成
python generate_nav.py --force

# 检查文件权限
chmod +x generate_nav.py
```

#### Q: 某些文件不想出现在导航中？
在生成脚本中添加排除规则：
```
# 强制重新生成
python generate_nav.py --force

# 检查文件权限
chmod +x generate_nav.py
```


#### Q: 如何自定义排序？
在 SUMMARY.md 中手动调整顺序
或在脚本中添加优先级逻辑：
```
# 在 generate_nav.py 中
priority_files = {
    'index.md': 0,
    'getting-started.md': 1
}

files = sorted(files, key=lambda x: priority_files.get(x, 999))
```

使用 mkdocs-literate-nav 后，您的知识库维护流程简化为：
创建新 Markdown 文件
运行 python generate_nav.py（或等待 CI 自动运行）

提交并推送更改