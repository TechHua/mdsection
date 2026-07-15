# mdsection

`mdsection` 是一个使用 Rust 编写的命令行工具，用于处理单个 Markdown 文件的 section 查询和内容替换。

它支持：

- 查询匹配的 Markdown section；
- 在目标唯一时安全地原地替换整个 section；
- 替换标题行；
- 替换普通物理行。

> 本文中的 **section** 指由 hash-style 标题（`#`～`######`）及其所属内容构成的 Markdown 区域。

## 命令概览

```text
mdsection <COMMAND>

Commands:
  query    Find and print matching Markdown sections
  replace  Replace matching content in a Markdown file
  help     Print this message or the help of the given subcommand(s)
```

查看版本：

```powershell
mdsection --version
```

查看帮助：

```powershell
mdsection --help
mdsection query --help
mdsection replace --help
```

## `query`

查找并输出匹配的 Markdown section。

```text
mdsection query [OPTIONS] --path <PATH> --pattern <PATTERN>
```

### 参数

| 参数 | 说明 |
|---|---|
| `--path <PATH>` | Markdown 文件路径。必填。 |
| `--pattern <PATTERN>` | 用于匹配的字面文本。必填；空值或全空白值无效。 |
| `--scope <section-title\|section>` | 查询范围。默认为 `section`。 |
| `--output-mode <section-only\|section-with-ancestors>` | 输出匹配 subtree，或同时包含祖先上下文。默认为 `section-only`。 |
| `--max-level <1..6>` | 可识别的最大 hash 标题级别。默认为 `6`。 |

### 查询范围

#### `section`

在 section 的标题和内容中匹配。

这是默认范围：

```powershell
mdsection query `
  --path .\reference.md `
  --pattern validity
```

#### `section-title`

仅在 section 标题中匹配：

```powershell
mdsection query `
  --path .\reference.md `
  --pattern validity `
  --scope section-title
```

### 输出模式

`--output-mode` 适用于 `section` 和 `section-title` 查询范围。

#### `section-only`

只输出匹配的 section subtree。

这是默认模式。

#### `section-with-ancestors`

输出匹配的 section subtree，并包含其祖先标题上下文：

```powershell
mdsection query `
  --path .\reference.md `
  --pattern validity `
  --scope section-title `
  --output-mode section-with-ancestors
```

一次查询可以返回多个 section。成功时，标准输出只包含 Markdown 查询结果，标准错误为空。

## `replace`

在唯一目标命中时替换 Markdown 文件中的内容。

```text
mdsection replace [OPTIONS] --path <PATH> --pattern <PATTERN> --replacement <REPLACEMENT>
```

### 参数

| 参数 | 说明 |
|---|---|
| `--path <PATH>` | 需要原地修改的 Markdown 文件路径。必填。 |
| `--pattern <PATTERN>` | 用于匹配的字面文本。必填；空值或全空白值无效。 |
| `--replacement <REPLACEMENT>` | 替换文本。必填；精确值 `-` 表示从 UTF-8 标准输入读取到 EOF。 |
| `--scope <section-title\|section\|line>` | 替换范围。默认为 `section`。 |
| `--max-level <1..6>` | 源文档可识别的最大 hash 标题级别。默认为 `6`；`line` 不依赖标题解析。 |

三种替换范围均支持多行 replacement。

### 唯一目标约束

`replace` 仅在操作目标唯一时修改文件。

目标的计数方式取决于 `--scope`：

| Scope | 目标计数方式 |
|---|---|
| `section` | 按 section 计数。同一 section 内出现多行或多次匹配，仍视为一个目标。 |
| `section-title` | 按标题物理行计数。 |
| `line` | 按物理行计数，包括标题前内容、标题行和 fenced code block 内的行。 |

当没有目标命中时，命令返回 `no match`。

当多个目标命中时，命令列出对应的一组目标行号并拒绝修改文件。

### 替换语义

#### `section`

替换唯一匹配的整个 section。

当 replacement 为空时，删除整个 section。

当 replacement 非空时：

- 如果第一逻辑行是合法的 `#`～`######` 标题，则使用 replacement 替换整个 section；
- 否则保留原 section 标题，并使用 replacement 替换标题之后的全部 subtree，包括原有子 section。

保留原标题并替换其余内容：

```powershell
mdsection replace `
  --path .\reference.md `
  --pattern validity `
  --replacement "New section content"
```

从文件读取完整 replacement：

```powershell
Get-Content -Raw .\replacement.md |
  mdsection replace `
    --path .\reference.md `
    --pattern validity `
    --replacement -
```

如果 `replacement.md` 的第一行包含合法标题，则整个 section 会被替换。

#### `section-title`

使用完整 replacement 替换唯一匹配的标题物理行。

原有的 `#` 标题标记不会被自动保留，因此 replacement 应包含最终需要写入的完整标题行。

例如：

```powershell
mdsection replace `
  --path .\reference.md `
  --pattern "Old title" `
  --replacement "## New title" `
  --scope section-title
```

当 replacement 为空时，删除该标题行。

#### `line`

使用完整 replacement 替换唯一匹配的物理行：

```powershell
mdsection replace `
  --path .\reference.md `
  --pattern obsolete `
  --replacement "Current content" `
  --scope line
```

删除唯一匹配的物理行：

```powershell
mdsection replace `
  --path .\reference.md `
  --pattern obsolete `
  --replacement "" `
  --scope line
```

## 多行 replacement

可以通过 PowerShell here-string 传递多行内容：

````powershell
$replacement = @'
## Installation

Install the package:

```powershell
cargo install xxxxx
```
'@

mdsection replace `
  --path .\README.md `
  --pattern Installation `
  --replacement $replacement
````

也可以通过标准输入传递：

```powershell
Get-Content -Raw .\replacement.md |
  mdsection replace `
    --path .\README.md `
    --pattern Installation `
    --replacement -
```

`-` 表示从标准输入读取。

## 匹配规则

`mdsection` 使用以下匹配规则：

- pattern 是字面文本，不是正则表达式；
- 匹配不区分大小写；
- ASCII pattern 使用整词边界；
- CJK pattern 使用字面包含；
- section 解析仅识别 hash-style 标题；
- fenced code block 内形似标题的内容不会被识别为 section 标题。

例如，以下内容中的 `## Example` 不会被解析为 section：

````markdown
```text
## Example
```
````

使用 `--scope line` 时不进行 section 解析，因此 fenced code block 内的物理行仍可被匹配和替换。

## 文件写入

成功的 `replace` 操作不会直接分段覆盖原文件。

工具会在目标文件所在目录中创建临时文件，写入完整的新内容，然后提交替换结果。

该机制提供以下保证：

- 操作失败时原文件保持不变；
- 不会留下部分写入的文件；
- 不创建备份文件；
- 成功后目标文件在原路径完成更新。

## 输出与失败契约

### 成功

#### `query`

- exit code 为 `0`；
- stdout 只包含查询结果 Markdown；
- stderr 为空。

#### `replace`

- exit code 为 `0`；
- stdout 为空；
- stderr 为空；
- 目标文件完成原地更新。

### 失败

所有命令失败时：

- exit code 非零；
- stdout 为空；
- stderr 包含一条简短诊断。

`replace` 失败时，目标文件保持不变。

常见失败情况包括：

- 参数无效；
- pattern 为空或全为空白；
- 文件无法读取；
- replacement 指定为 `-`，但标准输入不是有效的 UTF-8；
- 没有操作目标命中；
- 多个操作目标命中；
- 无法安全提交文件修改。
