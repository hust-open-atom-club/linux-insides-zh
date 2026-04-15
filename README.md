# Linux 内核揭秘

本仓库包含一本关于 Linux 内核及其内在机理的在编书籍，是 [linux-insides](https://github.com/0xAX/linux-insides) 的中文翻译版本。

**目的很简单** - 分享 Linux 内核内部原理及相关底层主题的知识。如果你想了解 Linux 内核的底层运作，请查阅[目录](https://github.com/hust-open-atom-club/linux-insides-zh/blob/master/SUMMARY.md)。

**问题/建议**: 如有相关问题或建议，请提交 issue。一方面，对于英文原文问题，请在上游仓库 - [linux-insides](https://github.com/0xAX/linux-insides) 中提交 issue；另一方面，对于中文翻译问题，请在下游仓库 - [linux-insides-zh](https://github.com/hust-open-atom-club/linux-insides-zh) 中提交 issue。

## 前置要求

- 熟悉[汇编语言](https://zh.wikipedia.org/wiki/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80)
- 熟练掌握 [C 语言](https://zh.wikipedia.org/wiki/C%E8%AF%AD%E8%A8%80)
- 此外，[Intel 软件开发者手册](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)中有大量关于 x86_64 处理器的参考资料

## 贡献

如有相关问题或建议，请不吝指教，提交 issues 或者 PRs。对于 `linux-insides-zh` 翻译项目，请通过以下方法进行贡献：

- 将英文原文翻译为中文（目前只提供简体中文的译文）；
- 同步尚未翻译的英文原文至本项目；
- 根据上游英文的更新，检查并更新已有的中文译文；
- 校对已翻译的中文译文，包括修正错别字、润色等；

目前本项目的**翻译进度**与**翻译认领规则**，请查看 [TRANSLATION_STATUS.md](TRANSLATION_STATUS.md)。

在开始翻译之前，请阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 与 [TRANSLATION_NOTES.md](TRANSLATION_NOTES.md)。关于翻译约定的任何问题或建议，同样请提交 issue 讨论。

## 使用 mdBook 构建

仓库已经支持直接使用 `mdBook` 构建网页书籍，同时支持生成 PDF 和 EPUB。

安装：

```bash
cargo install mdbook mdbook-pdf mdbook-epub --locked
```

构建：

```bash
mdbook build
```

本地预览：

```bash
mdbook serve --open
```

生成结果位于 `book/` 目录：
- HTML：`book/html/`
- PDF：`book/pdf/output.pdf`
- EPUB：`book/epub/Linux 内核揭秘.epub`

仓库同时提供了 GitHub Actions 工作流，在 `master` 分支更新后自动构建并部署 GitHub Pages。

## 邮件列表

我们开源俱乐部有一个 [Google Group 邮件列表](https://groups.google.com/g/hust-os-kernel-patches)用于学习和贡献 Linux 内核源码。

**加入方式：** 发送任意邮件到 `hust-os-kernel-patches+subscribe@googlegroups.com`，收到确认邮件后即可加入。

> [!TIP]
> 如果你有谷歌账号，可以直接通过[归档页面](https://groups.google.com/g/hust-os-kernel-patches)申请加入。

## 维护者与作者

- 原文作者：[@0xAX](https://x.com/0xAX)
- 中文维护者：[@mudongliang](https://github.com/mudongliang)、[@xinqiu](https://github.com/xinqiu)
- 中文贡献者：详见 [CONTRIBUTORS.md](CONTRIBUTORS.md)

## 许可证

本项目使用 [BY-NC-SA Creative Commons](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可证。
