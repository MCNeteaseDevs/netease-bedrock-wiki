---
front:
hard: 入门
time: 15分钟
---

# Allay部署教程

本系列教程将指导您使用Allay搭建一个服务器，安装NetAllay以使用网易独有接口，以及使用WaterdogPE搭建群组服。

本系列教程主要侧重于网易版相关内容。有关Allay的更多文档，请移步至[AllayMC官方文档站](https://docs.allaymc.org)。

相关链接：
- AllayMC交流群（QQ）：1072132791
- AllayMC官方文档站：https://docs.allaymc.org
- AllayMC插件市场：https://hub.allaymc.org
- AllayMC代码仓库：https://github.com/AllayMC/Allay

## 安装Java

Allay 需要**Java 21**才能运行。市面上有多种 Java 发行版，但我们推荐以下版本：

- [**GraalVM**](https://www.graalvm.org/) – 提供更好的性能
- [**OpenJDK**](https://adoptium.net/) – 提供更佳的稳定性

安装后，通过运行以下程序验证 Java 是否正确安装：

```shell
java --version
```

如果 Java 安装正确，您应该能看到没有错误提示的版本输出。

## 使用AllayLauncher

[AllayLauncher](https://github.com/AllayMC/AllayLauncher)是一个轻量级、快速的工具（用C++编写），能帮您轻松下载、更新和管理您的Allay实例。

安装它只需执行一个命令：

Windows (PowerShell):
```powershell
Invoke-Expression (Invoke-WebRequest -Uri "https://raw.githubusercontent.com/AllayMC/AllayLauncher/refs/heads/main/scripts/install_windows.ps1").Content
```

Linux:
```bash
wget -qO- https://raw.githubusercontent.com/AllayMC/AllayLauncher/refs/heads/main/scripts/install_linux.sh | bash
```

## 手动安装Allay

### 获取Allay核心文件

请从[GitHub Releases](https://github.com/AllayMC/Allay/releases/latest)页面获取最新稳定版本。您也可以尝试[Nightly Build](https://github.com/AllayMC/Allay/releases/tag/nightly)。

您应该会得到一个文件，名称如下：
```
allay-server-<version>-<commit-hash>[-dev]-shaded.jar
```

示例:
```
allay-server-0.1.0-dev-shaded.jar
```

> `-dev`后缀表示开发版本。

### 运行服务器

如果您的系统有图形界面（GUI），只需双击jar文件即可。如果正确安装了Java，会出现类似这样的窗口：

![installation-p1.png](images/installation-p1.png)

如果您在无头服务器（无图形界面）上，使用以下命令启动服务器：
```bash
java -jar allay-server-*-shaded.jar
```

您会在终端看到同样的启动输出。