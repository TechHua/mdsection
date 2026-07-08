# markdown-section-query

`markdown-section-query` is a single-file Markdown section query CLI. It reads one Markdown file, matches recognized sections, and writes a filtered Markdown view to stdout.

This release is CLI-only. 

## Version

```powershell
markdown-section-query --version
```

## Help

```powershell
markdown-section-query --help
```

## Usage

```powershell
markdown-section-query `
  --path <path> `
  --pattern <text> `
  [--query-scope <section-title|section>] `
  [--output-mode <section-only|section-with-ancestors>] `
  [--max-level <level>]
```

## Arguments

- `--path <path>`: Markdown file path. Required.
- `--pattern <text>`: Literal query text. Required. Empty or whitespace-only values are invalid.
- `--query-scope <section-title|section>`: Query headings only or headings plus section body. Default: `section-title`.
- `--output-mode <section-only|section-with-ancestors>`: Output matched subtrees only or include ancestor context. Default: `section-only`.
- `--max-level <level>`: Maximum recognized hash heading level from `1` to `6`. Default: `6`.

## Output Contract

On success:

- stdout contains only the filtered Markdown.
- stderr is empty.
- exit code is `0`.

On failure:

- stdout is empty.
- stderr contains a short diagnostic.
- exit code is non-zero.

## Examples

Default title query:

```powershell
markdown-section-query `
  --path .\reference.md `
  --pattern validity
```

Include ancestor context:

```powershell
markdown-section-query `
  --path .\reference.md `
  --pattern validity `
  --query-scope section-title `
  --output-mode section-with-ancestors
```

Query headings and body text:

```powershell
markdown-section-query `
  --path .\reference.md `
  --pattern "pass-through variables" `
  --query-scope section `
  --output-mode section-with-ancestors
```
