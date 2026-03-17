## 一、核心定位

`useradd` 是 Linux 系统中**最核心、最基础的新用户创建命令**，属于「用户管理类」，仅 root 用户可执行。核心作用是**创建新的系统用户，支持指定用户 ID（UID）、主组 / 附加组、家目录路径、登录 shell、用户注释、密码时效策略、创建系统用户、修改默认创建规则**，是用户管理、权限配置、账户组织的必备基础命令，也是 Linux 入门必学的用户管理命令。

> 补充说明：`useradd` 名称来源于 “user add”（添加用户）；与`userdel`（删除用户）、`usermod`（修改用户）并称为 Linux 用户管理三剑客；创建的用户信息会写入`/etc/passwd`、`/etc/shadow`、`/etc/group`、`/etc/gshadow`文件。

## 二、语法格式

bash

运行

```
# 核心语法：创建新用户
useradd [可选参数] 用户名
```

> 关键语法说明：
> 
> 1. **用户名**：必选，需符合命名规范（字母、数字、下划线、连字符，首字符为字母或下划线，长度不超过 32 字符）；
> 2. **可选参数**：如`-u`（指定 UID）、`-g`（指定主组）、`-G`（指定附加组）、`-s`（指定 shell）、`-d`（指定家目录）、`-r`（系统用户）等；
> 3. **默认行为**：自动分配下一个可用的非系统 UID（通常从 1000 开始）、创建同名主组、创建家目录（路径通常为`/home/用户名`）、设置默认登录 shell（通常为`/bin/bash`）、从骨架目录（`/etc/skel`）复制文件到家目录。

## 三、高频核心参数（按功能分类）

### 1. 基础创建类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|无参数（直接执行`useradd 用户名`）|创建普通用户，自动分配 UID/GID、创建家目录、设置默认 shell|日常快速创建普通用户，最基础、最常用的用法|
|`-m, --create-home`|强制**创建家目录**（默认通常已开启，部分系统需显式）|确保创建家目录、自定义创建规则|
|`-M, --no-create-home`|**不创建家目录**|临时用户、系统用户、无需家目录的场景|
|`-c, --comment COMMENT`|添加用户**注释**（通常为用户全名或描述）|用户信息标注、账户管理|
|`-h, --help`|输出 useradd 帮助信息|快速查看参数与用法|
|`-v, --version`|输出 useradd 版本信息|确认版本兼容性|

### 2. UID/GID 设置类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|`-u, --uid UID`|指定**用户 ID（UID）**，UID 为正整数，需唯一（除非用`-o`）|自定义 UID、固定 UID 的用户创建|
|`-o, --non-unique`|允许创建**非唯一 UID**的用户（UID 与已有用户重复）|特殊场景、共享 UID|
|`-g, --gid GROUP`|指定**主组**（GROUP 为组名或 GID），需为已存在的组|自定义主组、用户组管理|
|`-G, --groups GROUPS`|指定**附加组**（GROUPS 为逗号分隔的组名 / GID 列表），需为已存在的组|用户附加组管理、多组权限配置|
|`-N, --no-user-group`|**不创建同名主组**，使用默认组（通常为`users`）|避免创建同名组、使用公共组|

### 3. 家目录与 shell 类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|`-d, --home-dir HOME_DIR`|指定**家目录路径**（HOME_DIR 为绝对路径）|自定义家目录位置、非标准家目录|
|`-s, --shell SHELL`|指定**登录 shell**（SHELL 为 shell 的绝对路径）|自定义登录 shell、系统用户 shell 设置|
|`-k, --skel SKEL_DIR`|指定**骨架目录**（SKEL_DIR 为绝对路径），从该目录复制文件到家目录|自定义家目录初始文件、个性化配置|

### 4. 系统用户类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|`-r, --system`|创建**系统用户**，UID 从系统 UID 范围分配（通常 < 1000），不创建家目录（除非显式`-m`），默认 shell 通常为`/sbin/nologin`|系统服务用户创建、系统账户管理|

### 5. 密码时效类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|`-e, --expiredate EXPIRE_DATE`|设置账户**过期日期**（EXPIRE_DATE 格式为`YYYY-MM-DD`）|临时账户、账户有效期管理|
|`-f, --inactive INACTIVE`|设置密码**过期非活动天数**（INACTIVE 为天数，密码过期后 INACTIVE 天仍可登录，之后账户锁定）|密码过期缓冲期|

