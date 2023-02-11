---
title: 博客升级 内容提炼
date: 2023-02-11
comments: true
---

趁着周末把博客折腾了一番，主要改了两个东西：

- 将博客内容托管到 Github
- 调整为自动化部署，简化发布流程

<!--more-->

在此之前的发布流程都是，将写好的文章以 markdown 文件形式存放到固定目录中，执行 hexo 命令，并将最后生成的产物推送到阿里云。

每次都这么搞一遍真的很麻烦，而且高度依赖我电脑配置的环境，放假在家或者是用别的设备都没法进行操作，之前有看过文章可以通过自动构建的方式简化这个过程。想着把这块捡起来好好弄一下，工欲善其事必先利其器，好的工具真的可以提高我的积极性。然后就开始搜资料，看别人是怎么搞得，照着别人的操作步骤一步步来就完了。虽然大家都是通过 Github Action 来实现自动构建，但是每个人的流水线都不同，也没有直接能照搬的例子。

既然没有，就只能自己动手了，主要的思路就是，将执行 hexo 命令、push 代码到阿里云的操作都交给 Github Action 执行，自己只负责写文章然后提交到 Github 仓库中，这样无论是在什么电脑上只要能访问 github 就可以，下面是我博客构建用的 yaml 文件。

```yaml
name: publish

on:
  push:
    branches:
      - gh-pages

jobs:
  publish-blog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 14.2.0
          registry-url: https://registry.npmjs.org/
          
      - name: config ssh to alicloud
        run: |
          mkdir -p ~/.ssh/
          echo '${{secrets.HEXO_DEPLOY_KEY}}' > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -t rsa ${{secrets.CLOUDIP}} >> ~/.ssh/known_hosts
          
      - name: install hexo
        run: npm i -g hexo-cli
        
      - name: install hexo dependencies
        run: npm i
        
      - name: hexo init cmd
        run: hexo init blogs

      - name: install theme
        run: git submodule add https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
        working-directory: blogs
      
      - name: install render for theme
        run: |
          npm install hexo-renderer-pug --save
          npm install hexo-renderer-sass --save
          
      - name: cp theme config
        run: |
          cp /home/runner/work/myblogs/myblogs/_config.yml /home/runner/work/myblogs/myblogs/blogs/_config.yml
          cp /home/runner/work/myblogs/myblogs/theme_config.yml /home/runner/work/myblogs/myblogs/blogs/themes/maupassant/_config.yml
        
      - name: cp blog post
        run: cp -r /home/runner/work/myblogs/myblogs/posts/* /home/runner/work/myblogs/myblogs/blogs/source/
                
      - name: remove hello word file
        run: rm /home/runner/work/myblogs/myblogs/blogs/source/_posts/hello-world.md
        working-directory: blogs/source/_posts
        
      - name: cp theme images
        run: cp -r /home/runner/work/myblogs/myblogs/theme_img/* /home/runner/work/myblogs/myblogs/blogs/themes/maupassant/source/
      
      - name: generate
        run: hexo g
        working-directory: blogs
        
      - name: init git repo
        run: |
          git config --global user.name clamyang
          git config --global user.email ybq2888@163.com
          git config --global init.defaultBranch master
          git init
          git add .
          git commit -m "deploy!"
          git remote add origin git@${{secrets.CLOUDIP}}:/home/git/test.git
          git push --force origin master
        working-directory: blogs/public

```

因为对博客进行了额外的配置，所以要把配置文件，文章，图片提前放到 Github 仓库中，构建的时候复制到指定目录。

> 注意，不要把服务器密码等信息直接写到 workflow 中，可以在仓库中添加 secret 使用 `${{}}` 的方式引用进来。

这篇文章就是通过 Github Action 进行发布的，用起来确实舒服了很多。还发现可以通过 StackEdit 进行在线编辑，可以关联到 GitHub 仓库中的文件，写好后直接点击同步就能发布到博客上，只不过只能在网页上写文章，比起便利性和舒适性我还是选择舒适，写好后 git add commit push  三件套，也不是很麻烦。

我对博客的内容进行了整理，暂时删掉了很多旧文章，原因很简单，不想再以量取胜，看着 Github 上一次次的 commit 记录确实很爽，加上前期博客刚搭建好，搞了很多半截子的文章，但是回看那些内容的话，在深度和完整性上还差很多，正好借着这次机会筛选一波，选了几篇还比较满意的留下，剩下的文章待我完善后再放上来。



