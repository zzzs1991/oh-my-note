## Git

### Git的用处

- 代码管理
- 版本控制
- 工作流

Git是一款强大的软件，活学活用git可以极大的提高工作效率。

是否可以喊出那个口号，人生苦短，我爱git。

---

1.gitignore



```
在工程中很容易出现.gitignore并没有忽略掉我们已经添加的文件，那是因为.gitignore对已经追踪(track)的文件是无效的，需要清除缓存，清除缓存后文件将以未追踪的形式出现，这时重新添加(add)并提交(commit)就可以了。所以要尽量避免这种情况！

git rm -r --cached .
git add .
git commit -m "comment"

```

