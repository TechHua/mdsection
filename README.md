# mdsection

`mdsection` 是一个处理单个 Markdown 文件的 Rust CLI。它可以查询匹配的 section，也可以在目标唯一时安全地原地替换 section、标题行或普通物理行。

## 版本

```powershell
mdsection --version
```

## 命令

```text
mdsection <COMMAND>

Commands:
  query    Find and print matching Markdown sections
  replace  Replace matching content in a Markdown file
  help     Print this message or the help of the given subcommand(s)
```

### query

```text
mdsection query [OPTIONS] --path <PATH> --pattern <PATTERN>
```

- `--path <PATH>`：Markdown 文件路径，必填。
- `--pattern <PATTERN>`：字面匹配内容，必填；空值或全空白无效。
- `--scope <section-title|section>`：查询范围，默认 `section`。
- `--output-mode <section-only|section-with-ancestors>`：只输出匹配 subtree，或同时包含祖先上下文；默认 `section-only`。
- `--max-level <1..6>`：可识别的最大 hash 标题级别，默认 `6`。

`query` 可以返回多个 section。成功时 stdout 只包含 Markdown，stderr 为空。

### replace

```text
mdsection replace [OPTIONS] --path <PATH> --pattern <PATTERN> --replacement <REPLACEMENT>
```

- `--path <PATH>`：需要原地修改的 Markdown 文件路径，必填。
- `--pattern <PATTERN>`：字面匹配内容，必填；空值或全空白无效。
- `--replacement <REPLACEMENT>`：替换文本，必填；精确值 `-` 表示从 UTF-8 stdin 读取到 EOF。
- `--scope <section-title|section|line>`：替换范围，默认 `section`。
- `--max-level <1..6>`：源文档可识别的最大 hash 标题级别，默认 `6`；`line` 不依赖标题解析。

`replace` 只在唯一操作目标命中时写文件：

- `section` 按唯一 section 计数；同一 section 内多行或多次命中仍是一个目标。
- `section-title` 按标题行计数。
- `line` 按物理行计数，包括标题前内容、标题行和 fenced code block 内的行。
- 零目标时返回 no match；多目标时列出一基目标行号并拒绝修改。

替换范围语义：

- `section`：空 replacement 删除整个 section；非空 replacement 的第一逻辑行是合法 `#`～`######` 标题时替换整个 section，否则保留原标题并替换其余 subtree，包括原有子 section。
- `section-title`：用完整 replacement 替换整条标题物理行，不保留原 `#`；空值删除该行。
- `line`：用完整 replacement 替换唯一命中的物理行；空值删除该行。
- 三种 scope 都允许多行 replacement。

成功的 `replace` 不向 stdout 或 stderr 输出内容。写入使用目标文件同目录的临时文件完成完整内容提交；失败时原文件保持不变，不创建备份文件。

## 匹配规则

- pattern 是字面文本，不是正则表达式。
- 匹配不区分大小写。
- ASCII pattern 使用整词边界。
- CJK pattern 使用字面包含。
- section 解析只识别 hash-style 标题，并排除 fenced code block 内的伪标题。

## 示例

查询标题或正文包含 `validity` 的 section：

```powershell
mdsection query `
  --path .\reference.md `
  --pattern validity
```

只查询标题并包含祖先上下文：

```powershell
mdsection query `
  --path .\reference.md `
  --pattern validity `
  --scope section-title `
  --output-mode section-with-ancestors
```

保留命中 section 的原标题并替换其余内容：

```powershell
mdsection replace `
  --path .\reference.md `
  --pattern validity `
  --replacement "New section content"
```

从管道读取完整 section；输入第一行带标题，因此替换整个 section：

```powershell
Get-Content -Raw .\replacement.md |
  mdsection replace `
    --path .\reference.md `
    --pattern validity `
    --replacement -
```

删除唯一匹配的物理行：

```powershell
mdsection replace `
  --path .\reference.md `
  --pattern obsolete `
  --replacement "" `
  --scope line
```

## 输出与失败契约

成功时：

- `query`：exit code 为 `0`，stdout 只有查询结果 Markdown，stderr 为空。
- `replace`：exit code 为 `0`，stdout 和 stderr 都为空，文件原地更新。

失败时：

- exit code 非零。
- stdout 为空。
- stderr 包含一条简短诊断。
- `replace` 不改变目标文件。
