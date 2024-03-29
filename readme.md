# hexo-blog
个人博客记录

# 使用方式
## 环境搭建
- 安装nodejs  
    - 直接按照官网安装即可  
- 安装hexo  
    ```js 
    npm install -g hexo-cli
    ```
- 创建项目
    ```js
    hexo init hexo-blog
    cd hexo-blog
    npm install  
    ```
- 已有项目
    ```js
    git clone xxx
    cd xxx
    npm install
    ```
- hexo本地启动   
    ```js
    hexo g
    hexo s
    ```  

## 主题切换
- clone自己喜欢的主题到theme文件夹，或者直接下载拷贝都行  
- 修改_config.yml  
    ```
    theme: xxx
    ```
- 重新构建启动服务  
    ``` js 
    hexo g
    hexo s
    ```
- 官方主题地址: [https://hexo.io/themes/](https://hexo.io/themes/) 
- Fluid地址: [https://github.com/fluid-dev/hexo-theme-fluid](https://github.com/fluid-dev/hexo-theme-fluid)

## 新增文章  
- 设置资源文件夹
    ``` js
    post_asset_folder: true
    ```
- 新建文章
    ``` js
    hexo new post 新文章
    ```
- 编辑新文章的md文件
- 重新生成```hexo g```即可

## 删除文章
- 删除source/_post文件夹下的文件
- 重新执行```hexo g```构建即可

## 发布GitHub Pages
- 方案一 【不建议使用】
    - 将hexo-blog项目下的public文件下copy到github.io的仓库中，然后再进行推送。
    - 每次更新后需同步新仓库，比较麻烦不建议使用
- 方案二  
    - 安装hexo-deployer-git ```npm install hexo-deployer-git --save```
    - 修改跟目录下的_config.yml, 配置github信息 
    -  ```hexo g / hexo d```即可
    ```
    deploy:
    type: git
    repo: https://github.com/wujiyang/wujiyang.github.io.git
    branch: master
    token: xxxxxxx
    ```
    ![](token.png)

## Fluid使用手册

[https://hexo.fluid-dev.com/docs/guide/](https://hexo.fluid-dev.com/docs/guide/)