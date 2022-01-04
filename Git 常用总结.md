### Git 常用总结

#### 1. [git stash](https://git-scm.com/docs/git-stash/en)

当您想记录工作目录和索引的当前状态，但又想回到干净的工作目录时，请使用 `git stash`。 该命令保存您的本地修改并恢复工作目录以匹配 `HEAD` 提交。

```bash
git stash list [<log-options>]
git stash show [-u|--include-untracked|--only-untracked] [<diff-options>] [<stash>]
git stash drop [-q|--quiet] [<stash>]
git stash ( pop | apply ) [--index] [-q|--quiet] [<stash>]
git stash branch <branchname> [<stash>]
git stash [push [-p|--patch] [-k|--[no-]keep-index] [-q|--quiet]
	     [-u|--include-untracked] [-a|--all] [-m|--message <message>]
	     [--pathspec-from-file=<file> [--pathspec-file-nul]]
	     [--] [<pathspec>…​]]
git stash clear
git stash create [<message>]
git stash store [-m|--message <message>] [-q|--quiet] <commit>
```



#### 2. git XXX