### 6. 默认值修改类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|`-D, --defaults`|查看或修改`useradd`的**默认创建规则**（如`/etc/default/useradd`中的值），仅修改默认值不创建用户|自定义默认创建规则、批量创建前统一配置|

## 四、实操示例

### 1. 基础入门示例（新手必练，零门槛上手，需 root 权限或测试环境）

bash

运行

```
# 示例1：最基础用法，创建普通用户
useradd devuser
# 执行效果：创建devuser用户，自动分配UID/GID、创建家目录/home/devuser、设置默认shell/bin/bash
```

bash

运行

```
# 示例2：指定UID创建用户
useradd -u 1005 testuser
# 执行效果：创建testuser用户，UID为1005
```

bash

运行

```
# 示例3：指定主组创建用户
useradd -g devteam devuser2
# 执行效果：创建devuser2用户，主组为devteam（需devteam已存在）
```

bash

运行

```
# 示例4：指定附加组创建用户
useradd -G devteam,adm devuser3
# 执行效果：创建devuser3用户，附加组为devteam和adm（需组已存在）
```

bash

运行

```
# 示例5：指定登录shell创建用户
useradd -s /bin/zsh devuser4
# 执行效果：创建devuser4用户，登录shell为/bin/zsh
```

bash

运行

```
# 示例6：指定家目录路径创建用户
useradd -d /data/devuser5 devuser5
# 执行效果：创建devuser5用户，家目录为/data/devuser5
```

bash

运行

```
# 示例7：创建系统用户
useradd -r sysuser
# 执行效果：创建sysuser系统用户，UID从系统范围分配（<1000），不创建家目录，默认shell为/sbin/nologin
```

bash

运行

```
# 示例8：添加用户注释
useradd -c "Development User" devuser6
# 执行效果：创建devuser6用户，注释为"Development User"
```

bash

运行

```
# 示例9：设置账户过期日期
useradd -e 2026-12-31 tempuser
# 执行效果：创建tempuser用户，账户在2026-12-31过期
```

---

### 2. 高频日常场景示例

#### 场景 1：查看创建的用户信息

bash

运行

```
# 查看/etc/passwd文件，确认用户已创建
cat /etc/passwd | grep devuser
# 执行效果示例：
# devuser:x:1004:1004::/home/devuser:/bin/bash
```

bash

运行

```
# 用id命令查看用户的UID、GID、所属组
id devuser
# 执行效果示例：
# uid=1004(devuser) gid=1004(devuser) groups=1004(devuser)
```

#### 场景 2：结合 passwd 设置用户密码

bash

运行

```
# 创建用户后设置密码
useradd devuser
passwd devuser
# 执行效果：设置devuser的密码，用户可登录
```

#### 场景 3：结合 userdel 删除用户

bash

运行

```
# 创建用户后删除
useradd testuser
userdel testuser
# 执行效果：testuser用户被删除
```

#### 场景 4：结合 usermod 修改用户

bash

运行

```
# 创建用户后修改附加组
useradd devuser
usermod -aG adm devuser
# 执行效果：devuser的附加组添加adm
```

---

### 3. 进阶奇妙用法

#### 奇妙用法 1：批量创建用户（结合循环）

bash

运行

```
# 批量创建dev1到dev5用户
for i in {1..5}; do
    useradd dev$i
done
```

#### 奇妙用法 2：修改 useradd 默认值

bash

运行

```
# 查看当前默认值
useradd -D
# 执行效果示例：
# GROUP=100
# HOME=/home
# INACTIVE=-1
# EXPIRE=
# SHELL=/bin/bash
# SKEL=/etc/skel
# CREATE_MAIL_SPOOL=yes
```

bash

运行

```
# 修改默认shell为/bin/zsh
useradd -D -s /bin/zsh
# 执行效果：后续创建的用户默认shell为/bin/zsh
```

#### 奇妙用法 3：不创建同名主组，使用公共组

bash

运行

```
# 不创建同名主组，使用users组
useradd -N devuser7
# 执行效果：devuser7的主组为users，不创建devuser7组
```

## 五、避坑提示 & 注意事项

1. **新手最高频踩坑：需要 root 权限**
    
    - 普通用户**无法创建用户**，仅 root 用户可执行；
    - 强制要求：创建用户必须加`sudo`，或在 root 用户下操作；
    - 典型错误：普通用户执行`useradd devuser`（报错`useradd: Permission denied`）；
    - 正确示例：`sudo useradd devuser`。
    
