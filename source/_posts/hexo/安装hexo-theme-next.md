# 目的

安装hexo-theme-next，美化hexo界面。

# 前置知识

[安装hexo](http://localhost:4000/2019/08/10/hexo/%E5%AE%89%E8%A3%85hexo/)。

# 步骤

1. 访问hexo-theme-next官网，[http://theme-next.iissnan.com/getting-started.html](http://theme-next.iissnan.com/getting-started.html)。
2. 前往安装NexT -> 下载主题，选择下载稳定版本，前往NexT版本[发布页面](https://github.com/iissnan/hexo-theme-next/releases)，我这边看到最新稳定版本是5.1.4，将源代码下载下来。
3. 解压所下载的压缩包至站点的 themes 目录下， 并将解压后的文件夹名称（`hexo-theme-next-5.1.4`）更改为 `next`。
4. 打开blog/_config.yml，将theme字段更改为next。
5. 在blog目录下，使用`hexo clean`清除缓存，然后使用`hexo s --debug`启动服务，看到如下图所示界面就表示hexo-theme-next安装完毕了。
   ![](https://aliyun.hellozjf.com:7004/uploads/2019/8/10/TIM截图20190810101951.png)