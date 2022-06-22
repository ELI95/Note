- git fetch
    - git fetch is the command that tells your local git to retrieve the latest meta-data info from the original
    - git fetch doesn't do any file transferring. It's more like just checking to see if there are any changes available

- [merge & rebase](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
    
- [git pull & git pull --rebase](https://stackoverflow.com/questions/3357122/what-is-the-difference-between-git-pull-and-git-fetch-git-rebase)

- 如何rebase主干更新到分支
  - master分支执行：git pull，同步远端代码库master分支到更新到本地
  - 切换到开发分支：git checkout dev-branch
  - 在开发分支执行：git rebase master，将主干更新以rebase方式合入开发分支

- 如何拉取远程分支到本地
  - git fetch origin 远程分支名
  - git checkout 远程分支名
