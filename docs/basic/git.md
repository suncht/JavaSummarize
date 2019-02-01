#使用git将项目上传到github（最简单方法）
1. 进入Github首页，点击New repository新建一个项目
2. 在本地，把github上面的仓库克隆到本地
git clone https://github.com/****.git
3. 这样就可以使用IDE编辑器进行开发了
4. 提交代码，用以下命令代码 
    > git add .
    > git commit  -m  "提交信息"
    > git push -u origin master
5. 更新远程分支列表
    > git remote update origin --prune
    > git remote update origin -p
6. 切换分支（本地没有分支）
    > git branch -a
    > git checkout -b 分支名 origin/分支名
    切换分支（本地存在分支）
    > git checkout 分支名
7. 更新代码
    > git status
    > git pull
8. 合并代码(分支dev合并到分支master)
    > git checkout dev
    > git pull
    > git checkout master
    > git merge dev
    > git push -u origin master
