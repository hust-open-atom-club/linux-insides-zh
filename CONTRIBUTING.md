贡献
================================================================================

如果你想要给 [linux-insides-zh](https://github.com/MintCN/linux-insides-zh) 做贡献，请遵照如下的规则：

1. 点击 `fork` 按钮：

    ![fork](./Assets/fork_button.png)

2. 通过如下命令从你的 Github 帐号 `clone` 库： 

    ```
    git clone git@github.com:your_github_username/linux-insides-zh.git
    ```

3. 通过如下命令创建分支 (`branch`) :

    ```
    git checkout -b "linux-insides-zh-fix"
    # linux-insides-zh-fix is just an example
    ```

4. 对本地库进行修改。

5. 提交自己的修改，然后推送 (`push`) 到远端。

    ```
    git add your_changed_files
	git commit -m "your comment"
    git push --set-upstream origin linux-insides-zh-fix
    ```

6. 点击 `New pull request` 按钮，将 your_github_username 的 `linux-insides-zh` 库的 linux-insides-zh-fix 分支的修改提交到 MintCN 的 `linux-insides-zh` 库中。

十分感谢！
