贡献
================================================================================

如果你想要给 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 做贡献，请遵照如下的规则：

1. 按下 `fork` 按钮：

    ![fork](http://oi58.tinypic.com/jj2trm.jpg)

2. 通过如下命令从你的 Github 帐号 `clone` 库： 

```
    git clone git@github.com:your_github_username/linux-insides.git
```

3. 通过如下命令创建分支 (`branch`) :

```
    git checkout -b "linux-insides-zh-fix"
    # linux-insides-zh-fix is just an example
```

4. 做出来你自己的修改。

5. 提交自己的修改，然后推送 (`push`) 到远端。

```
    git add your_changed_files
	git commit -m "your comment"
    git push --set-upstream origin linux-insides-zh-fix
```

6. 按下 `New pull request` 按钮，将 your_github_username 库的 linux-insides-zh-fix 分支的修改提交到 MintCN 的 `linux-insides-zh` 库中。

Thank you.
