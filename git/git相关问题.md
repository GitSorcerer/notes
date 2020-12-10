# git相关问题

```shell
Another git process seems to be running in this repository, e.g.
an editor opened by 'git commit'. Please make sure all processes
are terminated then try again. If it still fails, a git process
may have crashed in this repository earlier:
remove the file manually to continue.
```

**原因:git在操作得时候中断了,导致锁住得资源没有释放**

**解决:删除.git文件夹中的index.lock文件**

