### 1.3.1 环境隔离的核心必要性

Python 全局环境中，所有第三方包均为全局共享，不同项目对同一包的版本需求存在天然差异（如项目 A 需`requests==2.25.1`，项目 B 需`requests==2.32.3`），全局环境无法实现多版本共存，会导致**依赖地狱**问题；同时可避免全局环境被大量包污染，保证项目的可移植性、可复现性与环境一致性。

虚拟环境的核心本质：为单个项目创建独立的 Python 运行环境，实现解释器、第三方包、依赖版本的完全隔离，不同项目的环境互不影响。

### 1.3.2 pip 包管理工具

pip 是 Python 官方默认的包管理工具，Python 3.4 及以上版本默认内置，用于第三方包的查找、安装、升级、卸载与依赖管理，核心命令如下：

表格

|命令|功能描述|
|---|---|
|`pip install package_name`|安装最新版本的指定包|
|`pip install package_name==x.y.z`|安装指定版本的包|
|`pip install --upgrade package_name`|升级指定包到最新版本|
|`pip uninstall package_name`|卸载指定包|
|`pip list`|查看当前环境已安装的所有包|
|`pip freeze > requirements.txt`|导出当前环境的所有依赖包与版本到 requirements.txt 文件|
|`pip install -r requirements.txt`|从 requirements.txt 文件批量安装所有依赖|

补充：国内可通过清华、阿里等镜像源提升下载速度，临时使用语法：`pip install package_name -i https://pypi.tuna.tsinghua.edu.cn/simple`；永久配置需修改对应平台的 pip 配置文件。

### 1.3.3 venv 原生虚拟环境模块

venv 是 Python 3.3 及以上版本内置的虚拟环境管理模块，无需额外安装，轻量原生，是基础项目环境隔离的官方推荐方案，核心操作如下：

1. **创建虚拟环境**：执行`python -m venv 环境名称`（常用名称为`venv`），执行后会在当前目录生成对应文件夹，包含独立的 Python 解释器、pip 工具、第三方包安装目录`site-packages`；
2. **激活虚拟环境**：
    
    - Windows（cmd）：`venv\Scripts\activate.bat`
    - Windows（PowerShell）：`venv\Scripts\Activate.ps1`
    - macOS/Linux：`source venv/bin/activate`
        
        激活后，终端提示符前会显示环境名称，此时所有 pip 操作仅作用于当前虚拟环境；
    
3. **退出虚拟环境**：执行`deactivate`命令。

### 1.3.4 Anaconda/Miniconda 发行版与环境管理

- Anaconda：面向数据科学领域的 Python 发行版，内置 conda 包管理器与数百个预安装的科学计算包（NumPy、Pandas 等）；
- Miniconda：Anaconda 的精简版，仅包含 Python 解释器与 conda 包管理器，体积更小，可按需安装包，是工业界主流选择。

conda 不仅可管理 Python 包，还可管理非 Python 依赖（如 C/C++ 库），同时支持多 Python 版本的环境隔离，是数据科学、AI 开发场景的首选工具，核心命令如下：

表格

|命令|功能描述|
|---|---|
|`conda create -n 环境名称 python=x.y`|创建指定 Python 版本的虚拟环境|
|`conda activate 环境名称`|激活指定虚拟环境|
|`conda deactivate`|退出当前虚拟环境|
|`conda install package_name`|在当前环境安装指定包|
|`conda env export > environment.yml`|导出当前环境配置到 yml 文件|
|`conda env create -f environment.yml`|从 yml 配置文件还原环境|

---
