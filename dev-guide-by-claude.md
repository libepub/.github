# The Complete EPUB & Kindle Developer Guide
### *A Practical Handbook for the libepub Project — Transforming Bengali Islamic Books into Digital Editions*

> **Who is this for?** You're a developer (or aspiring one) who can write basic HTML and CSS, and wants to produce professional `.epub` and `.mobi`/`.kfx` files from scratch. We'll walk through everything — file structure, XML internals, Bengali text support, accessibility, and automated GitHub Actions releases.

---

## Table of Contents

1. [What Even Is an EPUB?](#1-what-even-is-an-epub)
2. [EPUB 3 vs EPUB 2 — Which One Should You Use?](#2-epub-3-vs-epub-2)
3. [The Anatomy of an EPUB File](#3-the-anatomy-of-an-epub-file)
4. [Required XML Files Explained](#4-required-xml-files-explained)
5. [Writing Content in HTML/XHTML](#5-writing-content-in-htmlxhtml)
6. [CSS for EPUB — It's Not Like Regular Web CSS](#6-css-for-epub)
7. [Bengali Text, Unicode & RTL Support](#7-bengali-text-unicode--rtl-support)
8. [Images, Fonts & Media](#8-images-fonts--media)
9. [Building the EPUB Manually (Step by Step)](#9-building-the-epub-manually)
10. [Validation — Never Skip This](#10-validation)
11. [Converting to Kindle (KFX/MOBI)](#11-converting-to-kindle)
12. [The libepub Repository Structure](#12-the-libepub-repository-structure)
13. [GitHub Actions: Automated EPUB Releases](#13-github-actions-automated-epub-releases)
14. [Common Mistakes & How to Fix Them](#14-common-mistakes--how-to-fix-them)
15. [Recommended Tools](#15-recommended-tools)
16. [Quick Reference Cheatsheet](#16-quick-reference-cheatsheet)

---

## 1. What Even Is an EPUB?

An EPUB file (`.epub`) is just a **ZIP archive** with a very specific folder structure and a set of XML files that tell e-readers:

- What content is inside (the manifest)
- What order to display it in (the spine)
- Metadata like title, author, language, direction

You can literally rename any `.epub` to `.zip`, extract it, and browse the files. Try it — it's eye-opening.

```
my-book.epub  →  rename to →  my-book.zip  →  unzip  →  folder with HTML, CSS, XML
```

E-readers like Kindle, Kobo, iBooks, and Calibre parse these files according to the **EPUB specification** maintained by the W3C.

---

## 2. EPUB 3 vs EPUB 2

| Feature | EPUB 2 | EPUB 3 |
|---|---|---|
| HTML version | XHTML 1.1 | XHTML5 / HTML5 |
| CSS support | Limited CSS 2.1 | Full CSS 3 |
| Audio/Video | Not supported | Supported |
| JavaScript | Not supported | Supported |
| Accessibility | Basic | Full ARIA support |
| RTL languages | Partial | Full support |
| Bengali support | Works but limited | Works great |
| Kindle compatibility | Very high | High (with conversion) |

**Use EPUB 3.** It's the current standard, it handles Bengali and Arabic script properly, and all modern e-readers support it. Kindle will automatically convert it.

---

## 3. The Anatomy of an EPUB File

Here is what a complete, valid EPUB 3 folder looks like **before zipping**:

```
my-book/
├── mimetype                          ← MUST be first file, uncompressed
├── META-INF/
│   └── container.xml                 ← Points to the OPF file
└── OEBPS/                            ← All your actual content lives here
    ├── content.opf                   ← The master manifest/metadata file
    ├── toc.ncx                       ← Legacy table of contents (EPUB 2 compat)
    ├── nav.xhtml                     ← EPUB 3 navigation document
    ├── css/
    │   └── styles.css                ← Your stylesheet
    ├── fonts/
    │   ├── Kalpurush.ttf             ← Bengali font (embedded)
    │   └── Kalpurush.woff2
    ├── images/
    │   └── cover.jpg
    └── Text/
        ├── cover.xhtml
        ├── chapter01.xhtml
        ├── chapter02.xhtml
        └── ...
```

> **Why OEBPS?** It stands for *Open eBook Publication Structure*. It's a convention. Some tools use `content/` or `epub/` instead — all are valid as long as `container.xml` points to the right place.

---

## 4. Required XML Files Explained

### 4.1 `mimetype`

This is the simplest file in the entire EPUB. It contains exactly one line, no newline at the end:

```
application/epub+zip
```

**Rules:**
- Must be the **first file** in the ZIP
- Must be stored **uncompressed** (no deflate)
- Must have **no BOM** (byte order mark)
- Must have **no newline** at the end

If you mess up this file, e-readers will reject your EPUB. We'll handle the zipping correctly in the build script later.

---

### 4.2 `META-INF/container.xml`

This file is the entry point. Every EPUB reader opens this first to find where the actual book content is.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<container version="1.0" 
           xmlns="urn:oasis:names:tc:opendocument:xmlns:container">
  <rootfiles>
    <rootfile full-path="OEBPS/content.opf" 
              media-type="application/oebps-package+xml"/>
  </rootfiles>
</container>
```

That `full-path` attribute must exactly match where your `.opf` file lives. Double check capitalization — it's case-sensitive.

---

### 4.3 `OEBPS/content.opf` — The Master File

This is the most important file. OPF stands for *Open Packaging Format*. It contains:

1. **Metadata** — title, author, language, ISBN, publication date
2. **Manifest** — a list of every single file in the EPUB
3. **Spine** — the reading order

Here's a fully annotated example for a Bengali Islamic book:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<package version="3.0"
         xmlns="http://www.idpf.org/2007/opf"
         unique-identifier="book-id"
         xml:lang="bn"
         dir="ltr">

  <!-- ═══════════════════════════════════ -->
  <!--           METADATA                  -->
  <!-- ═══════════════════════════════════ -->
  <metadata xmlns:dc="http://purl.org/dc/elements/1.1/">

    <!-- Required: Unique ID. Use ISBN if you have it, or generate a UUID -->
    <dc:identifier id="book-id">urn:uuid:a1b2c3d4-e5f6-7890-abcd-ef1234567890</dc:identifier>

    <!-- Required: Title in Bengali -->
    <dc:title>রিয়াদুস সালেহীন</dc:title>

    <!-- Required: Language code (bn = Bengali) -->
    <dc:language>bn</dc:language>

    <!-- Required: Last modified timestamp — update this on every build -->
    <meta property="dcterms:modified">2025-04-16T00:00:00Z</meta>

    <!-- Optional but highly recommended -->
    <dc:creator id="author">ইমাম নববী (রহ.)</dc:creator>
    <dc:publisher>libepub</dc:publisher>
    <dc:description>হাদীসের বিখ্যাত সংকলন গ্রন্থ রিয়াদুস সালেহীন-এর বাংলা অনুবাদ।</dc:description>
    <dc:subject>ইসলাম</dc:subject>
    <dc:subject>হাদীস</dc:subject>
    <dc:rights>Public Domain</dc:rights>
    <dc:date>2025-04-16</dc:date>

    <!-- Cover image reference — must match manifest item id -->
    <meta name="cover" content="cover-image"/>

  </metadata>

  <!-- ═══════════════════════════════════ -->
  <!--           MANIFEST                  -->
  <!-- List EVERY file in the EPUB here    -->
  <!-- ═══════════════════════════════════ -->
  <manifest>

    <!-- Navigation document (EPUB 3 required) -->
    <item id="nav"
          href="nav.xhtml"
          media-type="application/xhtml+xml"
          properties="nav"/>

    <!-- Legacy NCX for older readers -->
    <item id="ncx"
          href="toc.ncx"
          media-type="application/x-dtbncx+xml"/>

    <!-- Stylesheet -->
    <item id="css"
          href="css/styles.css"
          media-type="text/css"/>

    <!-- Cover image -->
    <item id="cover-image"
          href="images/cover.jpg"
          media-type="image/jpeg"
          properties="cover-image"/>

    <!-- Content pages -->
    <item id="cover"
          href="Text/cover.xhtml"
          media-type="application/xhtml+xml"/>

    <item id="chapter01"
          href="Text/chapter01.xhtml"
          media-type="application/xhtml+xml"/>

    <item id="chapter02"
          href="Text/chapter02.xhtml"
          media-type="application/xhtml+xml"/>

    <!-- Fonts -->
    <item id="font-kalpurush-ttf"
          href="fonts/Kalpurush.ttf"
          media-type="font/ttf"/>

    <item id="font-kalpurush-woff2"
          href="fonts/Kalpurush.woff2"
          media-type="font/woff2"/>

  </manifest>

  <!-- ═══════════════════════════════════ -->
  <!--           SPINE                     -->
  <!-- Reading order — idref must match    -->
  <!-- manifest item ids above             -->
  <!-- ═══════════════════════════════════ -->
  <spine toc="ncx">
    <itemref idref="cover" linear="yes"/>
    <itemref idref="chapter01" linear="yes"/>
    <itemref idref="chapter02" linear="yes"/>
  </spine>

</package>
```

> **Important:** If a file is in the EPUB folder but NOT in the manifest, it's invisible to e-readers and the EPUB is technically invalid. Every single file must be listed.

---

### 4.4 `OEBPS/nav.xhtml` — The EPUB 3 Navigation

This is both a human-readable table of contents AND a machine-readable navigation structure. It replaces the old `toc.ncx` for EPUB 3, though we keep `toc.ncx` too for backward compatibility.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:epub="http://www.idpf.org/2007/ops"
      xml:lang="bn"
      lang="bn"
      dir="ltr">
<head>
  <meta charset="UTF-8"/>
  <title>বিষয়সূচি</title>
  <link rel="stylesheet" type="text/css" href="css/styles.css"/>
</head>
<body>

  <nav epub:type="toc" id="toc">
    <h1>বিষয়সূচি</h1>
    <ol>
      <li><a href="Text/cover.xhtml">প্রচ্ছদ</a></li>
      <li>
        <a href="Text/chapter01.xhtml">অধ্যায় ১: নিয়তের গুরুত্ব</a>
        <ol>
          <!-- Nested entries for sub-sections -->
          <li><a href="Text/chapter01.xhtml#section-1">হাদীস ১</a></li>
          <li><a href="Text/chapter01.xhtml#section-2">হাদীস ২</a></li>
        </ol>
      </li>
      <li><a href="Text/chapter02.xhtml">অধ্যায় ২: তওবা</a></li>
    </ol>
  </nav>

  <!-- Landmarks help accessibility tools navigate -->
  <nav epub:type="landmarks">
    <ol>
      <li><a epub:type="cover" href="Text/cover.xhtml">প্রচ্ছদ</a></li>
      <li><a epub:type="toc" href="nav.xhtml#toc">বিষয়সূচি</a></li>
      <li><a epub:type="bodymatter" href="Text/chapter01.xhtml">মূল বিষয়</a></li>
    </ol>
  </nav>

</body>
</html>
```

---

### 4.5 `OEBPS/toc.ncx` — Legacy Navigation (Keep for Compatibility)

Older Kindles and EPUB 2 readers use this file. Always include it.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ncx xmlns="http://www.daisy.org/z3986/2005/ncx/"
     version="2005-1"
     xml:lang="bn">

  <head>
    <!-- Must match the dc:identifier in content.opf -->
    <meta name="dtb:uid" content="urn:uuid:a1b2c3d4-e5f6-7890-abcd-ef1234567890"/>
    <meta name="dtb:depth" content="2"/>
    <meta name="dtb:totalPageCount" content="0"/>
    <meta name="dtb:maxPageNumber" content="0"/>
  </head>

  <docTitle>
    <text>রিয়াদুস সালেহীন</text>
  </docTitle>

  <navMap>
    <navPoint id="navpoint-1" playOrder="1">
      <navLabel><text>প্রচ্ছদ</text></navLabel>
      <content src="Text/cover.xhtml"/>
    </navPoint>

    <navPoint id="navpoint-2" playOrder="2">
      <navLabel><text>অধ্যায় ১: নিয়তের গুরুত্ব</text></navLabel>
      <content src="Text/chapter01.xhtml"/>
    </navPoint>

    <navPoint id="navpoint-3" playOrder="3">
      <navLabel><text>অধ্যায় ২: তওবা</text></navLabel>
      <content src="Text/chapter02.xhtml"/>
    </navPoint>
  </navMap>

</ncx>
```

---

## 5. Writing Content in HTML/XHTML

EPUB content files are **XHTML**, not regular HTML. The difference matters:

| Regular HTML | XHTML (EPUB) |
|---|---|
| Tags can be unclosed: `<br>` | All tags must be closed: `<br/>` |
| Attribute values optional quotes | All attributes must be quoted |
| `<html>` | Must declare namespaces |
| `&` is fine in text | Must use `&amp;` |
| Case-insensitive tags | Tags must be lowercase |
| `<img>` | `<img/>` (self-closing) |

### A Complete Content Page Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:epub="http://www.idpf.org/2007/ops"
      xml:lang="bn"
      lang="bn"
      dir="ltr">
<head>
  <meta charset="UTF-8"/>
  <title>অধ্যায় ১: নিয়তের গুরুত্ব</title>
  <link rel="stylesheet" type="text/css" href="../css/styles.css"/>
</head>
<body>

  <!-- epub:type attributes help e-readers understand content semantics -->
  <section epub:type="chapter" role="doc-chapter">

    <h1 class="chapter-title" id="chapter01">নিয়তের গুরুত্ব</h1>

    <!-- Arabic text (RTL) embedded inside Bengali (LTR) content -->
    <p class="arabic-hadith" dir="rtl" lang="ar">
      إِنَّمَا الأَعْمَالُ بِالنِّيَّاتِ
    </p>

    <p class="hadith-translation">
      "নিশ্চয়ই সকল আমল নিয়তের উপর নির্ভরশীল।"
    </p>

    <p class="hadith-source">
      — সহীহ বুখারী: ১, সহীহ মুসলিম: ১৯০৭
    </p>

    <p>
      ইমাম নববী (রহ.) এই হাদীসটি দিয়ে তাঁর বই শুরু করেছেন কারণ...
    </p>

    <!-- Section anchor for deep linking from TOC -->
    <section id="section-1" epub:type="subchapter">
      <h2>হাদীস ১</h2>
      <p>...</p>
    </section>

  </section>

</body>
</html>
```

### Common `epub:type` Values You'll Use

```
epub:type="cover"          → Cover page
epub:type="titlepage"      → Title page
epub:type="toc"            → Table of contents
epub:type="chapter"        → A chapter
epub:type="subchapter"     → Sub-section
epub:type="footnote"       → Footnotes
epub:type="endnotes"       → Endnotes section
epub:type="bibliography"   → References
epub:type="index"          → Index
epub:type="glossary"       → Glossary
epub:type="preface"        → Preface
epub:type="introduction"   → Introduction
epub:type="appendix"       → Appendix
```

---

## 6. CSS for EPUB

EPUB CSS is 90% normal CSS3, with some important caveats. Here's a production-ready stylesheet for a Bengali Islamic book:

```css
/* ============================================
   LIBEPUB — Base Stylesheet for Bengali Books
   Tested: Kindle, Kobo, iBooks, Calibre
   ============================================ */

/* ── Font Declarations ── */
@font-face {
  font-family: 'Kalpurush';
  src: url('../fonts/Kalpurush.woff2') format('woff2'),
       url('../fonts/Kalpurush.ttf') format('truetype');
  font-weight: normal;
  font-style: normal;
}

@font-face {
  font-family: 'Kalpurush';
  src: url('../fonts/Kalpurush-Bold.woff2') format('woff2'),
       url('../fonts/Kalpurush-Bold.ttf') format('truetype');
  font-weight: bold;
  font-style: normal;
}

/* ── Reset & Base ── */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Kalpurush', 'SolaimanLipi', 'Vrinda', serif;
  font-size: 1em;
  line-height: 1.8;           /* Bengali text needs generous line height */
  color: #1a1a1a;
  text-align: justify;
  -epub-hyphens: auto;        /* EPUB-specific hyphenation */
  hyphens: auto;
  orphans: 2;                 /* Prevent orphan lines at page breaks */
  widows: 2;
}

/* ── Page Layout ── */
/* EPUB doesn't have page margins in CSS — use body padding */
/* E-readers apply their own margins; be conservative here */
body {
  padding: 0.5em 0;
}

/* ── Headings ── */
h1, h2, h3, h4, h5, h6 {
  font-family: 'Kalpurush', serif;
  font-weight: bold;
  line-height: 1.4;
  text-align: center;
  /* Force page break before chapter titles */
  page-break-before: always;
  -epub-page-break-before: always;
  break-before: page;
  margin-bottom: 1.5em;
  margin-top: 1em;
}

h1.chapter-title {
  font-size: 1.6em;
  margin-top: 2em;
  padding-top: 0.5em;
  border-top: 2px solid #555;
}

h2 {
  font-size: 1.3em;
  page-break-before: avoid;
  -epub-page-break-before: avoid;
  break-before: avoid;
}

h3 {
  font-size: 1.1em;
}

/* Prevent page breaks AFTER headings (looks terrible) */
h1, h2, h3 {
  page-break-after: avoid;
  -epub-page-break-after: avoid;
  break-after: avoid;
}

/* ── Paragraphs ── */
p {
  margin-bottom: 0.8em;
  text-indent: 1.5em;
}

/* First paragraph after a heading needs no indent */
h1 + p,
h2 + p,
h3 + p,
.no-indent {
  text-indent: 0;
}

/* ── Arabic Hadith Text ── */
.arabic-hadith {
  font-family: 'Amiri', 'KFGQPC Uthmanic Script', 'Traditional Arabic', 'Arial Unicode MS', serif;
  font-size: 1.4em;
  line-height: 2.2;
  text-align: right;
  direction: rtl;
  unicode-bidi: bidi-override;
  margin: 1.5em 1em;
  color: #2c2c2c;
  text-indent: 0;
  border-right: 3px solid #8b6914;
  padding-right: 0.8em;
}

/* ── Hadith Translation & Source ── */
.hadith-translation {
  font-style: italic;
  text-align: center;
  text-indent: 0;
  margin: 0.5em 2em 0.3em;
  color: #333;
}

.hadith-source {
  text-align: center;
  font-size: 0.85em;
  color: #666;
  text-indent: 0;
  margin-bottom: 1.2em;
}

/* ── Footnotes ── */
.footnote {
  font-size: 0.8em;
  line-height: 1.5;
  color: #555;
  margin-top: 0.3em;
  text-indent: 0;
}

/* For inline footnote references */
sup.fn-ref {
  font-size: 0.7em;
  vertical-align: super;
  color: #0055aa;
}

/* ── Cover Page ── */
.cover-page {
  text-align: center;
  page-break-after: always;
  -epub-page-break-after: always;
  break-after: page;
}

.cover-image {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 0 auto;
}

/* ── Block Quotes ── */
blockquote {
  margin: 1em 2em;
  padding: 0.5em 1em;
  border-left: 3px solid #aaa;
  font-style: italic;
  color: #444;
}

/* ── Tables ── */
/* Kindle doesn't handle complex tables well — keep them simple */
table {
  width: 100%;
  border-collapse: collapse;
  margin: 1em 0;
  font-size: 0.9em;
}

th, td {
  border: 1px solid #ccc;
  padding: 0.4em 0.6em;
  text-align: right; /* RTL-friendly default */
}

th {
  background-color: #f0f0f0;
  font-weight: bold;
}

/* ── AVOID these CSS properties in EPUB ──
   - position: fixed / absolute (unpredictable)
   - float (unreliable on all readers)
   - overflow: hidden (hides content on some readers)
   - background images (inconsistent support)
   - CSS Grid (poor Kindle support as of 2024)
   - CSS Variables (poor older Kindle support)
   ──────────────────────────────────────── */

/* ── Utilities ── */
.center { text-align: center; }
.bold { font-weight: bold; }
.italic { font-style: italic; }
.small { font-size: 0.85em; }
.page-break {
  page-break-before: always;
  -epub-page-break-before: always;
  break-before: page;
}
```

### CSS Rules You Must Follow in EPUB

1. **No fixed/absolute positioning** — e-readers reflow content
2. **No floats for layout** — use them only for inline images with caution
3. **Always use relative units** (`em`, `%`) not pixels (`px`) — screen sizes vary wildly from phone to e-ink display
4. **Use vendor prefixes** — `-epub-hyphens`, `-epub-page-break-*` alongside the standard versions
5. **No CSS Grid for main layout** — Kindle support is terrible
6. **No CSS Custom Properties (variables)** — older Kindles don't support them
7. **Test on actual devices** — CSS that looks great in Calibre can look broken on a Kindle Paperwhite

---

## 7. Bengali Text, Unicode & RTL Support

This is where libepub is doing something genuinely important. Bengali text in digital books has historically been a mess. Here's how to do it right.

### Unicode is Non-Negotiable

All Bengali text must be in **Unicode (UTF-8)**, not legacy encodings like ANSI Bijoy or ASCII-Bengali. If the source book uses Bijoy encoding, you must convert it first.

```bash
# Convert Bijoy-encoded text to Unicode using avro-phonetic tools
# Or use: https://www.avro.software/convert

# Verify a file is UTF-8
file -i chapter01.xhtml
# Should output: text/html; charset=utf-8
```

### Declaring Language and Direction

```xml
<!-- On the html element -->
<html xml:lang="bn" lang="bn" dir="ltr">

<!-- Bengali reads left-to-right -->
<!-- When embedding Arabic (RTL), always declare per element: -->
<p dir="rtl" lang="ar" xml:lang="ar">...</p>
```

### Bidirectional (BiDi) Text — Mixing Bengali and Arabic

Islamic books commonly mix Bengali (LTR) and Arabic (RTL). The `dir` attribute and Unicode BiDi controls handle this:

```xml
<!-- Arabic inside Bengali paragraph -->
<p lang="bn">
  রাসুল (সা.) বলেছেন, 
  <span class="arabic-inline" dir="rtl" lang="ar">إِنَّمَا الأَعْمَالُ بِالنِّيَّاتِ</span>
  — অর্থাৎ, আমল নিয়তের উপর নির্ভরশীল।
</p>
```

```css
.arabic-inline {
  font-family: 'Amiri', serif;
  font-size: 1.2em;
  direction: rtl;
  unicode-bidi: isolate; /* CSS3 BiDi isolation */
  display: inline-block;
  vertical-align: middle;
}
```

### Bengali Numerals

Bengali books may use Bengali-script numerals (০১২৩...) or Arabic numerals (0123...). Be consistent throughout a book. If the source uses Bengali numerals, keep them — they're proper Unicode characters and display fine.

```
Bengali: ০ ১ ২ ৩ ৪ ৫ ৬ ৭ ৮ ৯
Arabic:  0 1 2 3 4 5 6 7 8 9
```

### Special Bengali Characters & Hasanta

```
‌  →  U+200C (Zero Width Non-Joiner) — breaks unwanted conjuncts
‍  →  U+200D (Zero Width Joiner)    — forces conjunct formation
্  →  U+09CD (Hasanta/Virama)        — consonant killer
```

### Recommended Bengali Fonts for EPUB

| Font | License | Notes |
|---|---|---|
| **Kalpurush** | SIL OFL | Best for body text, very readable |
| **SolaimanLipi** | Free | Very common, high compatibility |
| **Nikosh** | SIL OFL | Clean and modern |
| **Mitra Mono** | SIL OFL | Monospace, for code |
| **Akaash** | GPL | Old but reliable |

For Arabic text within Bengali books:

| Font | License | Notes |
|---|---|---|
| **Amiri** | SIL OFL | Beautiful, traditional Naskh |
| **Scheherazade New** | SIL OFL | Very legible, full diacritics |
| **KFGQPC Hafs** | Free for non-commercial | Quran-specific, beautiful |

---

## 8. Images, Fonts & Media

### Cover Image

The cover image is the most visible part of your book. Kindle and most stores require:

- Format: **JPEG** (preferred) or PNG
- Minimum size: **1600 × 2400 pixels** (portrait)
- Aspect ratio: **2:3** (width:height)
- File size: Under **3MB**
- Color mode: **RGB** (not CMYK)

```xml
<!-- In content.opf manifest -->
<item id="cover-image"
      href="images/cover.jpg"
      media-type="image/jpeg"
      properties="cover-image"/>

<!-- In Text/cover.xhtml -->
<div class="cover-page">
  <img src="../images/cover.jpg" 
       alt="রিয়াদুস সালেহীন — বাংলা অনুবাদ"
       class="cover-image"/>
</div>
```

### Embedding Fonts

Embedding fonts ensures your Bengali typography looks exactly right everywhere. Always embed both TTF (compatibility) and WOFF2 (performance):

```xml
<!-- In content.opf manifest — one entry per font file -->
<item id="font-kalpurush"
      href="fonts/Kalpurush.ttf"
      media-type="font/ttf"/>
<item id="font-kalpurush-woff2"
      href="fonts/Kalpurush.woff2"
      media-type="font/woff2"/>
```

```css
/* In styles.css */
@font-face {
  font-family: 'Kalpurush';
  src: url('../fonts/Kalpurush.woff2') format('woff2'),
       url('../fonts/Kalpurush.ttf') format('truetype');
  font-weight: normal;
  font-style: normal;
}
```

> **Font Licensing Warning:** Before embedding any font in a redistributable EPUB, verify the license allows embedding. SIL OFL fonts like Kalpurush explicitly allow embedding. Proprietary fonts typically do not.

### Image Guidelines for Inline Images

```xml
<!-- Always provide alt text in Bengali for accessibility -->
<figure>
  <img src="../images/masjid.jpg" 
       alt="মদীনা মুনাওওয়ারার মসজিদে নববী"
       style="max-width: 100%; height: auto;"/>
  <figcaption>মসজিদে নববী, মদীনা</figcaption>
</figure>
```

- Use `max-width: 100%` so images scale on small screens
- NEVER use fixed pixel widths for images
- PNG for diagrams/illustrations, JPEG for photos
- Optimize images before adding — large images bloat the EPUB

---

## 9. Building the EPUB Manually

### Step 1: Create the Folder Structure

```bash
mkdir -p my-book/{META-INF,OEBPS/{Text,css,fonts,images}}
```

### Step 2: Write the `mimetype` File

```bash
# The echo -n avoids adding a newline — critical!
echo -n "application/epub+zip" > my-book/mimetype
```

### Step 3: Write All XML and XHTML Files

Create `container.xml`, `content.opf`, `nav.xhtml`, `toc.ncx`, and all your chapter XHTML files using the templates from sections 4 and 5.

### Step 4: Pack It Into a ZIP

The order matters. `mimetype` must be first and uncompressed:

```bash
cd my-book

# Add mimetype first, uncompressed (store mode, no deflate)
zip -X0 ../output.epub mimetype

# Add everything else (with compression)
zip -rX9 ../output.epub META-INF OEBPS

# Rename to .epub
mv ../output.epub ../my-book.epub
```

**If you're on Windows:**
```powershell
# Use 7-Zip or a dedicated EPUB tool
# The mimetype order/compression requirement means WinZip doesn't work reliably
```

### Step 5: Automate with a Build Script

Create `build.sh` in the repo root:

```bash
#!/bin/bash
set -e  # Exit on any error

BOOK_NAME="${1:-output}"
SOURCE_DIR="./src"
OUTPUT_DIR="./dist"

echo "📚 Building EPUB: $BOOK_NAME"

mkdir -p "$OUTPUT_DIR"
OUTPUT_FILE="$OUTPUT_DIR/${BOOK_NAME}.epub"

# Remove old file if exists
rm -f "$OUTPUT_FILE"

# Step 1: mimetype first, uncompressed
cd "$SOURCE_DIR"
zip -X0 "../$OUTPUT_FILE" mimetype

# Step 2: Everything else, compressed
zip -rX9 "../$OUTPUT_FILE" META-INF OEBPS --exclude "*.DS_Store" --exclude "*Thumbs.db"
cd ..

echo "✅ Built: $OUTPUT_FILE"
echo "📏 Size: $(du -sh "$OUTPUT_FILE" | cut -f1)"
```

```bash
chmod +x build.sh
./build.sh riyad-us-saleheen
```

---

## 10. Validation

**Never skip validation.** A broken EPUB will be rejected by Kindle Direct Publishing, Apple Books, and other stores.

### EPUBCheck — The Official Validator

```bash
# Install Java (required)
sudo apt install default-jre  # Ubuntu/Debian

# Download EPUBCheck
wget https://github.com/w3c/epubcheck/releases/download/v5.1.0/epubcheck-5.1.0.zip
unzip epubcheck-5.1.0.zip

# Validate your EPUB
java -jar epubcheck-5.1.0/epubcheck.jar dist/riyad-us-saleheen.epub
```

**What EPUBCheck Catches:**
- Missing manifest entries
- Broken internal links and anchors
- Invalid XML/XHTML (unclosed tags, bad namespaces)
- Wrong media types
- Missing required metadata
- Encoding issues

**A passing validation looks like:**
```
Validating using EPUB version 3.3 rules.
No errors or warnings detected.
Messages: 0 fatals / 0 errors / 0 warnings / 0 infos
```

### Accessibility Check — ACE by DAISY

```bash
# Install
npm install -g @daisy/ace

# Run
ace dist/riyad-us-saleheen.epub --outdir ace-report/
```

This checks WCAG 2.1 accessibility — important for users with visual impairments who use screen readers.

### Adding Validation to Your Build Script

```bash
#!/bin/bash
# In build.sh, after creating the EPUB:

echo "🔍 Validating..."
java -jar /opt/epubcheck/epubcheck.jar "$OUTPUT_FILE"

if [ $? -eq 0 ]; then
  echo "✅ Validation passed!"
else
  echo "❌ Validation FAILED. Fix errors before releasing."
  exit 1
fi
```

---

## 11. Converting to Kindle

Amazon Kindle uses its own format (KFX for newer devices, MOBI for older ones). You have two paths:

### Path A: Upload EPUB Directly to KDP

Amazon's Kindle Direct Publishing (KDP) accepts EPUB 3 and converts it automatically. This is the recommended path for publishing. Just make sure your EPUB validates first.

### Path B: Use Calibre or KindleGen Locally

For local testing and development:

```bash
# Install Calibre (includes ebook-convert CLI)
sudo apt install calibre

# Convert EPUB to MOBI (for older Kindles)
ebook-convert input.epub output.mobi \
  --output-profile kindle \
  --mobi-file-type new \
  --embed-all-fonts

# Convert EPUB to AZW3 (for newer Kindles)
ebook-convert input.epub output.azw3 \
  --output-profile kindle_pw3 \
  --embed-all-fonts
```

### Kindle-Specific CSS Considerations

When targeting Kindle specifically, add these to your stylesheet:

```css
/* Kindle uses a custom rendering engine called "Book Hype" */
/* These help normalize rendering across Kindle generations */

/* Kindle ignores margins on body — use padding instead */
body {
  padding: 0;
  margin: 0;
}

/* Kindle has its own font size controls — don't fight them */
/* Use em units everywhere */
p { font-size: 1em; }

/* Kindle Fire (color) supports more CSS than e-ink Kindles */
/* Test on both */

/* Force Kindle to use embedded fonts — without this it may use its own */
@font-face {
  font-family: 'Kalpurush';
  src: url('../fonts/Kalpurush.ttf');
  -webkit-font-smoothing: antialiased;
}
```

### Testing on Kindle Without a Device

Use the **Kindle Previewer 3** app (free from Amazon) or the **Kindle Cloud Reader** in your browser. Both let you preview your book as it would appear on different Kindle models.

---

## 12. The libepub Repository Structure

For the libepub project, each book gets its own repository under the `libepub` GitHub organization. Here's the standardized structure:

```
libepub/riyad-us-saleheen/         ← GitHub repo: libepub/riyad-us-saleheen
├── README.md                      ← Book info, build instructions, contributors
├── build.sh                       ← Local build script
├── validate.sh                    ← Validation script
├── metadata.json                  ← Machine-readable book metadata
├── CHANGELOG.md                   ← Version history
├── .github/
│   └── workflows/
│       └── release.yml            ← GitHub Actions release workflow
└── src/
    ├── mimetype
    ├── META-INF/
    │   └── container.xml
    └── OEBPS/
        ├── content.opf
        ├── nav.xhtml
        ├── toc.ncx
        ├── css/
        │   └── styles.css
        ├── fonts/
        │   ├── Kalpurush.ttf
        │   └── Kalpurush.woff2
        ├── images/
        │   └── cover.jpg
        └── Text/
            ├── cover.xhtml
            ├── chapter01.xhtml
            ├── chapter02.xhtml
            └── ...
```

### `metadata.json` — Standardized Across All Books

```json
{
  "id": "riyad-us-saleheen",
  "title": "রিয়াদুস সালেহীন",
  "title_transliterated": "Riyad us-Saleheen",
  "author": "ইমাম নববী (রহ.)",
  "translator": "মাওলানা আবদুস শহীদ নাসিম",
  "language": "bn",
  "source_language": "ar",
  "category": "hadith",
  "isbn": null,
  "publisher": "libepub",
  "license": "public-domain",
  "version": "1.0.0",
  "epubcheck_version": "5.1.0"
}
```

---

## 13. GitHub Actions: Automated EPUB Releases

This is the automation backbone of libepub. Every time you push a version tag (like `v1.0.0`) to a book repository, the workflow automatically:

1. Builds the EPUB
2. Validates it with EPUBCheck
3. Creates a GitHub Release
4. Uploads the `.epub` as a release artifact

### `.github/workflows/release.yml`

```yaml
name: Build and Release EPUB

# Trigger on version tags like v1.0.0, v1.2.3
on:
  push:
    tags:
      - 'v*.*.*'
  # Also allow manual triggering from the GitHub UI
  workflow_dispatch:
    inputs:
      version:
        description: 'Version tag (e.g. v1.0.0)'
        required: true
        default: 'v1.0.0'

jobs:
  build-epub:
    name: Build, Validate & Release
    runs-on: ubuntu-latest

    steps:
      # ── Checkout the repo ──
      - name: Checkout repository
        uses: actions/checkout@v4

      # ── Set up Java (required for EPUBCheck) ──
      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      # ── Extract book name and version ──
      - name: Extract metadata
        id: meta
        run: |
          BOOK_ID=$(cat metadata.json | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
          BOOK_TITLE=$(cat metadata.json | python3 -c "import sys,json; print(json.load(sys.stdin)['title_transliterated'])")
          VERSION="${GITHUB_REF_NAME:-${{ github.event.inputs.version }}}"
          FILENAME="${BOOK_ID}-${VERSION}.epub"
          echo "book_id=$BOOK_ID" >> $GITHUB_OUTPUT
          echo "book_title=$BOOK_TITLE" >> $GITHUB_OUTPUT
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "filename=$FILENAME" >> $GITHUB_OUTPUT

      # ── Build the EPUB ──
      - name: Build EPUB
        run: |
          mkdir -p dist
          cd src
          # mimetype must be first, uncompressed (store)
          zip -X0 "../dist/${{ steps.meta.outputs.filename }}" mimetype
          # Everything else with maximum compression
          zip -rX9 "../dist/${{ steps.meta.outputs.filename }}" META-INF OEBPS \
            --exclude "*.DS_Store" \
            --exclude "*/Thumbs.db" \
            --exclude "*/.gitkeep"
          cd ..
          echo "Build complete: dist/${{ steps.meta.outputs.filename }}"
          ls -lh dist/

      # ── Download and run EPUBCheck ──
      - name: Download EPUBCheck
        run: |
          EPUBCHECK_VERSION="5.1.0"
          wget -q "https://github.com/w3c/epubcheck/releases/download/v${EPUBCHECK_VERSION}/epubcheck-${EPUBCHECK_VERSION}.zip"
          unzip -q "epubcheck-${EPUBCHECK_VERSION}.zip"
          echo "EPUBCHECK_JAR=$(pwd)/epubcheck-${EPUBCHECK_VERSION}/epubcheck.jar" >> $GITHUB_ENV

      - name: Validate EPUB
        run: |
          java -jar "$EPUBCHECK_JAR" "dist/${{ steps.meta.outputs.filename }}"
          echo "✅ EPUBCheck validation passed!"

      # ── Generate a release notes summary ──
      - name: Generate release notes
        id: release_notes
        run: |
          VERSION="${{ steps.meta.outputs.version }}"
          TITLE="${{ steps.meta.outputs.book_title }}"
          
          NOTES="## $TITLE — $VERSION\n\n"
          NOTES+="### What's Included\n"
          NOTES+="- EPUB 3.0 file, validated with EPUBCheck\n"
          NOTES+="- Embedded Bengali font (Kalpurush)\n"
          NOTES+="- Full table of contents\n\n"
          
          # Append CHANGELOG entry for this version if it exists
          if grep -q "$VERSION" CHANGELOG.md 2>/dev/null; then
            NOTES+="### Changelog\n"
            NOTES+=$(sed -n "/^## $VERSION/,/^## /p" CHANGELOG.md | head -n -1)
          fi
          
          echo "notes<<EOF" >> $GITHUB_OUTPUT
          echo -e "$NOTES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      # ── Create the GitHub Release and upload EPUB ──
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.meta.outputs.version }}
          name: "${{ steps.meta.outputs.book_title }} — ${{ steps.meta.outputs.version }}"
          body: ${{ steps.release_notes.outputs.notes }}
          draft: false
          prerelease: ${{ contains(steps.meta.outputs.version, '-') }}
          files: |
            dist/${{ steps.meta.outputs.filename }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # ── Optional: Notify the .github org repo ──
      # This dispatches an event to your org's .github repo for a central log
      - name: Notify central repository
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.ORG_PAT }}  # Personal Access Token with repo scope
          repository: libepub/.github
          event-type: epub-released
          client-payload: |
            {
              "book_id": "${{ steps.meta.outputs.book_id }}",
              "title": "${{ steps.meta.outputs.book_title }}",
              "version": "${{ steps.meta.outputs.version }}",
              "epub_url": "https://github.com/${{ github.repository }}/releases/download/${{ steps.meta.outputs.version }}/${{ steps.meta.outputs.filename }}"
            }
        continue-on-error: true  # Don't fail the release if notification fails
```

### Releasing a New Version

```bash
# Make your changes, commit them
git add .
git commit -m "Complete Chapter 5: Sabr and Gratitude"

# Tag the release (triggers the workflow)
git tag v1.1.0
git push origin main --tags

# GitHub Actions will now:
# 1. Build the EPUB
# 2. Validate it
# 3. Create a release at:
#    https://github.com/libepub/riyad-us-saleheen/releases/tag/v1.1.0
```

### Central Dashboard in `.github` Repository

Create a central `release-tracker.yml` workflow in the `libepub/.github` repository that listens for `epub-released` events and updates a central release log:

```yaml
# libepub/.github/.github/workflows/release-tracker.yml
name: Track EPUB Releases

on:
  repository_dispatch:
    types: [epub-released]

jobs:
  log-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Update release log
        run: |
          TIMESTAMP=$(date -u +"%Y-%m-%d %H:%M:%S UTC")
          ENTRY="| $TIMESTAMP | ${{ github.event.client_payload.title }} | ${{ github.event.client_payload.version }} | [Download](${{ github.event.client_payload.epub_url }}) |"
          
          # Append to RELEASES.md
          echo "$ENTRY" >> RELEASES.md
          
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add RELEASES.md
          git commit -m "📚 New release: ${{ github.event.client_payload.title }} ${{ github.event.client_payload.version }}"
          git push
```

---

## 14. Common Mistakes & How to Fix Them

### ❌ `mimetype` is compressed or in the wrong order

**Symptom:** EPUBCheck error: `PKG-003: Incorrect MIME type`

**Fix:**
```bash
# Wrong way:
zip -r output.epub .  # This compresses everything including mimetype

# Right way:
zip -X0 output.epub mimetype  # First: uncompressed
zip -rX9 output.epub META-INF OEBPS  # Then: everything else
```

---

### ❌ Missing manifest entry

**Symptom:** `OPF-003: Manifest item missing for file: css/styles.css`

**Fix:** Every single file in your OEBPS folder MUST have a `<item>` entry in `content.opf`'s `<manifest>` section. Check for any CSS, image, or font files you may have added but forgotten to register.

---

### ❌ XHTML is not well-formed

**Symptom:** `RSC-005: Content is not well-formed XML`

**Common causes and fixes:**

```xml
<!-- ❌ Unclosed tags -->
<p>Some text<br>
<img src="cover.jpg">

<!-- ✅ Fixed -->
<p>Some text<br/></p>
<img src="cover.jpg" alt="Cover"/>

<!-- ❌ Unescaped ampersand -->
<p>নবী ও সাহাবা</p>

<!-- ✅ Fixed -->
<p>নবী &amp; সাহাবা</p>

<!-- ❌ Missing DOCTYPE -->
<html xmlns="http://www.w3.org/1999/xhtml">

<!-- ✅ Fixed -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
```

---

### ❌ Bengali text shows as boxes or wrong font

**Symptom:** Text renders as □□□□ on Kindle

**Causes:**
1. Font not embedded — add it to manifest and `@font-face`
2. Font doesn't cover Bengali Unicode range — use Kalpurush or SolaimanLipi
3. Wrong `font-family` name in CSS (case-sensitive)

---

### ❌ Arabic text inside Bengali displays backwards

**Symptom:** Arabic letters are correct but order is reversed

**Fix:**
```xml
<!-- Always wrap Arabic in a span with dir="rtl" -->
<span dir="rtl" lang="ar" 
      style="unicode-bidi: isolate; font-family: 'Amiri';">
  بِسْمِ اللَّهِ الرَّحْمَٰنِ الرَّحِيمِ
</span>
```

---

### ❌ Table of contents doesn't work on Kindle

**Symptom:** Kindle's "Table of Contents" button shows nothing

**Causes:**
1. `toc.ncx` not included in manifest
2. `<spine toc="ncx">` not pointing to the NCX item ID
3. NCX `dtb:uid` doesn't match `content.opf` `dc:identifier`

---

### ❌ Images are huge / EPUB file size too large

**Fix:**
```bash
# Compress images with ImageMagick
mogrify -quality 85 -resize "1600x2400>" images/*.jpg

# Or use cwebp for WebP (check reader support first)
cwebp -q 80 cover.jpg -o cover.webp
```

---

## 15. Recommended Tools

### Essential (Free & Open Source)

| Tool | Purpose | Install |
|---|---|---|
| **EPUBCheck** | EPUB validation (official W3C tool) | `java -jar epubcheck.jar` |
| **Calibre** | EPUB viewing, conversion, editing | `sudo apt install calibre` |
| **ACE by DAISY** | Accessibility checker | `npm install -g @daisy/ace` |
| **Sigil** | EPUB visual editor (GUI) | `sudo apt install sigil` |
| **Pandoc** | Convert from Markdown/DOCX to EPUB | `sudo apt install pandoc` |

### Fonts

| Font | Use Case | Source |
|---|---|---|
| Kalpurush | Bengali body text | fonts.google.com |
| SolaimanLipi | Bengali high compatibility | ekushey.org |
| Amiri | Arabic text | amirifont.org |
| Scheherazade New | Arabic with diacritics | software.sil.org |

### Editors

| Editor | Notes |
|---|---|
| **VS Code** | Best for editing XML/XHTML — install XML extension |
| **Sigil** | Visual EPUB editor with built-in EPUBCheck |
| **Obsidian** | Good for drafting in Markdown before converting |

### Testing Devices/Apps

| App/Device | Platform |
|---|---|
| Kindle Previewer 3 | Windows/Mac — tests multiple Kindle models |
| Calibre viewer | Cross-platform |
| Apple Books | Mac/iOS |
| Kobo desktop | Windows/Mac |

---

## 16. Quick Reference Cheatsheet

### Mandatory Files Checklist

```
[ ] mimetype                    — "application/epub+zip", uncompressed, no newline
[ ] META-INF/container.xml      — Points to content.opf
[ ] OEBPS/content.opf           — Metadata, manifest, spine
[ ] OEBPS/nav.xhtml             — EPUB 3 navigation (in manifest with properties="nav")
[ ] OEBPS/toc.ncx               — Legacy NCX (for Kindle compatibility)
[ ] All files listed in manifest
[ ] Spine order correct
[ ] EPUBCheck passes with 0 errors
```

### XHTML File Checklist

```
[ ] xml declaration: <?xml version="1.0" encoding="UTF-8"?>
[ ] <!DOCTYPE html>
[ ] xmlns="http://www.w3.org/1999/xhtml" on <html>
[ ] xmlns:epub="http://www.idpf.org/2007/ops" on <html>
[ ] xml:lang and lang attributes set correctly
[ ] All tags closed
[ ] & escaped as &amp;
[ ] Images have alt attributes
[ ] CSS linked with relative path
```

### content.opf Checklist

```
[ ] dc:identifier with unique UUID
[ ] dc:title
[ ] dc:language (use "bn" for Bengali)
[ ] dcterms:modified (update on every release)
[ ] dc:creator
[ ] Every file in OEBPS listed in manifest
[ ] nav.xhtml has properties="nav" in manifest
[ ] Cover image has properties="cover-image" in manifest
[ ] NCX referenced in <spine toc="ncx">
[ ] UUID in dc:identifier matches dtb:uid in toc.ncx
```

### Build Commands

```bash
# Build
cd src && zip -X0 ../dist/output.epub mimetype && zip -rX9 ../dist/output.epub META-INF OEBPS

# Validate
java -jar epubcheck.jar dist/output.epub

# Convert to MOBI (Calibre)
ebook-convert output.epub output.mobi --output-profile kindle

# View in Calibre
calibre output.epub
```

### CSS Properties to Avoid in EPUB

```css
/* ❌ AVOID these */
position: fixed;
position: absolute;
float: left;
display: grid;
overflow: hidden;
background-image: url(...);
var(--custom-property);
width: 300px;  /* fixed pixels */
font-size: 16px; /* use em instead */
```

### Git Release Workflow

```bash
# 1. Edit your content
# 2. Commit
git add . && git commit -m "feat: add chapter 5"

# 3. Tag (this triggers the GitHub Action)
git tag v1.2.0

# 4. Push tag
git push origin main --tags

# 5. Watch the magic at:
# https://github.com/libepub/BOOK-NAME/actions
```

---

*Built with ♥ by the libepub team — digitizing Bengali Islamic heritage, one book at a time.*
*Writen by Claude Sonnet 4.6.*
*"ইলম অন্বেষণ করা প্রতিটি মুসলিমের উপর ফরজ।" — সহীহ ইবনে মাজাহ*
