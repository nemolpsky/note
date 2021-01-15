### Git回溯历史版本

很多时候在使用Git开发时，可能需要在master分支上拉取一个分支，但是master分支上又包含了未发布的特性，这个时候就需要用到回溯功能，也就是git reset，回溯到master分支某一个提交点上，从那里拉取新的分支，最后提交合并，保证不会因为别的特性分支造成的未知的影响。


---

#### 1. 创建dev1特性分支并推送

下面这一步是先创建了dev1的分支，添加了一个resetFile.txt的文件，文件中有一行```dev1 text```文本内容，然后把dev1分支推送到远程仓库。

```
E:\GitHub\TestGit>git branch
* master

E:\GitHub\TestGit>git branch dev1

E:\GitHub\TestGit>git branch
  dev1
* master

E:\GitHub\TestGit>git checkout dev1
Switched to branch 'dev1'

E:\GitHub\TestGit>git status
On branch dev1
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        resetFile.txt

E:\GitHub\TestGit>git add .

E:\GitHub\TestGit>git commit -m "add dev1 text."
[dev1 d74602a] add dev1 text.
 1 file changed, 1 insertion(+)
 create mode 100644 resetFile.txt


E:\GitHub\TestGit>git push --set-upstream origin dev1
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 4 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 295 bytes | 295.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
remote:
remote: Create a pull request for 'dev1' on GitHub by visiting:
remote:      https://github.com/nemolpsky/TestGit/pull/new/dev1
remote:
To github.com:nemolpsky/TestGit.git
 * [new branch]      dev1 -> dev1
Branch 'dev1' set up to track remote branch 'dev1' from 'origin'.
```

#### 2. 合并主分支

现在切换到主分支，执行```merge```操作，再推送到远程仓库，可以看到日志中已经包含了dev1分支上的东西。

```
E:\GitHub\TestGit>git switch master
Switched to branch 'master'
Your branch is up to date with 'origin/master'.

E:\GitHub\TestGit>git merge dev1
Updating bf9f293..d74602a
Fast-forward
 resetFile.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 resetFile.txt

E:\GitHub\TestGit>git push
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To github.com:nemolpsky/TestGit.git
   bf9f293..d74602a  master -> master

E:\GitHub\TestGit>git log
commit d74602a9fa8bb341b74a5874ce3f08bb08c51248 (HEAD -> master, origin/master, origin/dev1, origin/HEAD, dev1)
Author: nemolpsky <git@github.com:nemolpsky/Note.git>
Date:   Wed Dec 30 10:25:22 2020 +0800

    add dev1 text.   
```


#### 3. 回溯并创建dev2分支

使用git log可以看到最近的提交记录，其中哈希值为```bf9f29378c90010d0251e4c87d4f855b4d17c1a2```的提交记录是dev1分支合并之前的最新提交记录，所以可以使用git reset命令回溯到那个点，然后创建dev2分支，这个时候创建出来的分支就不再包括dev1分支的东西了。

```
E:\GitHub\TestGit>git reset --hard bf9f29378c90010d0251e4c87d4f855b4d17c1a2
HEAD is now at bf9f293 third

E:\GitHub\TestGit>git log
commit bf9f29378c90010d0251e4c87d4f855b4d17c1a2 (HEAD -> master)
Author: nemolpsky <git@github.com:nemolpsky/Note.git>
Date:   Sun Jun 7 11:17:28 2020 +0800

    third

E:\GitHub\TestGit>git branch dev2

E:\GitHub\TestGit>git branch
  dev1
  dev2
* master

E:\GitHub\TestGit>git switch dev2
Switched to branch 'dev2'

E:\GitHub\TestGit>dir
 驱动器 E 中的卷是 开发辅助
 卷的序列号是 6E0F-553A

 E:\GitHub\TestGit 的目录

2020/12/30  10:30    <DIR>          .
2020/12/30  10:30    <DIR>          ..
2020/12/30  10:18                 9 README.md
2020/12/30  10:18                48 Test
2020/12/30  10:18                18 Test2
2020/12/30  10:18                 3 Test3
               4 个文件             78 字节
               2 个目录 198,013,521,920 可用字节

```


#### 4. 合并回主分支

在dev2分支上也创建一个resetFile.txt文件，也添加一行文本然后提交。这个时候最重要的部分来了，就是合并回主分支。

```
E:\GitHub\TestGit>git status
On branch dev2
Untracked files:
  (use "git add <file>..." to include in what will be committed)
        resetFile.txt

nothing added to commit but untracked files present (use "git add" to track)

E:\GitHub\TestGit>git add .

E:\GitHub\TestGit>git commit -m "add dev2 text."
[dev2 d4f7248] add dev2 text.
 1 file changed, 1 insertion(+)
 create mode 100644 resetFile.txt
```

这个时候因为使用了回溯，git log是看不到master上最新的日志，那就要使用git reflog，这个命令可以查看到所有的操作，可以看到```d74602a (origin/master, origin/dev1, origin/HEAD, dev1) HEAD@{3}: merge dev1: Fast-forward```这行日志，表示执行了dev1分支合并到master分支的操作，也就是master目前最新的时间点，这个时候切回master分支，再使用git reset命令回到这个最新的时间点就可以。

```
E:\GitHub\TestGit>git reflog
d4f7248 (HEAD -> dev2) HEAD@{0}: commit: add dev2 text.
bf9f293 (master) HEAD@{1}: checkout: moving from master to dev2
bf9f293 (master) HEAD@{2}: reset: moving to bf9f29378c90010d0251e4c87d4f855b4d17c1a2
d74602a (origin/master, origin/dev1, origin/HEAD, dev1) HEAD@{3}: merge dev1: Fast-forward
bf9f293 (master) HEAD@{4}: checkout: moving from dev1 to master
d74602a (origin/master, origin/dev1, origin/HEAD, dev1) HEAD@{5}: commit: add dev1 text.
bf9f293 (master) HEAD@{6}: checkout: moving from master to dev1
bf9f293 (master) HEAD@{7}: clone: from github.com:nemolpsky/TestGit.git

E:\GitHub\TestGit>git switch master
Switched to branch 'master'
Your branch is behind 'origin/master' by 1 commit, and can be fast-forwarded.
  (use "git pull" to update your local branch)

E:\GitHub\TestGit>git reset --hard d74602a
HEAD is now at d74602a add dev1 text.  
```

这个时候会提示文件冲突，因为故意在dev1和dev2分支创建了相同的文件，肯定会冲突。

```
<<<<<<< HEAD
dev1 text。
=======
dev2 text.
>>>>>>> dev2
```

解决下冲突

```
dev1 text。
dev2 text.
```

再重新提交并推送，至此就会发现master分支上包含了dev1和dev2两个分支上的特性。这也就解决了当master分支上包含了一些特定分支的特性时该如何回溯拉取以前某个时间点的master分支并以此为基础创建分支。

```
E:\GitHub\TestGit>git add .

E:\GitHub\TestGit>git status
On branch master
Your branch is up to date with 'origin/master'.

All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes to be committed:
        modified:   resetFile.txt


E:\GitHub\TestGit>git commit -m "add dev1 and dev2."
[master 036a1a7] add dev1 and dev2.

E:\GitHub\TestGit>git push
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 4 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 554 bytes | 554.00 KiB/s, done.
Total 6 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 1 local object.
To github.com:nemolpsky/TestGit.git
   d74602a..036a1a7  master -> master
```