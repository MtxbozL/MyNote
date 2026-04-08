### 1.2.1 Python3 安装说明

Python 2 已于 2020 年 1 月 1 日停止官方维护，工业界主流使用 Python 3.8 及以上稳定版本，官方分发版可通过 Python 官网 获取，对应平台安装核心注意事项：

- Windows 平台：安装时必须勾选`Add Python to PATH`选项，实现环境变量自动配置；
- macOS/Linux 平台：系统通常预装 Python 3，可通过`python3 --version`验证版本，如需升级建议通过官方安装包或 Homebrew/apt 等包管理器安装，避免覆盖系统自带 Python 环境。

### 1.2.2 环境变量配置原理

环境变量是操作系统用于指定系统运行环境的核心参数，其中`PATH`变量用于存储可执行程序的搜索路径。Python 环境变量配置的核心目标，是将 Python 解释器的安装主目录、以及 Scripts（Windows）/bin（类 Unix）子目录添加到系统`PATH`中，使操作系统可在任意路径下通过`python`/`python3`命令调用解释器。

- 手动配置路径：Windows 通过「系统属性→高级→环境变量」编辑系统`PATH`；macOS/Linux 通过修改`~/.bashrc`/`~/.zshrc`文件，添加`export PATH="$PATH:/path/to/python/bin"`，执行`source`命令生效。

### 1.2.3 主流开发环境介绍

|开发环境|核心定位|适用场景|
|---|---|---|
|IDLE|Python 官方自带轻量集成开发环境，内置交互式解释器与脚本编辑器，支持基础语法高亮与调试|入门阶段基础语法练习，零额外配置|
|PyCharm|JetBrains 出品的 Python 专业 IDE，分为免费社区版与付费专业版，内置智能补全、语法检查、调试器、虚拟环境管理、版本控制全链路功能|企业级项目开发，工业界主流工具|
|VSCode|微软出品的轻量级开源代码编辑器，通过安装 Microsoft 官方 Python 扩展可实现完整 Python 开发能力，高度可定制，多语言兼容|多语言混合开发，轻量开发场景|

---