2. **用户名的命名规范**
    
    - 用户名需符合规范：首字符为字母或下划线，后续字符为字母、数字、下划线、连字符，长度不超过 32 字符；
    - 解决方法：命名时遵循规范，避免特殊字符和过长名称；
    - 典型错误：`useradd 123user`（首字符为数字，可能报错）；
    - 正确示例：`useradd user123`。
    
3. **UID/GID 的冲突问题**
    
    - 指定 UID/GID 时，UID/GID 需唯一（除非用`-o`），否则会报错；
    - 解决方法：指定 UID/GID 前先确认未被使用（`cat /etc/passwd | grep UID`、`cat /etc/group | grep GID`）；
    - 典型错误：`useradd -u 1004 testuser`（若 1004 已被使用则报错`useradd: UID '1004' already exists`）；
    - 正确示例：`useradd -u 1006 testuser`（确认 1006 未被使用）。
    
4. **家目录的创建与骨架目录**
    
    - 默认会从骨架目录（`/etc/skel`）复制文件（如`.bashrc`、`.bash_profile`）到家目录；
    - 解决方法：需自定义家目录初始文件时，修改`/etc/skel`目录，或用`-k`指定自定义骨架目录；
    - 典型错误：创建用户后发现家目录没有`.bashrc`（可能`/etc/skel`被修改）；
    - 正确示例：确保`/etc/skel`目录有需要的初始文件。
    
5. **系统用户的 UID 范围与 shell**
    
    - 系统用户的 UID 范围通常在`/etc/login.defs`中定义（UID_MIN_SYS、UID_MAX_SYS），通常 < 1000；
    - 系统用户的默认 shell 通常为`/sbin/nologin`，无法登录；
    - 解决方法：创建系统用户用`-r`，自动从系统范围分配 UID；若需系统用户可登录，用`-s`指定 shell；
    - 典型错误：`useradd -r sysuser`后尝试登录（失败，因为 shell 为 /sbin/nologin）；
    - 正确示例：`useradd -r -s /bin/bash sysuser`（仅在测试环境）。
    
6. **不要修改系统已存在的重要用户**
    
    - 不要修改系统已存在的重要用户（如 root、bin、daemon），否则会导致系统服务异常；
    - 强制要求：仅创建新的用户，不要修改系统已存在的用户；
    - 典型错误：`usermod -u 1000 root`（修改 root 的 UID，高风险，绝对禁止尝试）；
    - 正确示例：仅创建新的普通用户，如`useradd devuser`。
    
7. **创建用户后需设置密码**
    
    - 创建用户后默认无密码（或密码锁定），用户无法登录；
    - 解决方法：创建用户后及时用`passwd`设置密码，或用`passwd -e`强制首次登录改密；
    - 典型错误：创建用户后不设置密码，用户无法登录；
    - 正确示例：`useradd devuser && passwd devuser && passwd -e devuser`。
    
8. **`-D`修改默认值的影响**
    
    - `-D`修改的是`useradd`的默认创建规则（`/etc/default/useradd`），对后续创建的所有用户生效；
    - 解决方法：修改默认值前先确认，或修改后及时改回；
    - 典型错误：`useradd -D -s /bin/zsh`后，所有后续用户默认 shell 为 /bin/zsh（若不需要）；
    - 正确示例：仅在批量创建前统一修改默认值，创建后及时改回。
    

## 六、同类命令对比 & 拓展

表格

|命令|核心作用|与 useradd 的区别 & 适用场景|
|:--|:--|:--|
|`userdel`|删除用户|useradd 创建用户，userdel 删除用户；两者常配合使用，创建用 useradd，删除用 userdel|
|`usermod`|修改用户|useradd 创建用户，usermod 修改用户（UID、GID、家目录、shell、附加组等）；两者常配合使用，创建用 useradd，修改用 usermod|
|`passwd`|设置用户密码|useradd 创建用户，passwd 设置密码；两者常配合使用，创建用户后用 passwd 设置密码|
|`id`|查看用户信息|优势：查看用户的 UID、GID、所属组；劣势：无法创建 / 修改用户。查看用户信息用 id，创建 / 修改用 useradd/usermod|
|`cat /etc/passwd`|查看用户信息|优势：查看所有用户信息；劣势：无法创建 / 修改用户。查看用户信息用 cat /etc/passwd，创建 / 修改用 useradd/usermod|
|`chage`|修改用户密码时效|useradd 可设置基础密码时效（`-e`/`-f`）；chage 专门用于修改更详细的密码时效策略。基础时效设置用 useradd，详细时效配置用 chage|

