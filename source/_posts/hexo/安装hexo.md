# 目的

从零安装hexo博客框架。

# 前置知识

[安装nodejs](http://localhost:4000/2019/08/10/nodejs/%E5%AE%89%E8%A3%85nodejs/)。

# 步骤

1. 访问hexo官网，[https://hexo.io/zh-cn/](https://hexo.io/zh-cn/)
   ![](https://aliyun.hellozjf.com:7004/uploads/2019/8/10/TIM截图20190810095257.png)

2. 依次在自己电脑上执行官网最下面的几条命令

   ```
   npm install hexo-cli -g
   hexo init blog
   cd blog
   npm install
   hexo server
   ```

3. 敲完以上命令之后，访问，看到下图所示的界面，就说明hexo安装成功了
   ![](https://aliyun.hellozjf.com:7004/uploads/2019/8/10/TIM截图20190810100705.png)

# 问题

## ENOENT: no such file or directory

### 问题描述

```
npm WARN checkPermissions Missing write access to C:\Users\hellozjf\AppData\Roaming\npm\node_modules\hexo-cli\node_modules\is-number\node_modules\kind-of
npm WARN checkPermissions Missing write access to C:\Users\hellozjf\AppData\Roaming\npm\node_modules\hexo-cli\node_modules\is-number\node_modules
npm ERR! path C:\Users\hellozjf\AppData\Roaming\npm\node_modules\hexo-cli\node_modules\is-number\node_modules\kind-of
npm ERR! code ENOENT
npm ERR! errno -4058
npm ERR! syscall access
npm ERR! enoent ENOENT: no such file or directory, access 'C:\Users\hellozjf\AppData\Roaming\npm\node_modules\hexo-cli\node_modules\is-number\node_modules\kind-of'
npm ERR! enoent This is related to npm not being able to find a file.
npm ERR! enoent

npm ERR! A complete log of this run can be found in:
npm ERR!     C:\Users\hellozjf\AppData\Roaming\npm-cache\_logs\2019-08-10T01_56_29_641Z-debug.log
```

### 解决办法

删除`C:\Users\hellozjf\AppData\Roaming\npm\node_modules\hexo-cli`