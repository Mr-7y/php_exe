# php_exe - 静态编译 PHP 可执行程序（工作流编译器）

## 项目描述

php_exe 是一个基于 **[static-php-cli](https://static-php.dev/)** 的**静态编译 PHP 可执行程序**，专为**工作流编译**场景设计。项目使用 **GitHub Actions 工作流**自动构建，将 PHP 解释器及其所有依赖项静态链接为**单个二进制文件**，无需系统安装 PHP 运行时即可直接运行，极大简化了工作流引擎的部署和分发。

> **核心原理**：通过 GitHub Actions 提供的 Cloud Runner（Linux / Windows / macOS）触发 [.github/workflows/build-php.yml](.github/workflows/build-php.yml)，下载 static-php-cli 工具，读取 `craft.yml` 配置，自动下载 PHP 源码和依赖库并完成静态编译，最终产物通过 Artifacts 或 Release Assets 发布。

## 核心特性

- **完全静态编译**：所有依赖库（libc、libxml、openssl、zlib、mbstring 等）均静态链接，单个二进制文件即可运行
- **零依赖部署**：目标机器无需预装 PHP 及任何扩展库，拷贝即用（ODBC 除外，需目标机安装对应数据库驱动，见 3.6）
- **工作流专用**：针对工作流引擎的编译执行场景进行了优化，内置常用工作流处理扩展（含 ODBC / SQL Server / PostgreSQL / MySQL / SQLite 数据库驱动）
- **多平台自动构建**：通过 GitHub Actions 一次触发，同时生成 Linux x86_64 / Windows x64 / macOS x86_64 & ARM64 四个平台产物
- **参数化版本与扩展**：运行工作流时可自定义 PHP 版本和扩展列表，改完就跑，跑完就下
- **轻量级体积**：通过裁剪无用模块和 UPX 压缩，保持二进制体积最小化

## 适用场景

- **工作流引擎编译执行**：作为工作流系统的底层执行引擎，编译和运行工作流定义脚本
- **CI/CD 流水线**：在持续集成环境中执行 PHP 构建脚本，无需配置 PHP 环境
- **容器化部署**：在 Docker 等容器中作为单文件执行，减小镜像体积
- **离线环境**：在无网络连接的受限环境中运行 PHP 工作流任务
- **嵌入式系统**：在资源受限的嵌入式设备中执行 PHP 逻辑

---

## 一、GitHub Actions 工作流使用详解

### 1.1 工作流文件位置与作用

| 项目 | 说明 |
|------|------|
| **文件路径** | [.github/workflows/build-php.yml](.github/workflows/build-php.yml) |
| **名称** | `Build Static PHP CLI (Workflows Compiler)` |
| **作用** | 自动化调用 `static-php-cli`（简称 spc）对 PHP 源码进行**静态编译**，生成多平台的单文件 PHP CLI 可执行文件 |
| **触发方式** | `workflow_dispatch`（手动触发，可传参）、`push`（push 到 main 分支自动跑）、`release`（发布 Release 时自动上传产物） |
| **构建矩阵** | 4 个平台并发：Windows x64 / Linux x86_64 / macOS x86_64 (Intel) / macOS ARM64 (Apple Silicon) |
| **产物输出** | 通过 `actions/upload-artifact` 上传到 Run 页面，或在 release 事件中自动发布到 Release Assets |

### 1.2 工作流完整执行步骤

每个平台的 Job 都会执行以下流程：

```
┌─────────────────────────────────┐
│ 1. Checkout Repository          │ ← 拉取仓库代码
├─────────────────────────────────┤
│ 2. Setup PHP (for tools)        │ ← 用 shivammathur/setup-php 安装一个
│                                 │   临时 PHP 8.2，给 spc 工具自身用
├─────────────────────────────────┤
│ 3. Download static-php-cli      │ ← 下载对应平台的 spc 预编译二进制
│                                 │   （spc.exe / spc），执行 doctor --auto-fix
├─────────────────────────────────┤
│ 4. Generate craft.yml           │ ← 生成编译配置文件：指定 PHP 版本、
│                                 │   需要启用的扩展列表、SAPI（cli）
├─────────────────────────────────┤
│ 5. Build PHP CLI via craft      │ ← 执行 `spc craft`，自动下载 PHP 源码、
│                                 │   下载依赖库、编译、静态链接
├─────────────────────────────────┤
│ 6. Verify Build                 │ ← `php -v` 验证版本，`php -m` 打印扩展
├─────────────────────────────────┤
│ 7. Upload Artifact              │ ← 上传到 GitHub Actions Artifacts
├─────────────────────────────────┤
│ 8. Upload to Release (可选)     │ ← 如果是 release 触发，上传到 Release
└─────────────────────────────────┘
```

### 1.3 手动触发工作流（最简单的用法）

**操作步骤**：

1. 打开你的 GitHub 仓库页面，点击顶部的 **Actions** 标签
2. 左侧选择 **Build Static PHP CLI (Workflows Compiler)** 这个工作流
3. 点击右侧的 **Run workflow** 下拉按钮（蓝色）
4. 填写参数后点击 **Run workflow** 绿色按钮
5. 等待 10~30 分钟（扩展越多越久），Job 全部变绿后进入该次 Run 页面
6. 滚到最下方 **Artifacts** 区域，点击下载对应平台的 zip 包即可得到 `php.exe` 或 `php`

**可填写的参数**：

| 参数名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `php-version` | string | `8.4` | 要编译的 PHP 版本。支持：`8.4`、`8.3`、`8.2`、`8.1`（建议 8.2+） |
| `extensions` | string | 留空（= 内置全量列表，见 3.3） | 用**英文逗号**分隔的扩展名称列表，不要加空格。**留空**时按平台自动使用内置全量列表；**填写**时完全覆盖内置列表 |

---

## 二、不同 PHP 版本如何切换

### 2.1 方式 A：手动触发时直接填（临时用）

在 Actions → Run workflow 弹窗里，把 **PHP Version to build** 输入框改成你想要的版本号，例如填 `8.3` 或 `8.2`，然后运行。

### 2.2 方式 B：修改工作流默认值（长期用）

编辑 [.github/workflows/build-php.yml](.github/workflows/build-php.yml)，找到 `workflow_dispatch.inputs.php-version.default` 和所有 `inputs.php-version || '8.4'` 的回退值，改成你想要的默认版本：

```yaml
# 改这里
workflow_dispatch:
  inputs:
    php-version:
      default: '8.3'    # ← 改成默认 8.3

# 同时把每一处的回退值也改一下（搜索 inputs.php-version || '8.4'）
php-version: ${{ inputs.php-version || '8.3' }}
```

### 2.3 方式 C：直接修改 craft.yml（纯本地，不用 Actions 输入）

如果你不想每次手动填参数，直接在仓库根目录放一个固定的 `craft.yml`（并把工作流里 Generate craft.yml 那一步删掉或注释掉）：

```yaml
# craft.yml（放到仓库根目录即可）
php-version: 8.3
extensions: bcmath,ctype,dom,json,mbstring,openssl,pdo,pdo_sqlite,phar,simplexml,sockets,sqlite3,tokenizer,xml,xmlreader,xmlwriter,zip,zlib
sapi:
  - cli
```

### 2.4 版本兼容说明

| PHP 版本 | static-php-cli 支持 | 推荐场景 |
|----------|--------------------|----------|
| **8.4** | ✅ 完整支持（最新） | 尝鲜、新项目，需要最新语法（如属性钩子、Asymmetric Visibility） |
| **8.3** | ✅ 完整支持 | **工作流编译推荐**，稳定 + 新特性均衡 |
| **8.2** | ✅ 完整支持 | LTS 长期维护，老项目兼容性最佳 |
| **8.1** | ⚠️ 支持但不再推荐 | 仅用于兼容非常旧的代码（已停止官方安全更新） |
| **8.0 及以下** | ❌ | static-php-cli 不再维护，不要用 |

---

## 三、不同扩展如何安装 / 添加 / 删除

### 3.1 最简单的做法：触发工作流时改 extensions 参数

在 **Run workflow** 弹窗的 **Comma-separated extensions list** 输入框里，直接粘贴你想要的扩展列表，用英文逗号分隔，**不要有空格**。留空则使用 3.3 的内置全量列表；填写则完全覆盖内置列表。例如你只想用工作流精简版：

```
bcmath,ctype,dom,fileinfo,filter,json,mbstring,openssl,pdo,pdo_sqlite,phar,simplexml,sockets,sqlite3,tokenizer,xml,xmlreader,xmlwriter,zip,zlib
```

例如你想要 Windows 例程里的**最全版**：

```
amqp,apcu,bcmath,bz2,calendar,ctype,curl,dba,dom,ds,exif,ffi,fileinfo,filter,ftp,gd,iconv,igbinary,libxml,mbregex,mbstring,mysqli,mysqlnd,opcache,openssl,pdo,pdo_mysql,pdo_sqlite,pdo_sqlsrv,phar,rar,redis,session,shmop,simdjson,simplexml,soap,sockets,sodium,sqlite3,sqlsrv,ssh2,swow,sysvshm,tokenizer,xml,xmlreader,xmlwriter,yac,yaml,zip,zlib
```

### 3.2 改工作流默认扩展列表（长期固定）

默认列表不再写死在每一步里，而是集中放在 [.github/workflows/build-php.yml](.github/workflows/build-php.yml) 顶部的 `env:` 段：

```yaml
env:
  # 三平台（Linux / macOS / Windows）通用，含 odbc / pdo_odbc / 各类数据库驱动
  COMMON_EXTENSIONS: "amqp,apcu,bcmath,brotli,bz2,calendar,ctype,curl,dba,dom,ds,exif,fileinfo,filter,ftp,gd,gettext,gmp,iconv,igbinary,intl,libxml,lz4,maxminddb,mbregex,mbstring,msgpack,mongodb,mysqli,mysqlnd,odbc,opcache,openssl,pdo,pdo_mysql,pdo_odbc,pdo_sqlite,pdo_sqlsrv,phar,rar,readline,redis,session,shmop,simdjson,simplexml,soap,sockets,sodium,sqlite3,sqlsrv,ssh2,swow,sysvshm,tidy,tokenizer,xml,xmlreader,xmlwriter,xsl,xz,yaml,zip,zlib,zstd"
  # 仅 Linux / macOS 自动追加（Windows 上 pgsql 依赖 meson/python，CI 拉取 nuget 失败）
  UNIX_EXTRA_EXTENSIONS: "pcntl,pdo_pgsql,pgsql,posix,sysvmsg,sysvsem,ldap,imap,memcached,snmp,uuid"
  # 仅 Linux 自动追加
  LINUX_EXTRA_EXTENSIONS: "inotify"
```

> 为什么要按平台拆开：`pcntl`、`posix`、`imap`、`ldap`、`memcached`、`snmp`、`uuid`、`inotify` 等在 Windows 上不受 static-php-cli 支持；`pgsql`/`pdo_pgsql` 在 Windows 编译时需要临时拉取 Python/meson（CI 环境会因 nuget 失败），因此也放到 Unix 追加列表。放进通用列表会让 Windows Job 直接失败。

### 3.3 内置全量扩展清单（默认，含 ODBC）

`extensions` 参数留空时，工作流会按平台自动拼接以下三组列表（Linux 共 77 个，Windows 65 个）：

**三平台通用（COMMON_EXTENSIONS）**

```
amqp,apcu,bcmath,brotli,bz2,calendar,ctype,curl,dba,dom,ds,exif,fileinfo,filter,ftp,gd,gettext,gmp,iconv,igbinary,intl,libxml,lz4,maxminddb,mbregex,mbstring,msgpack,mongodb,mysqli,mysqlnd,odbc,opcache,openssl,pdo,pdo_mysql,pdo_odbc,pdo_sqlite,pdo_sqlsrv,phar,rar,readline,redis,session,shmop,simdjson,simplexml,soap,sockets,sodium,sqlite3,sqlsrv,ssh2,swow,sysvshm,tidy,tokenizer,xml,xmlreader,xmlwriter,xsl,xz,yaml,zip,zlib,zstd
```

**仅 Linux / macOS（UNIX_EXTRA_EXTENSIONS）**

```
pcntl,pdo_pgsql,pgsql,posix,sysvmsg,sysvsem,ldap,imap,memcached,snmp,uuid
```

**仅 Linux（LINUX_EXTRA_EXTENSIONS）**

```
inotify
```

| 分类 | 扩展 | 说明（工作流场景用途） |
|------|------|------------------------|
| **核心必备** | Core, date, filter, pcre, SPL, tokenizer | PHP 默认自带，不可移除 |
| **字符串/编码** | ctype, mbstring, mbregex, iconv, gettext, intl | 多字节字符串、国际化 |
| **JSON/XML** | dom, libxml, simplexml, xml, xmlreader, xmlwriter, xsl | 工作流定义文件读写、XSLT 转换 |
| **数据/时间** | bcmath, gmp, calendar | 数值计算（bcmath/gmp 大数）、日期调度 |
| **数据库/持久化** | pdo, pdo_mysql, mysqli, mysqlnd, pdo_sqlite, sqlite3, pdo_pgsql(Unix), pgsql(Unix), odbc, pdo_odbc, sqlsrv, pdo_sqlsrv | MySQL / SQLite / PostgreSQL / 通用 ODBC / SQL Server |
| **缓存/消息队列** | redis, apcu, memcached(Unix), amqp, mongodb, msgpack, igbinary | 缓存、分布式锁、消息队列、序列化 |
| **文件/归档/压缩** | fileinfo, phar, zip, zlib, bz2, xz, zstd, lz4, brotli, rar | 工作流包打包解包、状态压缩 |
| **网络/IO** | curl, ftp, sockets, ssh2, soap, readline, maxminddb | 远程调用、SFTP、SOAP、GeoIP |
| **图像处理** | gd, exif | 图片生成、缩略图、EXIF 读取 |
| **安全/加密** | openssl, sodium | 签名校验、加密传输 |
| **性能** | opcache, simdjson | 脚本加速编译执行、高性能 JSON |
| **协程/服务** | swow | 长驻工作流服务（协程） |
| **系统/CLI** | pcntl(Unix), posix(Unix), sysvmsg(Unix), sysvsem(Unix), sysvshm, shmop, inotify(Linux), session | 进程管理、信号、共享内存、文件监听 |
| **其他** | dba, ds, tidy, yaml, uuid(Unix), ldap(Unix), imap(Unix), snmp(Unix) | 按需启用 |

> Windows 版默认不含 `pgsql`/`pdo_pgsql`（编译需临时拉取 Python/meson，CI 会因 nuget 失败）。Windows 要连 PostgreSQL，可安装 psqlODBC 后走 `odbc`/`pdo_odbc`。
>
> 如果构建在下载阶段报 `Failed to get GitHub release API info` / `Failed to get GitHub latest release`，说明 spc 请求 GitHub API 被限流或服务故障。工作流已在 `craft.yml` 的 `download-options.custom-url` 中把主要依赖（PHP 源码 8.4.27、openssl、curl、icu、libzip、gd 相关库、PECL 扩展等）固定为 GitHub Release / PECL / php.net 直链，绕过 `api.github.com` 与 php.net 的版本查询接口；版本号写死在 [.github/workflows/build-php.yml](.github/workflows/build-php.yml) 中，需要升级依赖时手动改链接即可。注意：`php-src` 固定为 8.4.27，若通过 `php-version` 参数编译 8.3/8.2，需要同步删除该条 `php-src` 直链。若仍有个别构件走 API，可在仓库 `Settings → Secrets and variables → Actions` 添加名为 `SPC_GITHUB_TOKEN` 的 classic PAT（勾选 `public_repo`），工作流会自动使用它。**不要用内置 `GITHUB_TOKEN`**：它只能访问当前仓库，访问其他仓库会返回 404。

### 3.4 完整可用扩展速查表（static-php-cli 支持的全部扩展）

以 [static-php-cli 官方 Supported Extensions](https://static-php.dev/en/guide/extensions.html) 为准（v3 当前列表，名称全部小写、逗号分隔）：

**Linux / macOS / Windows 三平台均支持**

```
amqp,apcu,ast,bcmath,brotli,bz2,calendar,ctype,curl,dba,decimal,deepclone,dio,dom,ds,exif,fileinfo,filter,ftp,gd,gettext,gmp,gmssl,iconv,igbinary,imagick,intl,libxml,lz4,maxminddb,mbregex,mbstring,msgpack,mongodb,mysqli,mysqlnd,mysqlnd_ed25519,mysqlnd_parsec,odbc,opcache,openssl,opentelemetry,pcov,pdo,pdo_mysql,pdo_odbc,pdo_pgsql,pdo_sqlite,pdo_sqlsrv,pgsql,phar,rar,readline,redis,session,shmop,simdjson,simplexml,snappy,soap,sockets,sodium,sqlite3,sqlsrv,ssh2,swow,sysvshm,tidy,tokenizer,trader,uv,xlswriter,xml,xmlreader,xmlwriter,xsl,xz,yac,yaml,zip,zlib,zstd
```

**仅 Linux / macOS 支持**

```
clickhouse,event,excimer,fastchart,fastjson,grpc,http,imap,ldap,memcache,memcached,password-argon2,pcntl,posix,protobuf,raphf,rdkafka,snmp,swoole,swoole-hook-mysql,swoole-hook-odbc,swoole-hook-pgsql,swoole-hook-sqlite,spx,sysvmsg,sysvsem,uuid,xhprof
```

**仅 Linux 支持**

```
inotify
```

**仅 Windows 支持**

```
com_dotnet
```

> **提示**：
> - 遇到 `extension xxx is not supported on xxx OS` 报错，说明该扩展在当前平台不可用，从列表里移除即可。
> - `ffi` 在 Linux 默认的 musl 全静态目标下无法加载动态库，需改用 glibc 目标（如 `SPC_TARGET=native-native-gnu.2.17`）。
> - `parallel` 仅支持 ZTS 构建，本工作流是 NTS，不要添加。
> - `xdebug` 只能以共享扩展方式编译，不在本静态清单中。
> - `oci8` / `mcrypt` / `pdo_oci` 等不受 static-php-cli 支持，加了会失败。

### 3.5 添加自定义扩展的三种模式

**模式 1：只加官方支持的扩展** → 直接往 `extensions` 列表加名字即可，不用改别的。

**模式 2：想要禁用某些扩展减小体积** → 直接从 `extensions` 列表里移除名字即可。例如只想做极简工作流编译器，只要：

```
bcmath,ctype,json,mbstring,openssl,phar,sockets,tokenizer,xml,zip,zlib
```

体积能从 ~30MB 降到 ~15MB。

**模式 3：自己本地调试 / 快速试错** → 下载 spc 到本地，用命令行模式不用写 craft.yml：

```bash
# Linux/macOS 示例
curl -L https://dl.static-php.dev/v3/spc-bin/nightly/spc-linux-x86_64 -o spc && chmod +x spc
./spc doctor --auto-fix
./spc download --with-php="8.3"
./spc build "bcmath,ctype,json,mbstring,openssl,phar,zip,zlib" --build-cli
./buildroot/bin/php -v
```

```powershell
# Windows PowerShell 示例
Invoke-WebRequest -Uri "https://dl.static-php.dev/v3/spc-bin/nightly/spc-windows-x64.exe" -OutFile "spc.exe"
.\spc.exe doctor --auto-fix
.\spc.exe download --with-php="8.3"
.\spc.exe build "bcmath,ctype,json,mbstring,openssl,phar,zip,zlib" --build-cli
.\buildroot\bin\php.exe -v
```

### 3.6 ODBC / 数据库扩展的运行前提（重要）

`odbc`、`pdo_odbc` 已经静态编进二进制，`php -m` 里能看到；但 **ODBC 驱动本身不会被打包**，目标机器仍要安装并注册对应的数据库驱动：

| 目标数据库 | 需要安装的驱动 |
|------------|----------------|
| SQL Server | Microsoft ODBC Driver 18 for SQL Server |
| MySQL / MariaDB | MySQL Connector/ODBC 或 MariaDB Connector/ODBC |
| PostgreSQL | psqlODBC |
| Oracle | Oracle Instant Client + ODBC 驱动（`oci8` 本身无法静态编译，只能用 ODBC 方式） |

- **Windows**：驱动由系统 ODBC 管理器（`odbc32.dll`）加载，在“ODBC 数据源管理器 → 驱动程序”里确认已注册即可。
- **Linux / macOS**：驱动管理器（unixODBC）已编进二进制，但驱动 `.so` 需要安装并在 `/etc/odbcinst.ini`（或 `~/.odbcinst.ini`）里注册，可用 `odbcinst -q -d` 查看。
- **Linux 全静态（默认 musl）注意**：全静态二进制无法 `dlopen` 动态库，unixODBC 会加载驱动失败。若要在 Linux 上真正连接数据库，需要改用 glibc 目标（例如设置环境变量 `SPC_TARGET=native-native-gnu.2.17`），或设置 `SPC_MUSL_DYNAMIC=true` 让 libc 动态链接。仅验证 `php -m` 是否有扩展则不受影响。

PDO 连接示例：

```php
<?php
// SQL Server（ODBC Driver 18）
$pdo = new PDO(
    'odbc:Driver={ODBC Driver 18 for SQL Server};Server=host,1433;Database=db;TrustServerCertificate=yes',
    'user',
    'password'
);

// MySQL ODBC
$pdo = new PDO(
    'odbc:Driver={MySQL ODBC 8.0 Unicode Driver};Server=host;Database=db',
    'user',
    'password'
);
```

排查顺序：`php -m | grep -i odbc` → 确认系统驱动已安装 → `odbcinst -q -d` 确认已注册 → 再试连接。

---

## 四、快速开始（使用构建好的产物）

### 4.1 从 GitHub Actions Artifacts 下载

1. 打开仓库 → **Actions** → 点击最近一次成功的 Workflow Run
2. 页面底部 **Artifacts** 区域列出了 4 个平台的包：
   - `php-8.4-windows-cli` → Windows x64（里面是 `php.exe`）
   - `php-8.4-linux-x86_64-cli` → Linux x86_64（里面是 `php`）
   - `php-8.4-macos-x86_64-cli` → macOS Intel
   - `php-8.4-macos-arm64-cli` → macOS Apple Silicon (M1/M2/M3/M4)
3. 点击名字下载 zip，解压后得到二进制文件

### 4.2 本地使用

**Linux / macOS：**

```bash
chmod +x php
./php -v          # 验证版本
./php -m          # 查看所有已编译扩展
./php your-workflow-script.php
```

**Windows（PowerShell / cmd）：**

```powershell
.\php.exe -v
.\php.exe -m
.\php.exe your-workflow-script.php
```

### 4.3 工作流编译示例

```bash
# 1. 编译工作流定义脚本（JSON → Phar）
./php workflow_compile.php workflow-definition.json workflow-compiled.phar

# 2. 执行已编译工作流
./php run-workflow.php workflow-compiled.phar --env=prod
```

```php
<?php
// workflow_compile.php - 工作流定义编译脚本（把 JSON 定义编为 Phar）
$inputFile  = $argv[1] ?? 'workflow-definition.json';
$outputFile = $argv[2] ?? 'workflow.phar';

if (!file_exists($inputFile)) {
    fwrite(STDERR, "错误：找不到定义文件 {$inputFile}\n");
    exit(1);
}

$definition = json_decode(file_get_contents($inputFile), true);
if (json_last_error() !== JSON_ERROR_NONE) {
    fwrite(STDERR, "JSON 解析失败：" . json_last_error_msg() . "\n");
    exit(1);
}

// TODO: 这里放你的「工作流节点编译 / 依赖图校验 / 调度编排」逻辑
$compiled = [
    'version'   => '1.0',
    'compiled_at' => date('c'),
    'definition'  => $definition,
    'checksum'    => hash('sha256', json_encode($definition)),
];

@unlink($outputFile);
$phar = new Phar($outputFile);
$phar->buildFromIterator(
    new ArrayIterator([
        'compiled.json' => json_encode($compiled, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE),
        'run.php'       => '<?php $wf = json_decode(file_get_contents("phar://".__FILE__."/compiled.json"), true);'
                         .'echo "工作流 {$wf["checksum"]} 加载成功\n"; __HALT_COMPILER();',
    ]),
    ''
);
$phar->setStub(
    "#!/usr/bin/env php\n<?php\n"
    ."Phar::mapPhar(basename(__FILE__));\n"
    ."require 'phar://'.__FILE__.'/run.php';\n"
    ."__HALT_COMPILER();\n"
);
chmod($outputFile, 0755);

echo "✅ 工作流编译完成：{$outputFile}\n";
echo "   校验和：{$compiled['checksum']}\n";
```

---

## 五、本地手动构建（不用 GitHub Actions）

如果你不想用 GitHub Actions，也可以在自己电脑/服务器上直接跑 `static-php-cli`：

### 5.1 Linux（Ubuntu/Debian 示例）

```bash
# 1. 安装基础依赖
sudo apt-get update && sudo apt-get install -y curl wget git build-essential autoconf automake libtool pkg-config re2c bison

# 2. 下载 spc
curl -L https://dl.static-php.dev/v3/spc-bin/nightly/spc-linux-x86_64 -o spc
chmod +x spc

# 3. 环境检查（会自动安装缺失的编译工具）
./spc doctor --auto-fix

# 4. 下载 PHP 源码和依赖库
./spc download --with-php="8.3"

# 5. 构建（--build-cli 只编译 CLI SAPI）
./spc build "bcmath,ctype,curl,dom,fileinfo,json,mbstring,openssl,pdo,pdo_sqlite,phar,sockets,sqlite3,xml,zip,zlib" --build-cli

# 6. 产物
ls -lah buildroot/bin/php
./buildroot/bin/php -v
./buildroot/bin/php -m
```

### 5.2 Windows（PowerShell，建议用管理员身份）

```powershell
# 1. 下载 spc
Invoke-WebRequest -Uri "https://dl.static-php.dev/v3/spc-bin/nightly/spc-windows-x64.exe" -OutFile "spc.exe"

# 2. 自动修复环境（会下载 VS Build Tools 等）
.\spc.exe doctor --auto-fix

# 3. 下载源码
.\spc.exe download --with-php="8.3"

# 4. 构建
.\spc.exe build "bcmath,ctype,curl,dom,fileinfo,json,mbstring,openssl,pdo,pdo_mysql,pdo_sqlite,phar,sockets,sqlite3,xml,zip,zlib" --build-cli

# 5. 产物
.\buildroot\bin\php.exe -v
```

### 5.3 macOS

```bash
# Intel 芯片
curl -L https://dl.static-php.dev/v3/spc-bin/nightly/spc-darwin-x86_64 -o spc
# Apple Silicon 芯片（M1/M2/M3/M4）
# curl -L https://dl.static-php.dev/v3/spc-bin/nightly/spc-darwin-arm64 -o spc

chmod +x spc
./spc doctor --auto-fix
./spc download --with-php="8.3"
./spc build "bcmath,ctype,dom,fileinfo,json,mbstring,openssl,pdo,pdo_sqlite,phar,sockets,sqlite3,xml,zip,zlib" --build-cli
./buildroot/bin/php -v
```

---

## 六、常见问题 FAQ

**Q1：工作流跑失败，报 `doctor check failed` 怎么办？**
A：90% 是 Runner 环境缺编译工具。检查 `doctor --auto-fix` 那一步有没有成功；如果是缺少系统包，加一步 `apt-get install / brew install / choco install` 即可。

**Q2：报 `extension XXX is not supported / not found` 怎么处理？**
A：查上面第 3.4 节的**完整可用扩展列表**，拼写必须完全一致；如果报 OS 不支持（比如 Windows 下用 swoole），只能移除该扩展或换平台编译。

**Q3：扩展越多越好吗？**
A：不是。扩展越多：① 编译时间越长（GD / Intl / ImageMagick 特别慢）；② 二进制体积越大（加完常用的全套大概 40~80MB，精简版 12~20MB）；③ 某些扩展之间有版本冲突风险。工作流场景建议用第 3.3 节的**推荐列表**，够用即可。

**Q4：怎么减小二进制体积？**
A：① 只加必须的扩展；② 本地构建后加 `--UPX-compress` 参数（`spc build ... --build-cli --UPX-compress`）或手动跑 UPX：`upx --best --lzma buildroot/bin/php`，通常能再压缩 50%~70%。

**Q5：产物怎么发布到 Release？**
A：两种方式：① 在仓库 **Releases → Draft a new release → Publish release**，工作流会被 `release: created` 事件触发，自动把 4 个平台的产物上传到 Assets；② 每次成功 Run 后手动从 Artifacts 下载，再上传到 Release。

**Q6：怎么编译成 micro（SAPI），即把 PHP 代码和解释器打包成单个独立可执行文件？**
A：把工作流 craft.yml 里的 `sapi: [cli]` 改成 `sapi: [micro]`，然后用 `spc micro:combine` 命令把你的 PHP 脚本塞进去。工作流里可以加一步：`./spc micro:combine my-workflow-runner.php -O workflow-runner`，`workflow-runner` 就是一个双击就能跑 PHP 脚本的独立可执行文件，不含解释器启动开销，非常适合工作流分发。

---

## 七、版本说明

| 版本标识 | PHP 基础版本 | 说明 |
|----------|-------------|------|
| `8.4.x-wf` | PHP 8.4.x | 工作流编译最新版，尝鲜 |
| `8.3.x-wf` | PHP 8.3.x | **工作流编译推荐稳定版** |
| `8.2.x-wf` | PHP 8.2.x | 工作流编译 LTS 版（维护至 2025-12） |

---

## 八、文件结构

```
php_exe/
├── .github/
│   └── workflows/
│       └── build-php.yml          # ← GitHub Actions 自动构建工作流（核心文件）
└── README.md                       # ← 本说明文档
```

---

## License

MIT License
