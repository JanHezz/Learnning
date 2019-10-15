---
  title: 记一次git fork的经历
  date: {{ date }}   
  categories: ['工具']
  tags: ['git']       
  comments: true    
  img:             
---

本篇首发于[橙寂博客](http://www.luckyhe.com/post/44.html)转载请加上此标示。
本人对于git只会简单操作所以才有了这篇文章

缘由:在码云上一个开源项目fork了后，然后本地开发了提交了代码，原作者把我的代码分支合并了，这时候想要自己的代码跟原作者保持一致，于是于就有了这么一次经历
我询问了我一些同学，包括看了一些文章，想看我操作的请听我细细道来（悲惨的是，我一段操作后，他告诉我直接在码云上点击重新同步就行）。


## 优秀博客
记一次fork的经历参考的这一篇文章
[git远程fork总结](https://www.jianshu.com/p/d73dcee2d907)

#### 假设远程源仓库为A，自己fork后的远程仓库为B，自己本地的代码仓库为C1.

给 fork 配置一个 remote一般来说从自己远程仓库B去拉代码后就会有remote使用 git remote -v 查看远程状态。

```
git remote -v
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
```
#### 2. 添加一个将被同步给 fork 远程的上游仓库A
```
git remote add upstream https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git
```

再次查看状态确认是否配置成功。

```
git remote -v
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)
```
#### 3. 执行同步fork操作从上游仓库A fetch 分支和提交点，传送到本地，并会被存储在一个本地分支 upstream/master
```
git fetch upstream,默认会将远程所有的分支fetch下来remote:
Counting objects: 41, done.
remote: Compressing objects: 100% (41/41), done.
remote: Total 41 (delta 17), reused 0 (delta 0)
Unpacking objects: 100% (41/41), done.
From https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY
 * [new branch]      dev        -> upstream/dev
 * [new branch]      master     -> upstream/master
```
#### 4.将upstream的代码合并到本地仓库C上

4.1 分支切换&同步自己的远程仓库首先在本地代码C上从master创建分支
这一步是创建分支。如果是个人开发不需要。直接拉取到master 合并就好了。
```
git checkout -b dev
Switched to a new branch 'dev'

```
git checkout 这个命令是改变工作区 执行了上一步那么你的工作区就是dev。

执行git pull origin master 从自己的远程仓库B上拉取最新的代码到自己分支上
```
git push --set-upstream origin dev
这一步是把本地分支上传到远程B相当于操作 git push -u  origin  dev
```

4.2 执行合并upstream操作使用git merge upstream/master命令，把 upstream/master 分支合并到本地 **工作分支**  如果没改变默认就是master。
如果想同步远程仓库A非master源的代码到自己仓库B上来，比如说远程远有两个分支，master和dev则
合并非master分支的代码, 使用git merge upstream/dev命令，把 upstream/dev 分支合并到本地 dev 上这样就完成了同步，并且不会丢掉本地修改的内容。
```
git merge upstream/master
 Updating a422352..5fdff0f
 Fast-forward
  README                    |    9 -------
```
  - git merge a b
  这个命令就是合并分支，也就是说把a的代码合并到b。b如果没值默认是当前工作分支。  

#### 5.push本地代码到自己的远程仓库执行
git push origin master将本地仓库master分支的代码推送到自己远程仓库B的master分支上
或者git push origin dev:dev将本地仓库dev分支的代码推送到自己远程仓库B的dev分支上
