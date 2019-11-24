## Git

### Git的用处

-  代码管理
-  版本控制
-  工作流

Git是一款强大的软件，活学活用git可以极大的提高工作效率。

是否可以喊出那个口号，人生苦短，我爱git。

---

#### 1.gitignore



在工程中很容易出现.gitignore并没有忽略掉我们已经添加的文件，那是因为.gitignore对已经追踪(track)的文件是无效的，需要清除缓存，清除缓存后文件将以未追踪的形式出现，这时重新添加(add)并提交(commit)就可以了。所以要尽量避免这种情况！
```shell script
git rm -r --cached .
git add .
git commit -m "comment"

```

git 如何查看提交的名字
```shell script
git config user.name
git config user.email
```
git 如何更改提交的名字
```shell script
git config --global user.name 'nitrogen'
git config --global user.email yourEmail
```

#### git 连接超时
用ssh连接git有时会报错
```shell script
ssh: connect to host github.com port 22: Connection timed out

fatal: Could not read from remote repository.

Please make sure you have the correct access rights

and the repository exists.
```
测试 ssh 连接 失败
```shell script
ssh -T get@github.com
```
测试https连接
```shell script
$ ssh -T -p 443 git@ssh.github.com
```

```shell script
$ git config --local -e 
将
url = git@github.com:你的用户名/仓库名.git
改为
url = https://github.com/你的用户名/仓库名.git
```

