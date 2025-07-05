# pacman

pacman 是 Arch 发行版的包管理器。

## 基本用法

更新系统，并安装某个软件的最新版本。

```bash
$ sudo pacman -Syu [packagename]
```

- `-S`：安装一个软件包。
- `-y`：同步存储库数据库。
- `-u`：将系统更新到最新。
 
安装软件包而不更新系统的命令是

```bash
$ sudo pacman -S [packagename]
```

删除软件包。

```bash
$ sudo pacman -R [packagename]
```

## AUR

AUR 是  Arch 用户仓库 (AUR)，在那里可以找到一些默认渠道无法获取的软件，还能更轻松地从 GitHub 和其他来源提取内容。