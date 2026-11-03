# Convert markdown to PDF

## Prerequisites

- [VS Code](https://code.visualstudio.com/) installed
- [Markdown PDF](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-pdf) extension installed

## Set US Letter Paper Size

1. Open Settings: `Ctrl+,` (Linux) / `Cmd+,` (macOS).
2. Search for `markdown-pdf.format`.
3. Set the value to `Letter`.

Or add this to `settings.json`:

```json
"markdown-pdf.format": "Letter"   
```

## Steps

1. Open your `.md` file in VS Code.
2. Right-click anywhere in the doc and select **Markdown PDF: Export (pdf)**. Or use the command palette: `Ctrl+Shift+P` (Linux) / `Cmd+Shift+P` (macOS), type **Markdown PDF: Export (pdf)** and select it.
3. A PDF is generated alongside your source file.
