# PDF Processor

The `holodeck.lib.pdf_processor` package provides PDF-specific processing utilities for text extraction with heading detection and page extraction. These operations are used by `FileProcessor` to deliver enhanced PDF handling beyond what markitdown offers natively.

## Overview

The package exposes two public functions:

| Function                    | Purpose                                                                      |
| --------------------------- | ---------------------------------------------------------------------------- |
| `extract_pdf_with_headings` | Extract text from a PDF while detecting headings via bookmarks or font sizes |
| `extract_pdf_pages`         | Extract a subset of pages from a PDF into a temporary file                   |

### Heading Detection Strategies

`extract_pdf_with_headings` supports two strategies, applied in priority order:

1. **Bookmark-based detection** (preferred) -- Uses PDF outline/bookmark entries to identify headings and their hierarchy levels. More reliable when the PDF contains bookmarks.
1. **Font-size-based detection** (fallback) -- Analyzes font sizes against configurable thresholds. Used automatically when bookmarks are absent or when bookmark matching falls below a 30% match rate.

The output is Markdown text with heading markers (`#`, `##`, etc.) suitable for downstream processing by tools such as `StructuredChunker`.

## Package Exports

PDF processing utilities for HoloDeck.

This package provides PDF-specific file processing operations:

- **Heading Extraction**: Text extraction using pdfminer that produces markdown with proper heading markers (##). Supports two strategies:
- **Bookmark-based detection** (preferred): Uses PDF outline/bookmark entries to identify headings and their hierarchy. More reliable when bookmarks are present.
- **Font-size-based detection** (fallback): Analyzes font sizes against configurable thresholds. Used when bookmarks are absent or disabled.
- **Page Extraction**: Extract specific pages from PDF files using pypdf, producing temporary PDF files with the selected pages.

These operations are used by FileProcessor to provide enhanced PDF handling beyond what markitdown offers natively.

Example

from holodeck.lib.pdf_processor import extract_pdf_with_headings, extract_pdf_pages

## Extract text with heading detection (bookmarks preferred, font-size fallback)

markdown = extract_pdf_with_headings(Path("document.pdf"))

## Force font-size-only detection

markdown = extract_pdf_with_headings(Path("document.pdf"), use_bookmarks=False)

## Extract specific pages

temp_path = extract_pdf_pages(Path("document.pdf"), pages=[0, 1, 2])

Functions:

| Name                        | Description                             |
| --------------------------- | --------------------------------------- |
| `extract_pdf_with_headings` | Extract PDF text with heading detection |
| `extract_pdf_pages`         | Extract specific pages from a PDF file  |

## `extract_pdf_with_headings(file_path, heading_thresholds=None, use_bookmarks=True)`

Extract PDF text with heading detection.

Prefers bookmark-based heading detection when bookmarks are available, falling back to font-size-based detection otherwise.

Parameters:

| Name                 | Type               | Description                                                      | Default                                                                                  |
| -------------------- | ------------------ | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `file_path`          | `Path`             | Path to PDF file                                                 | *required*                                                                               |
| `heading_thresholds` | \`dict[float, int] | None\`                                                           | Font size -> heading level mapping. Default: {14.0: 1, 12.0: 2} (14pt+ = h1, 12pt+ = h2) |
| `use_bookmarks`      | `bool`             | Whether to attempt bookmark-based detection first. Default: True | `True`                                                                                   |

Returns:

| Type  | Description                                                    |
| ----- | -------------------------------------------------------------- |
| `str` | Markdown text with heading markers based on detected headings. |

Raises:

| Type        | Description                                           |
| ----------- | ----------------------------------------------------- |
| `Exception` | If PDF parsing fails (caller should handle fallback). |

Source code in `src/holodeck/lib/pdf_processor/heading_extractor.py`

```
def extract_pdf_with_headings(
    file_path: Path,
    heading_thresholds: dict[float, int] | None = None,
    use_bookmarks: bool = True,
) -> str:
    """Extract PDF text with heading detection.

    Prefers bookmark-based heading detection when bookmarks are available,
    falling back to font-size-based detection otherwise.

    Args:
        file_path: Path to PDF file
        heading_thresholds: Font size -> heading level mapping.
            Default: {14.0: 1, 12.0: 2} (14pt+ = h1, 12pt+ = h2)
        use_bookmarks: Whether to attempt bookmark-based detection first.
            Default: True

    Returns:
        Markdown text with heading markers based on detected headings.

    Raises:
        Exception: If PDF parsing fails (caller should handle fallback).
    """
    if use_bookmarks:
        try:
            has_bookmarks, outlines = _has_bookmarks(file_path)
            if has_bookmarks:
                result = _extract_with_bookmarks(file_path, outlines)
                if result is not None:
                    return result
                logger.debug(
                    "Bookmark extraction returned None, "
                    "falling back to font-size detection"
                )
        except Exception:
            logger.debug(
                "Bookmark detection failed, falling back to font-size detection",
                exc_info=True,
            )

    return _extract_with_font_sizes(file_path, heading_thresholds)
```

## `extract_pdf_pages(file_path, pages)`

Extract specific pages from PDF into temporary file.

Parameters:

| Name        | Type        | Description                                 | Default    |
| ----------- | ----------- | ------------------------------------------- | ---------- |
| `file_path` | `Path`      | Path to original PDF file                   | *required* |
| `pages`     | `list[int]` | List of page numbers to extract (0-indexed) | *required* |

Returns:

| Type   | Description                                     |
| ------ | ----------------------------------------------- |
| `Path` | Path to temporary PDF file with extracted pages |

Raises:

| Type          | Description                                 |
| ------------- | ------------------------------------------- |
| `ImportError` | If pypdf is not installed                   |
| `ValueError`  | If page numbers are invalid or out of range |

Source code in `src/holodeck/lib/pdf_processor/page_extractor.py`

```
def extract_pdf_pages(file_path: Path, pages: list[int]) -> Path:
    """Extract specific pages from PDF into temporary file.

    Args:
        file_path: Path to original PDF file
        pages: List of page numbers to extract (0-indexed)

    Returns:
        Path to temporary PDF file with extracted pages

    Raises:
        ImportError: If pypdf is not installed
        ValueError: If page numbers are invalid or out of range
    """
    try:
        from pypdf import PdfReader, PdfWriter
    except ImportError as e:
        raise ImportError(
            "pypdf is required for PDF page extraction. "
            "Install with: pip install 'markitdown[all]'"
        ) from e

    logger.debug(f"Extracting pages {pages} from PDF: {file_path}")

    try:
        reader = PdfReader(str(file_path))
        writer = PdfWriter()
        total_pages = len(reader.pages)

        # Validate page numbers
        for page_num in pages:
            if page_num < 0 or page_num >= total_pages:
                raise ValueError(
                    f"Page {page_num} out of range (PDF has {total_pages} pages)"
                )

        # Extract specified pages
        for page_num in pages:
            writer.add_page(reader.pages[page_num])

        # Create temporary file
        tmp = tempfile.NamedTemporaryFile(suffix=".pdf", delete=False)  # noqa: SIM115
        tmp_path = Path(tmp.name)
        writer.write(tmp)
        tmp.close()

        logger.debug(f"Extracted {len(pages)} pages from PDF to temp file: {tmp_path}")
        return tmp_path

    except Exception as e:
        logger.error(f"PDF page extraction failed: {e}")
        raise
```

## Heading Extractor

## `extract_pdf_with_headings(file_path, heading_thresholds=None, use_bookmarks=True)`

Extract PDF text with heading detection.

Prefers bookmark-based heading detection when bookmarks are available, falling back to font-size-based detection otherwise.

Parameters:

| Name                 | Type               | Description                                                      | Default                                                                                  |
| -------------------- | ------------------ | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `file_path`          | `Path`             | Path to PDF file                                                 | *required*                                                                               |
| `heading_thresholds` | \`dict[float, int] | None\`                                                           | Font size -> heading level mapping. Default: {14.0: 1, 12.0: 2} (14pt+ = h1, 12pt+ = h2) |
| `use_bookmarks`      | `bool`             | Whether to attempt bookmark-based detection first. Default: True | `True`                                                                                   |

Returns:

| Type  | Description                                                    |
| ----- | -------------------------------------------------------------- |
| `str` | Markdown text with heading markers based on detected headings. |

Raises:

| Type        | Description                                           |
| ----------- | ----------------------------------------------------- |
| `Exception` | If PDF parsing fails (caller should handle fallback). |

Source code in `src/holodeck/lib/pdf_processor/heading_extractor.py`

```
def extract_pdf_with_headings(
    file_path: Path,
    heading_thresholds: dict[float, int] | None = None,
    use_bookmarks: bool = True,
) -> str:
    """Extract PDF text with heading detection.

    Prefers bookmark-based heading detection when bookmarks are available,
    falling back to font-size-based detection otherwise.

    Args:
        file_path: Path to PDF file
        heading_thresholds: Font size -> heading level mapping.
            Default: {14.0: 1, 12.0: 2} (14pt+ = h1, 12pt+ = h2)
        use_bookmarks: Whether to attempt bookmark-based detection first.
            Default: True

    Returns:
        Markdown text with heading markers based on detected headings.

    Raises:
        Exception: If PDF parsing fails (caller should handle fallback).
    """
    if use_bookmarks:
        try:
            has_bookmarks, outlines = _has_bookmarks(file_path)
            if has_bookmarks:
                result = _extract_with_bookmarks(file_path, outlines)
                if result is not None:
                    return result
                logger.debug(
                    "Bookmark extraction returned None, "
                    "falling back to font-size detection"
                )
        except Exception:
            logger.debug(
                "Bookmark detection failed, falling back to font-size detection",
                exc_info=True,
            )

    return _extract_with_font_sizes(file_path, heading_thresholds)
```

## Page Extractor

## `extract_pdf_pages(file_path, pages)`

Extract specific pages from PDF into temporary file.

Parameters:

| Name        | Type        | Description                                 | Default    |
| ----------- | ----------- | ------------------------------------------- | ---------- |
| `file_path` | `Path`      | Path to original PDF file                   | *required* |
| `pages`     | `list[int]` | List of page numbers to extract (0-indexed) | *required* |

Returns:

| Type   | Description                                     |
| ------ | ----------------------------------------------- |
| `Path` | Path to temporary PDF file with extracted pages |

Raises:

| Type          | Description                                 |
| ------------- | ------------------------------------------- |
| `ImportError` | If pypdf is not installed                   |
| `ValueError`  | If page numbers are invalid or out of range |

Source code in `src/holodeck/lib/pdf_processor/page_extractor.py`

```
def extract_pdf_pages(file_path: Path, pages: list[int]) -> Path:
    """Extract specific pages from PDF into temporary file.

    Args:
        file_path: Path to original PDF file
        pages: List of page numbers to extract (0-indexed)

    Returns:
        Path to temporary PDF file with extracted pages

    Raises:
        ImportError: If pypdf is not installed
        ValueError: If page numbers are invalid or out of range
    """
    try:
        from pypdf import PdfReader, PdfWriter
    except ImportError as e:
        raise ImportError(
            "pypdf is required for PDF page extraction. "
            "Install with: pip install 'markitdown[all]'"
        ) from e

    logger.debug(f"Extracting pages {pages} from PDF: {file_path}")

    try:
        reader = PdfReader(str(file_path))
        writer = PdfWriter()
        total_pages = len(reader.pages)

        # Validate page numbers
        for page_num in pages:
            if page_num < 0 or page_num >= total_pages:
                raise ValueError(
                    f"Page {page_num} out of range (PDF has {total_pages} pages)"
                )

        # Extract specified pages
        for page_num in pages:
            writer.add_page(reader.pages[page_num])

        # Create temporary file
        tmp = tempfile.NamedTemporaryFile(suffix=".pdf", delete=False)  # noqa: SIM115
        tmp_path = Path(tmp.name)
        writer.write(tmp)
        tmp.close()

        logger.debug(f"Extracted {len(pages)} pages from PDF to temp file: {tmp_path}")
        return tmp_path

    except Exception as e:
        logger.error(f"PDF page extraction failed: {e}")
        raise
```

## Usage Examples

### Extract text with heading detection

```
from pathlib import Path
from holodeck.lib.pdf_processor import extract_pdf_with_headings

# Bookmark-preferred extraction (default)
markdown = extract_pdf_with_headings(Path("document.pdf"))

# Force font-size-only detection
markdown = extract_pdf_with_headings(Path("document.pdf"), use_bookmarks=False)

# Custom font-size thresholds (18pt+ = h1, 14pt+ = h2, 12pt+ = h3)
markdown = extract_pdf_with_headings(
    Path("document.pdf"),
    heading_thresholds={18.0: 1, 14.0: 2, 12.0: 3},
)
```

### Extract specific pages

```
from pathlib import Path
from holodeck.lib.pdf_processor import extract_pdf_pages

# Extract pages 0, 1, and 4 (0-indexed) into a temporary PDF
temp_path = extract_pdf_pages(Path("large-report.pdf"), pages=[0, 1, 4])

# Use the temporary file for further processing
print(f"Extracted pages saved to: {temp_path}")
```

## Dependencies

| Dependency     | Used By             | Purpose                                                     |
| -------------- | ------------------- | ----------------------------------------------------------- |
| `pdfminer.six` | `heading_extractor` | PDF text and font-size extraction, bookmark/outline parsing |
| `pypdf`        | `page_extractor`    | PDF page-level read/write operations                        |
