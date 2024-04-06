贡献
================================================================================

如果你想要给 [linux-insides-zh](https://github.com/hust-open-atom-club/linux-insides-zh) 做贡献，请遵照如下规则：

1. 点击 `fork` 按钮：

    ![fork](./Assets/fork_button.png)

2. 通过如下命令 `clone` 你 Github 帐号中的 `linux-insides-zh` 仓库：

    ```
    git clone git@github.com:your_github_username/linux-insides-zh.git
    ```

3. 通过如下命令创建分支 (`branch`)：

    ```
    git checkout -b "linux-insides-zh-fix"
    ```
   其中，`linux-insides-zh-fix` 仅是一个样例。

4. 对本地仓库进行修改。

5. 提交自己的修改，然后推送 (`push`) 到远端。

    ```
    git add your_changed_files
	git commit -m "your comment"
    git push --set-upstream origin linux-insides-zh-fix
    ```

6. 点击 `New Pull Request` 按钮，将你 Github 帐号中的 `linux-insides-zh` 仓库的 linux-insides-zh-fix 分支的修改提交到当前 `linux-insides-zh` 库中。

十分感谢！