## 七、课后练习任务

**重要提示：所有练习仅在测试环境操作，绝对不要修改系统已存在的重要用户！**

1. 基础创建练习：用`sudo useradd devuser`创建普通用户，用`cat /etc/passwd | grep devuser`和`id devuser`查看创建的用户信息。
2. 指定 UID 练习：用`sudo useradd -u 1005 testuser`创建指定 UID 的用户，用`cat /etc/passwd | grep testuser`验证 UID。
3. 指定主组练习：先创建`devteam`组（`sudo groupadd devteam`），用`sudo useradd -g devteam devuser2`创建指定主组的用户，用`id devuser2`验证主组。
4. 指定附加组练习：先创建`adm`组（`sudo groupadd adm`），用`sudo useradd -G devteam,adm devuser3`创建指定附加组的用户，用`id devuser3`验证附加组。
5. 指定 shell 练习：用`sudo useradd -s /bin/zsh devuser4`创建指定 shell 的用户，用`cat /etc/passwd | grep devuser4`验证 shell。
6. 指定家目录练习：用`sudo useradd -d /data/devuser5 devuser5`创建指定家目录的用户，用`ls -ld /data/devuser5`验证家目录。
7. 创建系统用户练习：用`sudo useradd -r sysuser`创建系统用户，用`cat /etc/passwd | grep sysuser`验证 UID 是否在系统范围、shell 是否为 /sbin/nologin。
8. 添加注释练习：用`sudo useradd -c "Development User" devuser6`创建带注释的用户，用`cat /etc/passwd | grep devuser6`验证注释。
9. 设置过期日期练习：用`sudo useradd -e 2026-12-31 tempuser`创建带过期日期的用户，用`chage -l tempuser`验证过期日期。
10. 查看 /etc/passwd 练习：用`cat /etc/passwd`查看所有用户信息，找到刚创建的用户。
11. 结合 passwd 练习：用`sudo useradd devuser && sudo passwd devuser`创建用户并设置密码，验证用户可登录（仅在测试环境）。
12. 结合 userdel 练习：用`sudo useradd testuser && sudo userdel testuser`创建后删除，用`cat /etc/passwd | grep testuser`验证是否删除。
13. 批量创建练习：用循环批量创建`dev1`到`dev5`用户，用`cat /etc/passwd | grep dev`验证。
14. 修改默认值练习：用`sudo useradd -D`查看当前默认值，用`sudo useradd -D -s /bin/zsh`修改默认 shell，再用`sudo useradd -D`验证，最后改回原默认值。
15. 不创建同名主组练习：用`sudo useradd -N devuser7`创建用户，用`id devuser7`验证主组为 users，用`cat /etc/group | grep devuser7`验证未创建同名组。
16. 权限不足练习：尝试用普通用户创建用户，观察权限不足的报错；再用`sudo`创建，验证效果。
17. 帮助练习：用`useradd --help`查看帮助信息，浏览所有参数。

继续userdel命令

已完成思考

# userdel - delete a user account and related files（删除用户账户及相关文件）

## 一、核心定位

`userdel` 是 Linux 系统中**最核心、最基础的用户账户删除命令**，属于「用户管理类」，仅 root 用户可执行。核心作用是**删除已存在的用户账户，支持删除用户的家目录和邮件、强制删除（需谨慎）**，是用户管理、权限清理、账户组织调整的必备基础命令，也是 Linux 入门必学的用户管理命令。

> 补充说明：`userdel` 名称来源于 “user delete”（删除用户）；与`useradd`（创建用户）、`usermod`（修改用户）并称为 Linux 用户管理三剑客；删除的用户信息会从`/etc/passwd`、`/etc/shadow`、`/etc/group`、`/etc/gshadow`文件中移除。

## 二、语法格式

bash

运行

```
# 核心语法：删除用户
userdel [可选参数] 用户名
```

> 关键语法说明：
> 
> 1. **用户名**：必选，需为已存在的用户；
> 2. **可选参数**：如`-r`（删除家目录）、`-f`（强制删除）等；
> 3. **默认行为**：仅删除用户账户信息，**保留用户的家目录和邮件**，删除前需确认用户未登录且不是任何进程的所有者。

