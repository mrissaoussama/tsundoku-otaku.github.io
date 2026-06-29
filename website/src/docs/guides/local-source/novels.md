---
title: Novels from Local source
titleTemplate: Local source - Guides
description: Import specifications for novels
---

# Local source Novels

If you like to download and organize your novels, then you want to know how to manage your own series in Tsundoku.
This explains how to import local novels, including EPUB volumes, and keep chapters in the expected order.

::: warning
This page explores some advanced features.
:::

## Importing EPUBs with the import screen

The easiest way to add EPUBs is the in-app import screen, which previews and organizes files before importing. You do not have to place files manually.

Open it from the **Library** overflow menu (**Import EPUB**), or send an EPUB to Tsundoku from another app.

### Opening EPUBs from another app

From a file manager, browser, or any app, use **Share** or **Open with → Tsundoku** on an `.epub` file. It opens straight in the import screen.

### Novel groups and volumes

Selected files are parsed and grouped into **novel groups**. Each group becomes one novel. The files inside it are its **volumes**. The app guesses the grouping and order from EPUB metadata and filenames.

For each group you can:

- Edit the **novel title**.
- **Reorder volumes** inside a group with the up/down arrows.
- Edit a **volume title** (shown only when a group has more than one volume).
- **Switch a volume** to another group, or to a new group, with the move (folder) menu.
- **Remove** a volume or a whole group.
- Expand a volume to preview its metadata and table of contents.

A group with more than one volume is imported as a single combined novel. The volumes keep their order.

Use **Auto rename & organize** in the top bar to re-group and rename everything from EPUB metadata again.

At the bottom, there is the option to **Add to library automatically** with a **category** picker. Otherwise, the imported EPUBs will be found in the Local Novels option in Browse tab.

## Base Folder

If you prefer a manual import or want to use other formats, set your local novels base folder in the app and place one folder per novel:

```text
LocalNovels/
  My Novel Title/
    ...chapter files or folders...
```

`My Novel Title` becomes the novel title shown in the app. Be careful not to rename this after adding this to your library, as the title is used to find the corresponding folder.

## Supported Chapter Formats

The local novel source accepts these chapter formats:

- Plain text: `.txt`, `.text`
- Markdown: `.md`
- HTML: `.html`, `.htm`, `.xhtml`
- EPUB: `.epub`
- ZIP archives: `.zip`, `.cbz`
- RAR archives: `.rar`, `.cbr`
- Directories containing text/html files

Hidden files/folders (starting with `.`) are ignored.

## Import Patterns

### 1) One chapter per file

Use multiple files directly inside the novel folder:

```text
LocalNovels/
  My Novel Title/
    001 - Prologue.txt
    002 - Chapter 1.txt
    003 - Chapter 2.html
```

Appearance in chapter list:

- One file = one chapter row.
- Chapter title comes from filename (without extension).
- Chapter number is parsed from filename when possible.

### 2) One chapter per folder

Use chapter folders that contain text/html pages:

```text
LocalNovels/
  My Novel Title/
    001 - Chapter 1/
      01.xhtml
      02.xhtml
    002 - Chapter 2/
      page1.html
      page2.html
```

Files inside each folder are read in filename order.

Appearance in chapter list:

- One folder = one chapter row (not one row per file inside).
- The chapter title is the folder name.
- File contents inside the folder are concatenated in filename order into one reader page text stream.

### 3) One chapter per archive

Use zip/rar containers with text/html files:

```text
LocalNovels/
  My Novel Title/
    001 - Chapter 1.zip
    002 - Chapter 2.cbz
```

Text/html entries inside archives are read in filename order.

Appearance in chapter list:

- One archive = one chapter row.
- Archive entries are concatenated in filename order when opened.

### 4) EPUB as single chapter

A single-chapter EPUB is imported as one chapter:

```text
LocalNovels/
  My Novel Title/
    001 - Side Story.epub
```

Appearance in chapter list:

- One EPUB file = one chapter row.
- Chapter title defaults to EPUB filename (or metadata when available).

### 5) EPUB with internal TOC chapters (multi-chapter EPUB)

If an EPUB has multiple TOC entries, each TOC entry becomes a separate chapter.

```text
LocalNovels/
  My Novel Title/
    Volume 01.epub
```

The app reads each TOC chapter by its internal reference.

Appearance in chapter list:

- One TOC entry = one chapter row.
- Titles come from TOC labels.
- Chapter numbers are generated as sequential values to preserve global reading order.
- When multiple EPUB volume files exist, chapter titles are prefixed with the EPUB filename for context.

## Multi-Volume EPUB Ordering

For multi-volume novels where each volume is an EPUB with multiple TOC chapters:

- Volume file order is determined by filename natural order.
- TOC chapter order is kept within each volume.
- TOC chapters are assigned continuous chapter numbers (1, 2, 3...) in file order.
- Final chapter list is ordered by chapter number, then by chapter title.

Recommended naming:

```text
Volume 01.epub
Volume 02.epub
Volume 03.epub
```

or

```text
001 - Volume 1.epub
002 - Volume 2.epub
003 - Volume 3.epub
```

Avoid unnumbered names like only `Part A.epub`, `Part B.epub` unless lexical order is intended.

## Exporting novels to EPUB

You can export a novel back out to an EPUB file.

- **Single novel** - open the novel, select chapters, and choose **Export EPUB**.
- **Multiple novels** - select novels in the library and use the batch **Export EPUB** action.

In the export dialog you can set the filename and choose:

- **Downloaded only**, or pull from the source.
- **Join volumes** into one file, or keep them separate (produces a zip).
- **Translation mode**: original, translated, or both (both produces a zip with two files).
- **Filename options**: include chapter count, chapter range, status, or volume number.
- **Include custom CSS / JS**: from your active snippets.

## Managing local novels
With the select feature in browse Local novels, you can:
- **Delete** - remove a local novel (and its files).
- **Refresh Covers**

## Metadata Notes

- EPUB metadata can populate chapter/novel fields.
- Cover image may be extracted from EPUB (embedded or external URL).
- The last EPUB's cover will be used as cover if none exists
- If metadata is missing, filename-based values are used.

## Troubleshooting

- Wrong order: add zero-padded numbers (`001`, `002`, `003`) to filenames.

## Mixed Content Rules

- Mixed files in the same novel folder are supported: `.epub`, `.txt/.html`, archives, and chapter folders can coexist.
- Ordering is still based on top-level filename natural order.
- EPUB files inside a chapter folder are not treated as EPUB chapters; folder mode reads only text/html files inside that folder.
