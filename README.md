# php_exe - 静态编译 PHP 可执行程序

## 项目描述

php_exe 是一个**静态编译的 PHP 可执行程序**，专为**工作流编译**场景设计。本项目将 PHP 解释器及其所有依赖项静态链接为单个二进制文件，无需系统安装 PHP 运行时即可直接运行，极大简化了工作流引擎的部署和分发。

## 核心特性

- **完全静态编译**：所有依赖库（libc、libxml、openssl、zlib、mbstring 等）均静态链接，单个二进制文件即可运行
- **零依赖部署**：目标机器无需预装 PHP 及任何扩展库，拷贝即用
- **工作流专用**：针对工作流引擎的编译执行场景进行了优化，内置常用工作流处理扩展
- **跨平台兼容**：支持 Linux x86_64 / ARM64 架构
- **轻量级体积**：通过裁剪无用模块和 UPX 压缩，保持二进制体积最小化

## 适用场景

- **工作流引擎编译执行**：作为工作流系统的底层执行引擎，编译和运行工作流定义脚本
- **CI/CD 流水线**：在持续集成环境中执行 PHP 构建脚本，无需配置 PHP 环境
- **容器化部署**：在 Docker 等容器中作为单文件执行，减小镜像体积
- **离线环境**：在无网络连接的受限环境中运行 PHP 工作流任务
- **嵌入式系统**：在资源受限的嵌入式设备中执行 PHP 逻辑

## 内置扩展

工作流编译版本默认内置以下扩展：

| 扩展 | 说明 |
|------|------|
| Core | PHP 核心 |
| bcmath | 任意精度数学运算（工作流数值计算） |
| ctype | 字符类型检测 |
| date | 日期时间处理 |
| dom | DOM/XML 操作（工作流 XML 解析） |
| filter | 数据过滤验证 |
| gd | 图像处理（可选） |
| json | JSON 编解码（工作流数据交换） |
| mbstring | 多字节字符串处理 |
| openssl | 加密/签名（工作流安全校验） |
| pcre | 正则表达式 |
| PDO | 数据库抽象层 |
| pdo_sqlite | SQLite 驱动（工作流状态持久化） |
| phar | PHP 归档 |
| SimpleXML | 简单 XML 解析 |
| sockets | Socket 通信 |
| SPL | 标准 PHP 库 |
| sqlite3 | SQLite3 数据库 |
| tokenizer | 代码词法分析 |
| xml | XML 解析器 |
| xmlreader | XML 流读取 |
| xmlwriter | XML 流写入 |
| zip | ZIP 压缩（工作流打包） |
| zlib | 数据压缩 |

## 快速开始

### 获取可执行文件

从 Releases 页面下载对应架构的预编译二进制文件：

```bash
# 下载 Linux x86_64 版本
wget https://github.com/your-org/php_exe/releases/latest/download/php_exe-linux-x86_64

# 赋予执行权限
chmod +x php_exe-linux-x86_64

# 验证版本
./php_exe-linux-x86_64 -v
```

### 工作流编译示例

```bash
# 编译工作流定义脚本
./php_exe workflow_compile.php --input workflow-definition.json --output workflow-compiled.phar

# 执行已编译的工作流
./php_exe run-workflow.php workflow-compiled.phar
```

### 代码示例

```php
<?php
// workflow_compile.php - 工作流编译脚本

$inputFile = $argv[1] ?? 'workflow-definition.json';
$outputFile = $argv[2] ?? 'workflow.phar';

// 读取工作流定义
$definition = json_decode(file_get_contents($inputFile), true);

// 编译验证工作流节点
$compiled = compileWorkflow($definition);

// 打包为 Phar
$phar = new Phar($outputFile);
$phar->buildFromIterator(
    new ArrayIterator(['compiled.json' => json_encode($compiled)]),
    ''
);
$phar->setStub("#!/usr/bin/env php\n<?php Phar::mapPhar(); include 'phar://workflow.phar/compiled.json'; __HALT_COMPILER();");

echo "工作流编译完成: {$outputFile}\n";
```

## 编译构建

如需自行从源码编译静态 PHP 可执行程序：

```bash
# 克隆项目
git clone https://github.com/your-org/php_exe.git
cd php_exe

# 构建静态编译版本
./build.sh --with-workflow --arch=x86_64

# 输出位于
ls -la dist/php_exe
```

构建依赖：
- GCC / Clang 支持静态链接的 C 编译器
- make、autoconf、automake 等构建工具
- 必要的系统静态库（glibc-static 等）

## 版本说明

| 版本 | PHP 基础版本 | 说明 |
|------|-------------|------|
| 8.3.x-wf | PHP 8.3.x | 工作流编译稳定版 |
| 8.2.x-wf | PHP 8.2.x | 工作流编译 LTS 版 |

## License

MIT License

