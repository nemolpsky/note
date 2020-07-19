### 1. IDEA配置Git
点击```File -> Settings -> Version Control -> Git```选项，配置Git执行路径，点击Test能成功显示版本号即可。


---

### 2. 基本使用
- 生成公钥秘钥

  要连接远程仓库必须在远程仓库配置秘钥，执行下面的命令会自动生成公钥和秘钥，```id_rsa```是私钥，```id_rsa.pub```是公钥，一般在C盘的用户目录里的```.ssh```目录，在远程仓库配置公钥，例如GitHub中的Add SSH Key，这样才可以远程连接远程仓库。

   ```
   ssh-keygen -t rsa -C "test@example.com"
   ```
- 从远程库克隆
  
  Git支持```Http```协议和```SSH```协议，一般都使用```SSH```协议，例如下面这个就是一个在GitHub上的SSH协议的地址，可以使用```git clone```命令将这个远程仓库中的代码下载到本地。

  ```
  git@github.com:nemolpsky/TestGit.git
  ```
  
  在桌面或任意文件夹内点击右键，选择```Git Bash Here```即可打开Git的命令窗口，选择路径，比如下面选择了```/e/linux```路径，执行```git clone```命令就可以将代码下载到本地，然后使用IDEA打开。

  ```
  $ pwd
    /e/linux

  $ git clone git@github.com:nemolpsky/TestGit.git
    Cloning into 'TestGit'...
    remote: Enumerating objects: 183, done.
    remote: Counting objects: 100% (183/183), done.
    remote: Compressing objects: 100% (128/128), done.
    remote: Total 183 (delta 33), reused 161 (delta 14), pack-reused 0
    Receiving objects: 100% (183/183), 17.46 KiB | 1024 bytes/s, done.
    Resolving deltas: 100% (33/33), done.
  ```
  
  或者直接使用IDEA克隆也可以，导入项目时选择```Get from Version Control```即可，然后同样填入上面的地址，再选择路径也是可以直接克隆代码到本地。

- 查看分支状态

  使用git status可以看到分支的状态，未添加或者未提交的文件等等。
  ```
  $ git status
  ```
- 查看当前分支

  可以使用命令查看，IEDA的右下角也会直接显示，比如现在本地代码是指向的master分支。
  ```
  $ git branch
    branch1
    * master
  ```

- 创建分支
 
  可以使用命令来创建

  ```
  $ git branch dev
  $ git branch
    branch1
    dev
    * master

  ```

  也可以使用IDEA来创建，点击右下角的Git选择```New Branch```填写分支名即可创建新分支。

- 关联本地仓库和远程仓库

  可以分别创建本地仓库和远程仓库，然后再使用下列命令关联，后面就可以将本地仓库中的代码推送到远程仓库了。
  ```
  git remote add origin git@github.com:test/test.git
  ```

- 拉取远程仓库代码
  
  使用```pull```命令可以从远程仓库拉取代码到本地仓库，可以使用```--allow-unrelated-histories```命令，作用是第一次关联本地仓库和远程仓库后将两个边不同的提交记录合并起来，否则直接拉取会失败，可以使用```-u```参数，绑定本地仓库和远程仓库的某个分支，下次再```pull```的时候就不用指定版本了。
  ```
  git pull origin master 
  ```

- 提交本地仓库和远程仓库
  
  使用命令的话直接使用```git commit```和```git push```命令分别提交到本地仓库和推送到远程仓库，比如下面的命令就是提交Test文件到本地仓库，再推送到远程仓库，注意和SVN不同的是Git的```push```是会把所有```commit```的文件全都推送到远程仓库，所以一定要注意```commit```的东西是没有问题的。

  同理，```push```的时候也可以使用```-u```参数。

  ```
  $ git commit Test -m "git commit"
    [master b206f87] git commit
    1 file changed, 2 insertions(+), 1 deletion(-)

  $ git push origin master
    Enumerating objects: 5, done.
    Counting objects: 100% (5/5), done.
    Delta compression using up to 4 threads
    Compressing objects: 100% (2/2), done.
    Writing objects: 100% (3/3), 347 bytes | 347.00  KiB/s, done.
    Total 3 (delta 0), reused 0 (delta 0)
    To github.com:nemolpsky/TestGit.git
    fe6bb9b..b206f87  master -> master

  ``` 
- 合并代码

  合并代码只需要切到需要合并的分支上，然后指定要合并的分支即可，比如要把```master```的代码合并到```dev```分支，切换到```dev```，然后```merge master```即可把所有的改动合并到```dev```分支上。

  ```
  $ git switch dev

  $ git merge master
    Updating fe6bb9b..b206f87
    Fast-forward
    Test | 3 ++-
    1 file changed, 2 insertions(+), 1 deletion(-)
  ```

- 选择性合并
  
  这个功能是最重要的，上面的```merge```命令是直接把```master```分支上所有的变动都合并到当前代码中，有时候我们只是想要合并其中一个，使用```cherry-pick```就可以解决这个问题，选中要合并的```commit id```即可。

  在IDEA中只要选中提交记录右键选择```cherry-pick```即可。

  ```
  $ git log
    commit bf9f29378c90010d0251e4c87d4f855b4d17c1a2 (HEAD -> master, origin/master, origin/HEAD)

  $ git cherry-pick       
    bf9f29378c90010d0251e4c87d4f855b4d17c1a2
    [dev 0b6bbb3] third
    Date: Sun Jun 7 11:17:28 2020 +0800
    1 file changed, 1 insertion(+)
  ```

- 放弃修改和回滚commit
  
  ```git checkout```可以将工作区中未提交的修改都丢弃，对应IDEA中的```Rollback```。
  ```
  $ git checkout -- Test
  ```

  而```git reset```不仅可以放弃所有暂存区的修改，还可以回退版本，在IDEA上选中更早的```commit```记录就有```reset```选项。
  ```
  // 撤销暂存区的修改
  $ git reset HEAD readme.txt

  // 回滚到上次版本，HEAD^是上个版本，HEAD^^是上上个版本，所以写成HEAD~100是上100个版本
  $ git reset --hard HEAD^
  ```