## 三、高频核心参数（按功能分类）

### 1. 基础删除类

表格

|参数|核心作用|高频使用场景|
|:--|:--|:--|
|无参数（直接执行`userdel 用户名`）|删除用户账户信息，**保留家目录和邮件**|临时删除用户、保留用户数据，最基础、最常用的用法|
|`-r, --remove`|删除用户的**家目录和邮件**（家目录通常为`/home/用户名`，邮件通常为`/var/spool/mail/用户名`）|彻底删除用户、清理用户数据，**需谨慎**|
|`-f, --force`|强制删除，即使用户登录中也删除（需谨慎，可能导致进程异常）|特殊场景、强制清理用户|
|`-h, --help`|输出 userdel 帮助信息|快速查看参数与用法|
|`-v, --version`|输出 userdel 版本信息|确认版本兼容性|

## 四、实操示例

### 1. 基础入门示例（新手必练，零门槛上手，需 root 权限或测试环境）

bash

运行

```
# 示例1：最基础用法，删除用户账户，保留家目录
useradd testuser
userdel testuser
# 执行效果：testuser用户账户被删除，家目录/home/testuser保留
```

bash

运行

```
# 示例2：删除用户账户及家目录和邮件（需谨慎）
useradd testuser
userdel -r testuser
# 执行效果：testuser用户账户被删除，家目录/home/testuser和邮件/var/spool/mail/testuser也被删除
```

bash

运行

```
# 示例3：强制删除用户（需谨慎）
useradd testuser
userdel -f testuser
# 执行效果：testuser用户被强制删除，即使用户登录中也删除
```

---

### 2. 高频日常场景示例

#### 场景 1：查看删除前后的用户信息

bash

运行

```
# 查看删除前的用户信息
cat /etc/passwd | grep testuser
# 删除用户
userdel testuser
# 查看删除后的用户信息（无输出则已删除）
cat /etc/passwd | grep testuser
```

#### 场景 2：结合 useradd 创建后删除

bash

运行

```
# 创建用户后删除
useradd devuser
userdel devuser
# 执行效果：devuser用户被创建后删除
```

#### 场景 3：确认家目录是否删除

bash

运行

```
# 用-r删除用户后，确认家目录已删除
useradd testuser
userdel -r testuser
ls -ld /home/testuser
# 执行效果：ls报错，家目录已删除
```

---

### 3. 进阶奇妙用法

#### 奇妙用法 1：批量删除用户（结合循环）

bash

运行

```
# 批量删除dev1到dev5用户（先确认这些用户存在且可删除）
for i in {1..5}; do
    userdel dev$i
done
```

#### 奇妙用法 2：删除用户但保留家目录（默认行为）

bash

运行

```
# 删除用户但保留家目录（默认）
useradd testuser
userdel testuser
ls -ld /home/testuser
# 执行效果：用户被删除，家目录保留
```

## 五、避坑提示 & 注意事项

1. **新手最高频踩坑：需要 root 权限**
    
    - 普通用户**无法删除用户**，仅 root 用户可执行；
    - 强制要求：删除用户必须加`sudo`，或在 root 用户下操作；
    - 典型错误：普通用户执行`userdel testuser`（报错`userdel: Permission denied`）；
    - 正确示例：`sudo userdel testuser`。
    
2. **不能删除用户的主组**
    
    - 若用户的主组是该用户的同名组（`useradd`默认创建），删除用户时同名主组会自动删除；若主组不是同名组，则不能删除主组；
    - 解决方法：删除用户前先确认主组情况，若需删除主组用`groupdel`；
    - 典型错误：`userdel testuser`后尝试删除同名主组（已自动删除）；
    - 正确示例：仅删除用户，同名主组自动删除。
    
3. **不能删除系统已存在的重要用户**
    
    - 不要删除系统已存在的重要用户（如 root、bin、daemon），否则会导致系统服务异常、无法启动；
    - 强制要求：仅删除自己创建的普通用户，不要删除系统已存在的用户；
    - 典型错误：`userdel root`（高风险，绝对禁止尝试）；
    - 正确示例：仅删除自己创建的普通用户，如`userdel testuser`。
    
