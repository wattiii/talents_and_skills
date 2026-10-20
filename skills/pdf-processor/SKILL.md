---
name: pdf-processor
description: PDF processing workflows: extraction, transformation, and automation patterns
version: 1.0.0
author: James Watt, Hermes Agent
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [pdf, document, automation]
    category: productivity
---

# PDF Processor Skill

## When to Use

- Extracting text or data from PDF documents
- Transforming PDF content for further processing
- Automating repetitive PDF tasks

## Procedure

### Text Extraction

```bash
# Extract text from PDF using pymupdf
python3 -c "
import fitz
doc = fitz.open('document.pdf')
for page in doc:
    text = page.get_text()
    print(text)
"
```

### PDF to Markdown Conversion

```bash
# Convert PDF to markdown using pandoc
pandoc document.pdf -o document.md

# Or using Python with pdfminer.six
python3 -c "
from pdfminer.high_level import extract_text
text = extract_text('document.pdf')
with open('document.txt', 'w') as f:
    f.write(text)
"
```

### Batch PDF Processing

```bash
# Process all PDFs in a directory
for f in *.pdf; do
    python3 -c "
import fitz, sys
doc = fitz.open('$f')
text = doc[0].get_text()
with open('$f.txt', 'w') as out:
    out.write(text)
"
done
```

## Pitfalls

- **Scanned PDFs** — OCR required (use pytesseract or pdfpc)
- **Large PDFs** — extract pages iteratively, don't load entire doc into memory
- **Password-protected PDFs** — handle decryption separately, don't embed credentials in scripts

## Verification

```bash
# Test extraction on a sample PDF
python3 -c "
import fitz
doc = fitz.open('sample.pdf')
print(f'Pages: {len(doc)}')
print(f'First page text length: {len(doc[0].get_text())}')
"
```

Expected: prints page count and first page text length > 0.

# Common Commands Reference

| Command                                                 | Purpose             |
| ------------------------------------------------------- | ------------------- |
| `python3 -c "import fitz; doc = fitz.open('file.pdf')"` | Open PDF            |
| `doc[page].get_text()`                                  | Extract page text   |
| `pandoc file.pdf -o file.md`                            | Convert to markdown |
| `pdfgrep -i pattern file.pdf`                           | Search text in PDF  |
