# GitHub Action 自动部署 React 应用 

本文以 Ant Design Pro 为例，Github Action 中包含了编该项目所需的一些步骤，请视情况进行修改。

## 注意：

1. 创建的 OSS 需要**公开读**权限。
2. 启用**静态网站托管**模式：打开 OSS 设置-静态页面，配置为如下内容： 
* 默认首页：index.html
* 默认 404 页：index.html
* 子目录首：已开通
* 文件 404 规则：Redirect
3. 推荐为 OSS 启用 CDN 加速，在此不赘述。