4. **`-r`删除家目录的高危风险**
    
    - `-r` 会删除用户的家目录和邮件，无任何提示，一旦执行无法恢复；
    - 强制要求：
        
        - 使用`-r`前，**必须先确认家目录无重要数据，或提前备份**；
        - 重要用户家目录先备份，再删除；
        
    - 典型错误：`userdel -r devuser`（家目录有重要代码，未备份，高风险）；
    - 正确示例：先备份家目录，再用`userdel -r devuser`。
    
5. **删除前确认用户未登录**
    
    - 删除用户前需确认用户未登录，且不是任何进程的所有者，否则会报错或导致进程异常；
    - 解决方法：删除前用`who`或`ps aux | grep 用户名`确认用户未登录；
    - 典型错误：`userdel testuser`报错`userdel: user testuser is currently used by process 1234`；
    - 正确示例：先终止用户的进程，或等用户退出后再删除。
    
6. **家目录的备份**
    
    - 删除用户前，若家目录有重要数据，必须提前备份；
    - 解决方法：用`tar`备份家目录，如`tar -czf testuser_home.tar.gz /home/testuser`；
    - 典型错误：未备份就用`userdel -r`删除用户，导致数据丢失；
    - 正确示例：先备份家目录，再删除用户。
    
7. **不要直接编辑`/etc/passwd`**
    
    - 用户信息存储在`/etc/passwd`文件，不要直接编辑该文件；
    - 解决方法：仅用`useradd`、`userdel`、`usermod`等命令修改用户信息；
    - 典型错误：直接编辑`/etc/passwd`删除用户（可能导致文件损坏，其他用户无法登录）；
    - 正确示例：仅用`userdel`删除用户，不要直接编辑`/etc/passwd`。
    
8. **`-f`强制删除的使用**
    
    - `-f` 会强制删除用户，即使用户登录中也删除，无任何提示，一旦执行可能导致进程异常；
    - 强制要求：不要使用`-f`，除非在特殊测试环境且已备份；
    - 典型错误：`userdel -f testuser`（用户登录中，导致进程异常）；
    - 正确示例：仅在特殊测试环境使用`-f`，且提前备份。
    

## 六、同类命令对比 & 拓展

表格

|命令|核心作用|与 userdel 的区别 & 适用场景|
|:--|:--|:--|
|`useradd`|创建用户|userdel 删除用户，useradd 创建用户；两者常配合使用，创建用 useradd，删除用 userdel|
|`usermod`|修改用户|userdel 删除用户，usermod 修改用户（UID、GID、家目录、shell、附加组等）；两者常配合使用，创建用 useradd，修改用 usermod，删除用 userdel|
|`groupdel`|删除用户组|userdel 删除用户，groupdel 删除用户组；两者常配合使用，删除用户后同名主组自动删除，其他组用 groupdel|
|`id`|查看用户信息|优势：查看用户的 UID、GID、所属组；劣势：无法删除用户。删除用户前用 id 确认用户信息，删除用 userdel|
|`cat /etc/passwd`|查看用户信息|优势：查看所有用户信息；劣势：无法删除用户。查看用户信息用 cat /etc/passwd，删除用 userdel，两者常配合使用|

## 七、课后练习任务

**重要提示：所有练习仅在测试环境操作，绝对不要删除系统已存在的重要用户（如 root）！**

1. 基础删除练习：用`sudo useradd testuser`创建用户，再用`sudo userdel testuser`删除，用`cat /etc/passwd | grep testuser`验证是否删除，用`ls -ld /home/testuser`验证家目录是否保留。
2. 删除家目录练习：用`sudo useradd testuser`创建用户，再用`sudo userdel -r testuser`删除，用`ls -ld /home/testuser`验证家目录是否删除。
3. 强制删除练习：用`sudo useradd testuser`创建用户，再用`sudo userdel -f testuser`强制删除（仅在测试环境），验证效果。
4. 结合 useradd 练习：用`sudo useradd devuser`创建用户，再用`sudo userdel devuser`删除，查看删除前后的`/etc/passwd`。
5. 查看 /etc/passwd 练习：用`cat /etc/passwd`查看所有用户信息，确认刚删除的用户已不存在。
6. 批量删除练习：用循环批量创建`dev1`到`dev5`用户，再用循环批量删除，用`cat /etc/passwd | grep dev`验证。
7. 权限不足练习：尝试用普通用户删除用户，观察权限不足的报错；再用`sudo`删除，验证效果。
8. 帮助练习：用`userdel --help`查看帮助信息，浏览所有参数。