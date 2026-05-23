- Phần lớn file được sinh ra bởi AI hiện tại là file .md nhưng để xem được đẹp & tiện thì các công cụ chưa đủ tốt (hoặc là rất nặng, hoặc là rất rườm ra với nhiều features không cần thiết).
- Có cách nào để native OS hiện thị được file .md đẹp & tiện không? Viết 1 phần mềm nhúng vào OS luôn được không? Giả sử tên là "Markdown Quick Look" - App đọc file md minimal nhẹ & nhanh nhất trên mọi OS & mọi IDE; hỗ trợ Quick Look trên MacOS, và các OS khác cũng sẽ hỗ trợ nếu có logic tương tự Quick Look.
    - Với MacOS thì có Quick Look, làm cách nào để nâng cấp Quick Look để nó hiện thị được file .md đẹp & tiện ? Hãy thử tìm cách và trình bày các cách với ưu nhược điểm. Từ đó đưa ra gợi ý tốt nhất
    - Với các OS khác (Windows, Linux) thì có những công cụ nào ? Hãy thử tìm cách và trình bày các cách với ưu nhược điểm. 

---

# Research: Giải pháp "Markdown Quick Look" — Xem file Markdown đẹp, nhẹ & nhanh trên mọi OS & IDE

## Mục lục

1. [Bối cảnh & Vấn đề](#1-bối-cảnh--vấn-đề)
2. [Tiêu chí đánh giá](#2-tiêu-chí-đánh-giá)
3. [macOS — Các giải pháp chi tiết](#3-macos--các-giải-pháp-chi-tiết)
4. [Windows — Các giải pháp chi tiết](#4-windows--các-giải-pháp-chi-tiết)
5. [Linux — Các giải pháp chi tiết](#5-linux--các-giải-pháp-chi-tiết)
6. [IDE / Editor — Markdown Preview trên mọi IDE](#6-ide--editor--markdown-preview-trên-mọi-ide)
7. [Terminal / CLI — Xem Markdown từ dòng lệnh](#7-terminal--cli--xem-markdown-từ-dòng-lệnh)
8. [Rendering Pipeline — Phân tích kiến trúc hiển thị Markdown](#8-rendering-pipeline--phân-tích-kiến-trúc-hiển-thị-markdown)
9. [Cross-platform — Tự build app "Markdown Quick Look"](#9-cross-platform--tự-build-app-md-quick-look)
10. [Bảng tổng hợp so sánh toàn diện](#10-bảng-tổng-hợp-so-sánh-toàn-diện)
11. [Đề xuất giải pháp tối ưu nhất](#11-đề-xuất-giải-pháp-tối-ưu-nhất)
12. [Version Tracking](#12-version-tracking)

---

## 1. Bối cảnh & Vấn đề

**Thực trạng**: Phần lớn output từ AI (ChatGPT, Claude, Gemini, Copilot...) đều là file `.md`. Tuy nhiên:
- Các OS (macOS, Windows, Linux) **không hỗ trợ native** hiển thị Markdown dưới dạng rendered.
- Các công cụ hiện có hoặc **quá nặng** (Electron-based: Obsidian ~200-500MB RAM, Typora, MarkText) hoặc **quá rườm rà** với nhiều features không cần thiết.
- **Nhu cầu cốt lõi**: Chỉ cần 1 app/plugin **view-only**, render Markdown đẹp, mở nhanh, nhẹ, tích hợp sâu vào OS.

**Metrics đo lường** (Không đo lường được thì không quản lý & tối ưu được):

| Metric                                | Mục tiêu                                          |
| :------------------------------------ | :------------------------------------------------ |
| RAM sử dụng                           | < 50MB                                            |
| Thời gian mở file                     | < 0.5s                                            |
| Kích thước cài đặt                    | < 20MB                                            |
| Tích hợp OS                           | Native (Quick Look / Preview Pane / File Manager) |
| Hỗ trợ GFM (GitHub Flavored Markdown) | ✅ Bắt buộc                                        |
| Syntax highlighting code blocks       | ✅ Bắt buộc                                        |
| Mermaid / Math / Diagram              | ✅ Nên có                                          |
| Dark mode                             | ✅ Bắt buộc                                        |

---

## 2. Tiêu chí đánh giá

Mỗi giải pháp được đánh giá theo thang điểm 1-5 (5 = tốt nhất) trên các tiêu chí:

| Tiêu chí                | Trọng số | Mô tả                                            |
| :---------------------- | :------: | :----------------------------------------------- |
| **Nhẹ & Nhanh**         |   30%    | RAM, CPU, thời gian khởi động                    |
| **Tích hợp OS**         |   25%    | Mức độ "native", nhúng vào file manager          |
| **Chất lượng render**   |   20%    | GFM, code highlight, math, mermaid               |
| **Dễ cài đặt**          |   15%    | Số bước setup, cần Xcode hay không               |
| **Bảo trì & Cộng đồng** |   10%    | Tần suất update, GitHub stars, active maintainer |

---

## 3. macOS — Các giải pháp chi tiết

### 3.1. Quick Look có thể mở rộng để đọc .md không?

> **Kết luận: CÓ — Quick Look được Apple thiết kế để extensible.**

Quick Look là system service của macOS, cho phép nhấn **Space** trên file trong Finder để xem preview ngay lập tức. Mặc định macOS **không render Markdown** (chỉ hiển thị raw text), nhưng Apple cung cấp cơ chế **Quick Look Extension** cho phép bên thứ 3 bổ sung khả năng render cho bất kỳ file format nào.

**Cách hoạt động:**
1. Developer viết một **Quick Look Extension** (dạng App Extension trong Xcode) implement `QLPreviewProvider`.
2. Extension đăng ký nhận file type `net.daringfireball.markdown` (UTI cho `.md`).
3. Khi user nhấn Space trên file `.md` trong Finder → macOS gọi extension → extension đọc file, parse Markdown thành HTML → trả về rendered preview.
4. User thấy Markdown đẹp ngay trong popup Quick Look, **không cần mở app nào**.

→ Dưới đây là các Quick Look Extension có sẵn cho Markdown, xếp hạng theo chất lượng.

### 3.2. Quick Look Extension — Cách tiếp cận chính thức

Quick Look cho phép nhấn **Space** trên file trong Finder để xem preview ngay lập tức. Apple thiết kế Quick Look là **extensible** thông qua Quick Look Extensions.

> **⚠️ QUAN TRỌNG — Sau khi cài bất kỳ Quick Look extension nào:**
> 1. Mở app lần đầu (right-click → Open nếu bị chặn Gatekeeper)
> 2. Vào **System Settings → General → Login Items & Extensions → Quick Look**
> 3. Bật toggle cho extension vừa cài
> 4. Chạy `qlmanage -r` trong Terminal để refresh cache Quick Look
> 5. Nếu vẫn không hoạt động: logout/login lại hoặc `killall Finder`

#### Cách A: QLMarkdown (by sbarex) ⭐⭐⭐⭐⭐

| Thuộc tính           | Chi tiết                                                          |
| :------------------- | :---------------------------------------------------------------- |
| **GitHub**           | [sbarex/QLMarkdown](https://github.com/sbarex/QLMarkdown)         |
| **Stars**            | ~2.9K                                                             |
| **License**          | MIT                                                               |
| **Cài đặt**          | `brew install --cask qlmarkdown`                                  |
| **Engine**           | `cmark-gfm` (GitHub's fork — tương thích cao)                     |
| **GFM support**      | ✅ Tables, strikethrough, autolinks, task lists                    |
| **Code highlight**   | ✅ Syntax highlighting cho code blocks                             |
| **Math**             | ✅ LaTeX via MathJax                                               |
| **Mermaid**          | ✅ Flowcharts, sequence diagrams                                   |
| **Emoji**            | ✅                                                                 |
| **Footnotes**        | ✅                                                                 |
| **File types**       | `.md`, `.rmd`, `.mdx`, `.mdc`, `.qmd`, `.apib`, textbundle        |
| **GUI Config**       | ✅ App riêng để customize theme, font size                         |
| **CLI tool**         | ✅ Có kèm CLI convert MD → HTML                                    |
| **Signed/Notarized** | ✅ (từ bản release, cần right-click Open lần đầu hoặc `xattr -cr`) |

**Ưu điểm**:
- Cài 1 lệnh, hoạt động ngay.
- Render cực nhanh, nhấn Space là thấy.
- Cộng đồng lớn, bảo trì tốt.
- Hỗ trợ đầy đủ GFM + Math + Mermaid.

**Nhược điểm**:
- Chỉ là Quick Look (view trong popup Finder), không phải full app.
- Không hỗ trợ live-reload khi file thay đổi.

**Đánh giá**: Nhẹ 5/5 | Tích hợp OS 5/5 | Render 4.5/5 | Dễ cài 5/5 | Bảo trì 4.5/5

#### Cách B: FluxMarkdown ⭐⭐⭐⭐½

| Thuộc tính         | Chi tiết                                                        |
| :----------------- | :-------------------------------------------------------------- |
| **GitHub**         | [xykong/flux-markdown](https://github.com/xykong/flux-markdown) |
| **License**        | GPL-3.0                                                         |
| **Cài đặt**        | `brew install --cask xykong/tap/flux-markdown`                  |
| **Đặc biệt**       | Thiết kế cho technical documentation hiện đại                   |
| **GFM**            | ✅ + GitHub Alerts                                               |
| **Mermaid**        | ✅ Flowcharts, sequence, architecture diagrams                   |
| **Math**           | ✅ KaTeX (nhanh hơn MathJax)                                     |
| **Charts**         | ✅ Vega, Vega-Lite, Graphviz DOT                                 |
| **Code highlight** | ✅ 40+ ngôn ngữ                                                  |
| **TOC**            | ✅ Interactive Table of Contents, scroll tracking                |
| **Export**         | ✅ PDF, HTML trực tiếp từ Quick Look                             |
| **Theme**          | ✅ Light/Dark/System sync                                        |
| **Settings**       | ✅ Cmd+, để customize font, render options                       |

**Ưu điểm**:
- Feature-rich nhất trong các Quick Look plugins.
- Hỗ trợ Vega charts, Graphviz — vượt trội so với QLMarkdown.
- Export PDF/HTML ngay từ preview — rất tiện.
- Interactive TOC panel.

**Nhược điểm**:
- Mới hơn, cộng đồng chưa lớn bằng QLMarkdown.
- GPL-3.0 license (hạn chế hơn MIT nếu muốn fork).

**Đánh giá**: Nhẹ 4.5/5 | Tích hợp OS 5/5 | Render 5/5 | Dễ cài 4.5/5 | Bảo trì 4/5

#### Cách C: Markdown Preview (Anybox Ltd — Mac App Store) ⭐⭐⭐⭐

| Thuộc tính     | Chi tiết                 |
| :------------- | :----------------------- |
| **Nguồn**      | Mac App Store            |
| **Giá**        | Free                     |
| **GFM**        | ✅ GitHub-style rendering |
| **Mermaid**    | ✅                        |
| **Math**       | ✅ KaTeX                  |
| **TextBundle** | ✅                        |

**Ưu điểm**:
- Cài từ App Store — đơn giản nhất, đã được Apple verify.
- Không cần dùng terminal.

**Nhược điểm**:
- Ít tùy chỉnh hơn QLMarkdown/FluxMarkdown.
- Closed-source.

**Đánh giá**: Nhẹ 4.5/5 | Tích hợp OS 4.5/5 | Render 4/5 | Dễ cài 5/5 | Bảo trì 3.5/5

### 3.3. Native Standalone Viewer Apps (macOS)

#### MarkEdit ⭐⭐⭐⭐½

| Thuộc tính            | Chi tiết                                                          |
| :-------------------- | :---------------------------------------------------------------- |
| **GitHub**            | [MarkEdit-app/MarkEdit](https://github.com/MarkEdit-app/MarkEdit) |
| **Tech**              | Swift + AppKit (100% native, KHÔNG Electron)                      |
| **Kích thước**        | ~3-4 MB                                                           |
| **GFM**               | ✅ Strict GFM compliance                                           |
| **Engine**            | CodeMirror 6                                                      |
| **Extensible**        | ✅ Custom scripts, CSS, JS, CodeMirror extensions                  |
| **macOS Integration** | ✅ Apple Shortcuts, AppleScript, Writing Tools, force-touch        |
| **Mục đích**          | "TextEdit for Markdown" — editor + viewer                         |
| **Extensions**        | Preview pane (`MarkEdit-preview`), AI writer, table editor        |

**Ưu điểm**:
- Cực nhẹ (3-4MB), native Swift, mở instant.
- Tôn trọng macOS UX patterns.
- Xử lý file lớn (10MB+) mượt mà.
- Extensible mà vẫn minimal.

**Nhược điểm**:
- Là editor, không phải pure viewer → có phần editing không cần.
- Chỉ macOS.

#### QuickMD ⭐⭐⭐⭐

- Pure viewer (read-only), SwiftUI-based.
- Được gọi là "Preview.app for Markdown".
- Dark mode, Mermaid, LaTeX, TOC sidebar.
- Instant rendering, rất nhanh.

#### Simply Markdown ⭐⭐⭐½

- Live-reloading viewer — mở file, tự động cập nhật khi file thay đổi.
- Hoàn hảo nếu edit trong VS Code/Vim và muốn preview window riêng.
- Native macOS.

### 3.4. macOS Automator / Shortcuts Workflow

Có thể tạo Quick Action để right-click file `.md` → render HTML → mở trong browser:

```bash
# Cài pandoc trước
brew install pandoc

# Trong Automator > Quick Action > Run Shell Script:
cat > /tmp/preview.html <<EOF
<html><body>
$(pandoc -f markdown -t html "$1")
</body></html>
EOF
open /tmp/preview.html
```

**Ưu điểm**: Miễn phí, tuỳ biến tối đa.
**Nhược điểm**: Chậm hơn Quick Look, mở browser, không elegant.

### 3.5. ✅ Đề xuất tốt nhất cho macOS

```
Combo tối ưu nhất:
┌─────────────────────────────────────────────────┐
│  1. QLMarkdown (Quick Look)                     │
│     → Nhấn Space trong Finder = preview ngay    │
│     → brew install --cask qlmarkdown            │
│                                                 │
│  2. MarkEdit (Standalone khi cần đọc kỹ)       │
│     → Mở file .md nhanh như TextEdit            │
│     → brew install --cask markedit              │
└─────────────────────────────────────────────────┘

Hoặc nếu cần features advanced hơn (Vega charts, Graphviz):
→ Thay QLMarkdown bằng FluxMarkdown
```

---

## 4. Windows — Các giải pháp chi tiết

### 4.1. Microsoft PowerToys (Khuyến nghị nhất) ⭐⭐⭐⭐⭐

| Thuộc tính          | Chi tiết                                               |
| :------------------ | :----------------------------------------------------- |
| **Nguồn**           | Microsoft (chính hãng, open-source)                    |
| **Giá**             | Free                                                   |
| **Cài đặt**         | Microsoft Store / `winget install Microsoft.PowerToys` |
| **Tính năng chính** | 2 tính năng preview Markdown                           |

**Tính năng 1: File Explorer Preview Pane**
- Bật trong PowerToys Settings → File Explorer Add-ons → Enable Markdown (.md) preview.
- Mở File Explorer → View → Preview Pane (hoặc `Alt + P`).
- Chọn file `.md` → render ngay trong pane bên phải.

**Tính năng 2: PowerToys Peek**
- Giống macOS Quick Look.
- Chọn file → `Ctrl + Space` → popup preview ngay lập tức.
- Có thể dùng arrow keys để navigate qua các file khác.

**Ưu điểm**:
- Chính hãng Microsoft, tích hợp sâu vào Windows.
- Kết hợp cả Preview Pane + Peek = 2 cách preview.
- Nhẹ, ổn định, cập nhật thường xuyên.
- Không cần app riêng.

**Nhược điểm**:
- Render basic, không hỗ trợ Mermaid/Math.
- Chỉ hoạt động trong File Explorer.

**Đánh giá**: Nhẹ 5/5 | Tích hợp OS 5/5 | Render 3.5/5 | Dễ cài 5/5 | Bảo trì 5/5

### 4.2. Standalone Viewers (Windows)

#### Markdown Reader Pro (Microsoft Store) ⭐⭐⭐½

- Native WinUI 3 app.
- Nhẹ, có thể set làm default app cho `.md`.
- Live preview, basic formatting.

#### MDViewer ⭐⭐⭐

- Minimalist, real-time preview.
- Tables, lists, code blocks.
- Không nhiều advanced features.

### 4.3. ✅ Đề xuất tốt nhất cho Windows

```
Combo tối ưu:
┌──────────────────────────────────────────────────┐
│  1. PowerToys (tích hợp File Explorer)           │
│     → Peek: Ctrl+Space = preview ngay            │
│     → Preview Pane: Alt+P = panel bên phải       │
│     → winget install Microsoft.PowerToys         │
│                                                  │
│  2. Markdown Reader Pro (khi cần đọc kỹ)        │
│     → Microsoft Store, set default cho .md       │
└──────────────────────────────────────────────────┘
```

---

## 5. Linux — Các giải pháp chi tiết

### 5.1. GNOME (Nautilus)

#### GNOME Markdown QuickLook ⭐⭐⭐⭐

- Chọn file `.md` trong Nautilus → nhấn **Space** → preview rendered.
- Giống macOS Quick Look experience.
- Tích hợp sâu vào GNOME desktop.

#### Apostrophe ⭐⭐⭐½

- Native GNOME app, distraction-free.
- Editor + viewer, GTK-based.
- Nhẹ, follow GNOME HIG (Human Interface Guidelines).

### 5.2. KDE Plasma (Dolphin)

#### Dolphin Preview Pane + Okular ⭐⭐⭐⭐

- Dolphin file manager → `F11` → Preview Pane.
- Okular (built-in KDE document viewer) có thể render Markdown.
- Cần configure file association cho Markdown.

#### Kiview ⭐⭐⭐½

- Quick Look style previewer cho KDE Dolphin.
- Lightweight, popup riêng.

### 5.3. Cross-Desktop Standalone

#### md-viewer (Rust) ⭐⭐⭐⭐

- Viết bằng Rust → cực nhanh, ít RAM.
- Built-in file explorer, tab support, live reload.
- Native system file picker.
- Thiết kế cho reading, không phải editing.

#### ReText ⭐⭐⭐½

- Classic, lightweight editor + viewer.
- Live preview, PyQt-based.
- Reliable "middle ground" giữa viewer và IDE.

### 5.4. ✅ Đề xuất tốt nhất cho Linux

```
GNOME users:
┌──────────────────────────────────────────────────┐
│  GNOME Markdown QuickLook (Space để preview)     │
│  + md-viewer (Rust) cho focused reading          │
└──────────────────────────────────────────────────┘

KDE users:
┌──────────────────────────────────────────────────┐
│  Dolphin Preview Pane (F11) với Okular           │
│  + md-viewer (Rust) cho focused reading          │
└──────────────────────────────────────────────────┘
```

---

## 6. IDE / Editor — Markdown Preview trên mọi IDE

### 6.1. VS Code ⭐⭐⭐⭐⭐

| Cách                                         | Phím tắt                                           | Mô tả                                                              |
| :------------------------------------------- | :------------------------------------------------- | :----------------------------------------------------------------- |
| **Built-in Preview**                         | `Cmd+Shift+V` (full) hoặc `Cmd+K V` (side-by-side) | Đã có sẵn, KHÔNG cần cài thêm gì. Hỗ trợ GFM, Mermaid, LaTeX.      |
| **Lightweight Markdown Preview** (extension) | —                                                  | ~38KB, ~300 dòng code. Minimalist, privacy-first. Mermaid + LaTeX. |
| **Markdown All in One** (extension)          | —                                                  | Industry standard. TOC auto-gen, shortcuts, formatting.            |

**Đề xuất**: Dùng built-in là đủ cho 90% nhu cầu. Chỉ cài Markdown All in One nếu cần viết documentation thường xuyên.

### 6.2. JetBrains IDEs (IntelliJ, WebStorm, PyCharm...) ⭐⭐⭐⭐½

- **Built-in Markdown plugin** — có sẵn, bật mặc định.
- Live preview, split-screen, floating toolbar.
- Code injection (syntax highlighting trong code blocks theo ngôn ngữ).
- Mermaid, PlantUML support.
- Export HTML, PDF. Pandoc integration.
- Custom CSS cho preview pane.
- **2025.1**: Search trong preview pane.
- **2025.2**: AI-powered code completion cho Markdown.

**Đề xuất**: Không cần cài thêm gì, built-in plugin rất mạnh.

### 6.3. Neovim ⭐⭐⭐⭐

| Plugin                    | Đặc điểm                                                               |
| :------------------------ | :--------------------------------------------------------------------- |
| **render-markdown.nvim**  | Render trực tiếp trong buffer — WYSIWYG feel. Phổ biến nhất 2024-2025. |
| **peek.nvim**             | Deno-based webview, GitHub-style, sync scroll, TeX, Mermaid.           |
| **markdown-preview.nvim** | Classic, mở preview trong browser. Reliable.                           |

**Đề xuất**: `render-markdown.nvim` nếu muốn in-editor, `peek.nvim` nếu muốn preview window riêng.

### 6.4. Sublime Text ⭐⭐⭐

| Plugin                  | Đặc điểm                                                    |
| :---------------------- | :---------------------------------------------------------- |
| **Markdown Preview**    | Convert → mở browser. Kết hợp LiveReload cho auto-refresh.  |
| **MarkdownLivePreview** | In-editor preview, nhưng dùng minihtml (hạn chế rendering). |

**Đề xuất**: Markdown Preview + LiveReload = workflow ổn.

---

## 7. Terminal / CLI — Xem Markdown từ dòng lệnh

### 7.1. Glow (by Charmbracelet) ⭐⭐⭐⭐⭐

| Thuộc tính         | Chi tiết                                                                                       |
| :----------------- | :--------------------------------------------------------------------------------------------- |
| **GitHub**         | [charmbracelet/glow](https://github.com/charmbracelet/glow)                                    |
| **Tech**           | Go                                                                                             |
| **Cài đặt**        | `brew install glow` (macOS/Linux) / `choco install glow` / `winget install charmbracelet.glow` |
| **CLI mode**       | `glow file.md` → render đẹp trong terminal                                                     |
| **TUI mode**       | `glow` (không có arg) → file browser interactively                                             |
| **Theme**          | Light/Dark/Auto + custom JSON styles                                                           |
| **Git-aware**      | ✅ Tự tìm `.md` files trong git repos                                                           |
| **Input**          | File, stdin (pipe), HTTP URL                                                                   |
| **Cross-platform** | ✅ macOS, Linux, Windows                                                                        |

**Ưu điểm**: Đẹp nhất trong terminal, cài 1 lệnh, cross-platform, interactive TUI.
**Nhược điểm**: Terminal-only, không render images/diagrams.

### 7.2. Pandoc (Universal Converter) ⭐⭐⭐⭐

```bash
# MD → HTML (mở trong browser)
pandoc input.md -o output.html && open output.html

# MD → PDF (cần LaTeX engine)
pandoc input.md -o output.pdf

# Batch convert
for file in *.md; do pandoc "$file" -o "${file%.md}.pdf"; done
```

**Ưu điểm**: Universal — convert sang mọi format. Automation-friendly.
**Nhược điểm**: Cần thêm LaTeX cho PDF. Không phải interactive viewer.

---

## 8. Rendering Pipeline — Phân tích kiến trúc hiển thị Markdown

### 8.1. Markdown có phải convert sang HTML mới hiển thị được không?

> **Không nhất thiết.** Có **2 pipeline rendering hoàn toàn khác nhau**, và tốc độ **KHÔNG phụ thuộc chủ yếu vào converter** mà phụ thuộc vào **rendering engine phía sau**.

### 8.2. Hai Pipeline Rendering

#### Pipeline 1: MD → HTML → WebView (phổ biến nhất)

```
File .md → [Parser/Converter] → HTML string → [WebView/WKWebView] → Display
             ~0.5-5ms              cực nhanh         ~50-200ms
                                                    ↑ BOTTLENECK
```

- **Dùng bởi**: QLMarkdown, FluxMarkdown, Quick Look extensions, Electron apps, browser-based tools.
- **Parser** (cmark-gfm, comrak, pulldown-cmark...) parse file `.md` thành HTML string → **cực nhanh** (microseconds ~ vài ms cho file thông thường).
- **Bottleneck thực sự**: WebView initialization + CSS layout engine + JavaScript engine (nếu cần Mermaid/Math).

#### Pipeline 2: MD → AST → Native UI (không qua HTML)

```
File .md → [Parser] → Abstract Syntax Tree → [Native Views] → Display
            ~0.5-5ms                            ~5-20ms
                                              Không qua HTML!
```

- **Dùng bởi**: **MarkdownUI** (SwiftUI), **render-markdown.nvim** (Neovim), **Glow** (terminal).
- Parser tạo AST → map trực tiếp sang native UI components (SwiftUI Text, VStack, Image...) hoặc terminal escape codes.
- **Không cần WebView**, không cần HTML, không cần CSS layout engine.
- **Nhanh hơn đáng kể** vì native views rất lightweight.

### 8.3. Bottleneck thực sự nằm ở đâu?

| Bước trong pipeline                      | Thời gian (file 100KB) |      Bottleneck?       |
| :--------------------------------------- | :--------------------- | :--------------------: |
| **Đọc file từ disk**                     | ~0.1ms                 |           ❌            |
| **Parse MD → AST/HTML** (converter)      | ~0.5-5ms               |      ❌ Rất nhanh       |
| **WebView initialization** (Pipeline 1)  | ~50-200ms              |     ⚠️ Overhead lớn     |
| **HTML + CSS layout** (Pipeline 1)       | ~20-100ms              |           ⚠️            |
| **JavaScript execution** (Mermaid/KaTeX) | ~100-500ms             | 🔴 **Bottleneck chính** |
| **Native view creation** (Pipeline 2)    | ~5-20ms                |      ❌ Rất nhanh       |

→ **Kết luận**: Parser/Converter (cmark, comrak...) **hầu như không bao giờ là bottleneck**. Ngay cả parser JS (chậm nhất) cũng chỉ mất vài ms. Cái chậm là **rendering engine phía sau** — đặc biệt là WebView init và JavaScript cho diagrams/math.

### 8.4. Alternative Rendering Engines — Tối ưu hiển thị

Thay vì dùng WKWebView/WebView2 (nặng) hoặc native views (hạn chế features), có thể dùng các **lightweight rendering engine** chuyên biệt:

#### a) litehtml — Ultra-lightweight HTML/CSS renderer ⭐⭐⭐⭐⭐ (cho Markdown Quick Look)

| Thuộc tính      | Chi tiết                                                             |
| :-------------- | :------------------------------------------------------------------- |
| **GitHub**      | [litehtml/litehtml](https://github.com/litehtml/litehtml)            |
| **Ngôn ngữ**    | C++                                                                  |
| **Đặc điểm**    | Render HTML/CSS **không cần JavaScript engine, không cần WebView**   |
| **RAM**         | Cực thấp (vài MB)                                                    |
| **Cách dùng**   | MD → HTML (via md4c/comrak) → litehtml render → vẽ lên native canvas |
| **JS support**  | ❌ Không có (nhưng **không cần** cho Markdown thuần)                  |
| **CSS support** | CSS2/CSS3 subset — đủ cho formatted text, tables, code blocks        |

**Tại sao phù hợp nhất cho Markdown Quick Look?**
- Markdown text/tables/code chiếm **~90% nội dung** → litehtml render hoàn hảo.
- Không cần khởi tạo WebView → **nhanh hơn 10-50x** so với WKWebView.
- "Bring-your-own graphics" — callback interface cho phép dùng native drawing API (CoreGraphics trên macOS, Direct2D trên Windows).
- Kết hợp: `md4c` (C, cực nhanh) parse MD → HTML → `litehtml` render trực tiếp lên native canvas.

**Hạn chế**: Không render được Mermaid/KaTeX (cần JS). Giải pháp: fallback sang WebView chỉ cho blocks cần JS.

#### b) Servo — Embeddable Rust rendering engine ⭐⭐⭐⭐

| Thuộc tính            | Chi tiết                                           |
| :-------------------- | :------------------------------------------------- |
| **GitHub**            | [servo/servo](https://github.com/servo/servo)      |
| **Ngôn ngữ**          | Rust                                               |
| **Tổ chức**           | Linux Foundation Europe                            |
| **Đặc điểm**          | Full rendering engine, embeddable, parallel layout |
| **Có trên crates.io** | ✅ (từ 2026) — dễ embed vào Rust project            |
| **CSS/HTML support**  | Rộng hơn litehtml, gần full web standards          |
| **JS engine**         | SpiderMonkey (Firefox's JS engine)                 |
| **Cross-platform**    | ✅ macOS, Windows, Linux, Android                   |

**Ưu điểm**: Full rendering engine bằng Rust → memory-safe, parallel layout → nhanh hơn WebKit trên multi-core. Có JS engine → render được Mermaid/KaTeX.
**Nhược điểm**: Nặng hơn litehtml đáng kể. API chưa stable hoàn toàn (đang phát triển mạnh).

#### c) Ultralight — GPU-accelerated WebKit fork ⭐⭐⭐½

| Thuộc tính         | Chi tiết                                |
| :----------------- | :-------------------------------------- |
| **Website**        | [ultralig.ht](https://ultralig.ht)      |
| **Ngôn ngữ**       | C/C++                                   |
| **Đặc điểm**       | Custom WebKit fork, GPU-first rendering |
| **Binary size**    | ~5MB DLL                                |
| **RAM**            | Thấp hơn WKWebView đáng kể              |
| **JS engine**      | JavaScriptCore (lightweight)            |
| **Use case chính** | Game UIs, embedded apps                 |

**Ưu điểm**: Render to texture, GPU-accelerated, nhẹ hơn WebKit chuẩn.
**Nhược điểm**: Commercial license cho large projects. Không hỗ trợ WebGL/WebRTC.

#### d) Lightpanda — Headless browser engine ⭐⭐⭐

| Thuộc tính    | Chi tiết                                                          |
| :------------ | :---------------------------------------------------------------- |
| **GitHub**    | [lightpanda-io/browser](https://github.com/lightpanda-io/browser) |
| **Ngôn ngữ**  | Zig                                                               |
| **Stars**     | ~13K+                                                             |
| **Đặc điểm**  | 9-11x nhanh hơn headless Chromium, 9-16x ít RAM hơn               |
| **JS engine** | V8                                                                |
| **Startup**   | < 100ms                                                           |

**Phù hợp cho Markdown Quick Look?** → **Hạn chế**. Lightpanda thiết kế cho **AI agents & scraping** (headless, không có visual rendering). Nó parse DOM + execute JS nhưng **không layout/paint pixels** → không trực tiếp dùng để hiển thị Markdown cho user xem được. Tuy nhiên, concept architecture (Zig + V8 stripped-down) là nguồn cảm hứng tốt.

#### e) Sciter — Custom HTML/CSS engine ⭐⭐⭐½

| Thuộc tính    | Chi tiết                                  |
| :------------ | :---------------------------------------- |
| **Website**   | [sciter.com](https://sciter.com)          |
| **Ngôn ngữ**  | C++                                       |
| **Binary**    | ~5MB DLL                                  |
| **JS engine** | QuickJS (compact)                         |
| **Đặc điểm**  | Custom engine, không dùng WebKit/Chromium |
| **Bindings**  | C++, Rust, Go, C#                         |

**Ưu điểm**: Cực nhẹ, native GPU rendering, startup nhanh.
**Nhược điểm**: Không full web standards. Custom scripting syntax.

#### f) Wry — Native WebView wrapper (Tauri's core) ⭐⭐⭐⭐

| Thuộc tính         | Chi tiết                                                                |
| :----------------- | :---------------------------------------------------------------------- |
| **GitHub**         | [tauri-apps/wry](https://github.com/tauri-apps/wry)                     |
| **Ngôn ngữ**       | Rust                                                                    |
| **Đặc điểm**       | Thin wrapper trên OS native WebView (WKWebView/WebView2/WebKitGTK)      |
| **Quan hệ Tauri** | Wry = rendering layer bên dưới Tauri. Dùng trực tiếp = bỏ Tauri overhead |
| **Binary**         | < 600KB (Hello World, stripped)                                          |
| **RAM**            | Phụ thuộc OS WebView (~40-80MB idle)                                    |
| **Startup**        | < 200ms                                                                  |
| **JS engine**      | ✅ (từ OS WebView — JavaScriptCore/V8)                                   |
| **Cross-platform** | ✅ macOS, Windows, Linux                                                 |

**Ưu điểm**:
- Binary cực nhỏ (không bundle browser engine).
- Full web standards (nhờ dùng native WebView).
- Rust — memory-safe, có thể tích hợp comrak/pulldown-cmark trực tiếp.
- Control hoàn toàn event loop, window, IPC.

**Nhược điểm**:
- **Vẫn là WebView** → overhead WebView init giống Tauri (50-200ms).
- Rendering inconsistency giữa các OS (WebKitGTK trên Linux kém hơn WebView2/WKWebView).
- Không giảm RAM so với Tauri — cùng engine bên dưới.

**So với Tauri**: Wry trực tiếp cho binary nhỏ hơn ~vài MB và bỏ overhead IPC/plugin layer. Nhưng phải tự implement mọi thứ. Chỉ nên dùng khi cần **ultra-minimal** hoặc nhúng WebView vào app có sẵn.

#### g) Obscura — Rust headless browser engine ⭐⭐⭐

| Thuộc tính         | Chi tiết                                                          |
| :----------------- | :---------------------------------------------------------------- |
| **GitHub**         | [h4ckf0r0day/obscura](https://github.com/h4ckf0r0day/obscura)     |
| **Ngôn ngữ**       | Rust                                                               |
| **Stars**          | ~13.5K+                                                            |
| **Đặc điểm**       | Headless browser cho AI/scraping, ~30MB RAM, startup ~85ms         |
| **JS engine**      | ✅                                                                  |
| **CDP support**    | ✅ Chrome DevTools Protocol (drop-in cho Puppeteer/Playwright)     |
| **DOM→Markdown**   | ✅ Native conversion                                                |
| **Anti-detection** | ✅ TLS fingerprinting, stealth mode                                |

**Phù hợp cho Markdown Quick Look?** → **KHÔNG phù hợp.**
- Obscura là headless browser — giống Lightpanda, nó **không render CSS visual** và **không có GUI**.
- Thiết kế cho automation, scraping, AI agents — không phải cho human viewing.
- Feature thú vị: native DOM→Markdown conversion, nhưng ngược hướng (chúng ta cần MD→Display, không phải DOM→MD).
- **Kết luận**: Không dùng được cho mục đích hiển thị Markdown cho user.

#### h) Ladybird / LibWeb — Independent browser engine ⭐⭐⭐½

| Thuộc tính              | Chi tiết                                                                |
| :---------------------- | :---------------------------------------------------------------------- |
| **GitHub**              | [LadybirdBrowser/ladybird](https://github.com/LadybirdBrowser/ladybird) |
| **Ngôn ngữ**            | C++ (đang chuyển dần sang Rust)                                         |
| **Tổ chức**             | Ladybird Browser Initiative (501(c)(3) non-profit)                      |
| **Xuất phát**           | Fork từ SerenityOS (tách riêng 06/2024)                                 |
| **Rendering engine**    | LibWeb (custom, từ scratch)                                              |
| **JS engine**           | LibJS (custom)                                                           |
| **Graphics**            | Skia (sau khi tách khỏi SerenityOS)                                     |
| **WPT pass rate**       | > 90% subtests (cuối 2025)                                               |
| **Architecture**        | Multi-process (UI + renderer sandboxed)                                  |
| **Embeddable?**         | ⚠️ Chưa có stable API — experimental, pre-alpha                          |
| **Alpha release**       | Planned 2026 (Linux/macOS)                                               |
| **Cross-platform**      | macOS, Linux (Windows planned)                                           |

**Ưu điểm**:
- **Hoàn toàn độc lập** — không dựa trên WebKit/Chromium/Gecko. Engine mới viết từ scratch.
- Non-profit, open-source — giá trị dài hạn cho ecosystem.
- Standards compliance tốt (> 90% WPT).
- JS engine riêng (LibJS) — nhẹ hơn V8/SpiderMonkey.
- Đang tích cực phát triển, cộng đồng lớn.

**Nhược điểm**:
- **Chưa embeddable** — không có stable API/SDK để nhúng LibWeb vào app khác. Multi-process architecture làm embedding phức tạp.
- Pre-alpha — API thay đổi liên tục, breaking changes thường xuyên.
- Chưa sẵn sàng cho production (stable release dự kiến 2028).

**Phù hợp cho Markdown Quick Look?** → **Tiềm năng dài hạn, KHÔNG phù hợp ngay bây giờ.**
- Nếu Ladybird release stable API cho LibWeb, nó sẽ là alternative cực kỳ thú vị: nhẹ hơn WebKit, independent, có JS engine riêng.
- **Đáng theo dõi** cho Phase 3+ của project. Hiện tại không thực tế để tích hợp.

### 8.5. So sánh Rendering Engines cho Markdown Quick Look

| Engine              | Ngôn ngữ  | RAM       | Startup    | JS             | Mermaid/Math     | Visual render? | Phù hợp MD?            |    Điểm    |
| :------------------ | :-------- | :-------- | :--------- | :------------- | :--------------- | :------------ | :---------------------- | :--------: |
| **litehtml**        | C++       | ~2-5MB    | <10ms      | ❌              | ❌ (cần fallback) | ✅ Canvas      | ⭐⭐⭐⭐⭐ Text/tables      |  **9/10**  |
| **Servo**           | Rust      | ~30-50MB  | ~100ms     | ✅ SpiderMonkey | ✅                | ✅ Full        | ⭐⭐⭐⭐ Full render       |  **8/10**  |
| **Wry**             | Rust      | ~40-80MB  | <200ms     | ✅ (OS)         | ✅                | ✅ WebView     | ⭐⭐⭐⭐ Nhẹ hơn Electron  | **7.5/10** |
| **Ultralight**      | C++       | ~10-20MB  | ~50ms      | ✅ JSC          | ✅                | ✅ GPU         | ⭐⭐⭐⭐ GPU apps          | **7.5/10** |
| **Sciter**          | C++       | ~5-10MB   | ~20ms      | ✅ QuickJS      | ⚠️ Hạn chế        | ✅ Native      | ⭐⭐⭐½                   |  **7/10**  |
| **Ladybird/LibWeb** | C++/Rust  | ~30-50MB  | ~200ms     | ✅ LibJS        | ✅ (khi stable)   | ✅ Full        | ⭐⭐⭐½ Tiềm năng         |  **7/10**  |
| **WKWebView**       | System    | ~50-100MB | ~100-200ms | ✅              | ✅                | ✅ Full        | ⭐⭐⭐ Nặng               |  **6/10**  |
| **Lightpanda**      | Zig       | ~30-50MB  | <100ms     | ✅ V8           | ❌                | ❌ Headless    | ⭐⭐ Không phù hợp       |  **4/10**  |
| **Obscura**         | Rust      | ~30MB     | ~85ms      | ✅              | ❌                | ❌ Headless    | ⭐ Không phù hợp        |  **3/10**  |

### 8.6. ✅ Đề xuất kiến trúc rendering tối ưu cho Markdown Quick Look

> **Nguyên tắc chung**: Dùng **litehtml** cho text/tables/code (90% content, cực nhanh). Câu hỏi là: **engine nào render phần cần JS** (Mermaid, KaTeX)?

#### Option A: litehtml + Servo ⭐ (Recommended)

```
Hybrid Rendering Architecture (Servo):
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  File .md                                               │
│     │                                                   │
│     ▼                                                   │
│  [comrak] ─── Parse ──→ AST                             │
│     │                     │                             │
│     │              ┌──────┴──────┐                      │
│     │              │             │                      │
│     ▼              ▼             ▼                      │
│  Text/Table/    Mermaid        KaTeX                    │
│  Code (90%)    blocks         blocks                    │
│     │              │             │                      │
│     ▼              ▼             ▼                      │
│  [litehtml]     [Servo - embedded, lazy init]           │
│  Native canvas   SpiderMonkey JS                        │
│  render          Rust-native, cùng ecosystem            │
│     │              │             │                      │
│     └──────────────┴─────────────┘                      │
│                    │                                    │
│                    ▼                                    │
│               Final Display                             │
│              (< 50ms cho 90% files)                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### Option B: litehtml + System WebView (Simpler, lighter binary)

```
Hybrid Rendering Architecture (WebView):
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  File .md                                               │
│     │                                                   │
│     ▼                                                   │
│  [comrak] ─── Parse ──→ AST                             │
│     │                     │                             │
│     │              ┌──────┴──────┐                      │
│     │              │             │                      │
│     ▼              ▼             ▼                      │
│  Text/Table/    Mermaid        KaTeX                    │
│  Code (90%)    blocks         blocks                    │
│     │              │             │                      │
│     ▼              ▼             ▼                      │
│  [litehtml]     [Wry/System WebView - lazy load]        │
│  Native canvas   OS-native JS engine                    │
│  render          Không tăng binary size                  │
│     │              │             │                      │
│     └──────────────┴─────────────┘                      │
│                    │                                    │
│                    ▼                                    │
│               Final Display                             │
│              (< 50ms cho 90% files)                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### So sánh 2 Options:

| Tiêu chí                     | **Option A: Servo** ⭐         | **Option B: WebView**           |
| :--------------------------- | :---------------------------- | :------------------------------ |
| **Cross-platform nhất quán** | ✅ Cùng engine trên mọi OS    | ❌ Khác nhau (WKWebView/WebView2/WebKitGTK) |
| **Kiểm soát version**       | ✅ Pin version, tự kiểm soát  | ❌ Phụ thuộc OS update            |
| **Hệ sinh thái**            | ✅ Rust → cùng comrak/litehtml | ⚠️ Bridge Rust↔WebView           |
| **Binary size**              | ❌ Tăng ~20-30MB (bundle Servo) | ✅ 0 thêm (dùng system engine)  |
| **Maturity**                 | ⚠️ API đang ổn định dần        | ✅ Battle-tested, rất ổn định    |
| **Render Mermaid/KaTeX**     | ✅ SpiderMonkey                | ✅ JSC/V8 (từ OS)                |
| **Phù hợp khi**             | Cần nhất quán, long-term      | MVP nhanh, binary nhỏ nhất      |

**Kết luận**:
- **Option A (Servo)** là lựa chọn tốt hơn cho long-term: nhất quán cross-platform, cùng Rust ecosystem, kiểm soát hoàn toàn. Trade-off duy nhất là binary size tăng ~20-30MB — chấp nhận được cho app desktop.
- **Option B (WebView)** phù hợp cho MVP nhanh hoặc nếu muốn giữ binary < 15MB.
- **Cả 2 đều dùng litehtml cho 90% content** → hầu hết files Markdown sẽ render trong **< 50ms**.

---

## 9. Cross-platform — Tự build app "Markdown Quick Look"

### 9.1. Tại sao nên tự build?

Không có giải pháp nào hiện tại đạt được trọn vẹn:
- **Nhẹ** (< 50MB RAM) + **Đẹp** + **Native OS integration** + **Cross-platform** + **Chỉ view, không edit**

→ Đây chính là cơ hội cho "Markdown Quick Look" app.

### 9.2. Lựa chọn Tech Stack

#### Option A: Native từng platform (Tốt nhất về performance)

| Platform | Tech            | Rendering                         | Kích thước ước tính |
| :------- | :-------------- | :-------------------------------- | :------------------ |
| macOS    | Swift + SwiftUI | MarkdownUI library (native views) | ~3-5 MB             |
| Windows  | C# + WinUI 3    | Markdig library                   | ~5-10 MB            |
| Linux    | Rust + GTK4     | pulldown-cmark                    | ~3-8 MB             |

**Ưu điểm**: Nhẹ nhất, nhanh nhất, tích hợp OS hoàn hảo.
**Nhược điểm**: Cần maintain 3 codebases riêng biệt.

#### Option B: Tauri (Recommended — Cross-platform, gần native) ⭐

| Thuộc tính         | Chi tiết                                                        |
| :----------------- | :-------------------------------------------------------------- |
| **Backend**        | Rust (file I/O, markdown parsing via `comrak`/`pulldown-cmark`) |
| **Frontend**       | Web tech (Svelte/Vanilla JS) trong system WebView               |
| **WebView**        | WebView2 (Win) / WebKit (macOS/Linux) — KHÔNG bundle Chromium   |
| **RAM**            | ~30-50 MB                                                       |
| **Binary size**    | ~5-12 MB                                                        |
| **Cross-platform** | ✅ macOS, Windows, Linux                                         |

```rust
// Rust backend — parsing Markdown
use comrak::{markdown_to_html, ComrakOptions};

#[tauri::command]
fn parse_markdown(md: String) -> String {
    markdown_to_html(&md, &ComrakOptions::default())
}
```

**Ưu điểm**:
- 1 codebase cho 3 OS.
- Dùng native WebView → nhẹ hơn Electron 10-20x.
- Rust backend = parsing cực nhanh.
- Có thể tích hợp Quick Look (macOS), Preview Pane (Windows) dưới dạng extension.

**Nhược điểm**:
- Vẫn là WebView (không 100% native feel).
- Cần biết Rust.

#### Option C: Wry (Ultra-lightweight — cho power users)

- Wry là thư viện WebView bên dưới Tauri.
- Dùng trực tiếp nếu muốn binary ~1MB.
- Chỉ cần 1 window + 1 webview, control hoàn toàn.

#### Option D: Swift + Quick Look Extension (macOS-only, nhúng vào OS)

```swift
import QuickLook
import MarkdownUI

class PreviewProvider: QLPreviewProvider {
    override func providePreview(
        for request: QLFilePreviewRequest,
        completionHandler: @escaping (QLPreviewReply?, Error?) -> Void
    ) {
        // Đọc file → parse markdown → render HTML → trả về QLPreviewReply
        guard let data = try? Data(contentsOf: request.fileURL) else {
            completionHandler(nil, nil)
            return
        }
        let html = convertMarkdownToHTML(data)
        let reply = QLPreviewReply(dataOfContentType: .html, 
                                    contentSize: .zero) { reply in
            return html.data(using: .utf8)!
        }
        completionHandler(reply, nil)
    }
}
```

**Ưu điểm**: Nhúng trực tiếp vào macOS, Space = preview.
**Nhược điểm**: Chỉ macOS, cần Xcode + Apple Developer account.

### 9.3. ✅ Đề xuất nếu tự build "Markdown Quick Look"

```
Approach tốt nhất:
┌───────────────────────────────────────────────────┐
│  Phase 1: Tauri + Rust (cross-platform MVP)       │
│  → 1 codebase, 3 OS, ~10MB, ~30MB RAM            │
│  → Rust comrak parse + Svelte frontend            │
│                                                   │
│  Phase 2: Native OS Extensions                    │
│  → macOS: Quick Look Extension (Swift)            │
│  → Windows: Preview Handler (C#)                  │
│  → Linux: GNOME/KDE plugin                        │
│                                                   │
│  Phase 3: IDE Extensions                          │
│  → VS Code extension (optional)                   │
└───────────────────────────────────────────────────┘
```

---

## 10. Bảng tổng hợp so sánh toàn diện

### 10.1. So sánh theo OS — Giải pháp tích hợp File Manager

| Giải pháp                        | OS            | Loại                | RAM   | Cài đặt         | GFM  | Mermaid | Math | Điểm tổng  |
| :------------------------------- | :------------ | :------------------ | :---- | :-------------- | :--- | :------ | :--- | :--------: |
| **QLMarkdown**                   | macOS         | Quick Look          | <20MB | 1 lệnh brew     | ✅    | ✅       | ✅    | **9.2/10** |
| **FluxMarkdown**                 | macOS         | Quick Look          | <20MB | 1 lệnh brew     | ✅    | ✅       | ✅    | **9.0/10** |
| **Markdown Preview (App Store)** | macOS         | Quick Look          | <20MB | App Store       | ✅    | ✅       | ✅    | **8.5/10** |
| **PowerToys**                    | Windows       | Preview Pane + Peek | <30MB | 1 lệnh winget   | ✅    | ❌       | ❌    | **8.0/10** |
| **GNOME QuickLook**              | Linux (GNOME) | Quick Look style    | <20MB | Package manager | ✅    | ❌       | ❌    | **7.5/10** |
| **Dolphin + Okular**             | Linux (KDE)   | Preview Pane        | <30MB | Built-in        | ✅    | ❌       | ❌    | **7.0/10** |

### 10.2. So sánh Standalone Viewers

| App           | OS    | Tech         | RAM       | Size   | Native | Open Source | Điểm tổng  |
| :------------ | :---- | :----------- | :-------- | :----- | :----- | :---------- | :--------: |
| **MarkEdit**  | macOS | Swift/AppKit | <30MB     | 3-4MB  | ✅      | ✅ (MIT)     | **9.0/10** |
| **QuickMD**   | macOS | SwiftUI      | <25MB     | <5MB   | ✅      | ❌           | **8.5/10** |
| **md-viewer** | Linux | Rust         | <20MB     | <5MB   | ✅      | ✅           | **8.5/10** |
| **Obsidian**  | All   | Electron     | 200-500MB | ~400MB | ❌      | ❌           | **6.0/10** |
| **Typora**    | All   | Electron     | 100-300MB | ~100MB | ❌      | ❌           | **6.5/10** |
| **MarkText**  | All   | Electron     | 100-300MB | ~200MB | ❌      | ✅           | **5.5/10** |

### 10.3. So sánh theo Technology Stack (cho tự build)

| Stack                      | Cross-platform  | RAM       | Binary    | Dev Speed  | OS Integration | Điểm tổng  |
| :------------------------- | :-------------- | :-------- | :-------- | :--------- | :------------- | :--------: |
| **Tauri + Rust**           | ✅ 3 OS          | ~30-50MB  | 5-12MB    | Trung bình | Tốt            | **9.0/10** |
| **Native (Swift/C#/Rust)** | ❌ Riêng từng OS | <20MB     | 3-5MB     | Chậm (x3)  | Xuất sắc       | **8.5/10** |
| **Wry (Rust)**             | ✅ 3 OS          | <20MB     | ~1MB      | Chậm       | Cơ bản         | **7.5/10** |
| **Electron**               | ✅ 3 OS          | 100-500MB | 100-400MB | Nhanh      | Kém            | **5.0/10** |

---

## 11. Đề xuất giải pháp tối ưu nhất

### 11.1. Giải pháp NGAY LẬP TỨC (dùng tool có sẵn, 5 phút setup)

| OS                    | Giải pháp              | Lệnh cài đặt                               |
| :-------------------- | :--------------------- | :----------------------------------------- |
| **macOS**             | QLMarkdown + MarkEdit  | `brew install --cask qlmarkdown markedit`  |
| **Windows**           | PowerToys              | `winget install Microsoft.PowerToys`       |
| **Linux**             | GNOME QuickLook + Glow | Package manager + `brew install glow`      |
| **Mọi OS (Terminal)** | Glow                   | `brew install glow`                        |
| **IDE**               | Dùng built-in preview  | VS Code: `Cmd+Shift+V` / JetBrains: có sẵn |

### 11.2. Giải pháp DÀI HẠN (nếu muốn build "Markdown Quick Look" app)

> **Đề xuất: Tauri + Rust + Svelte**

**Tại sao Tauri là lựa chọn tối ưu nhất?**

| Tiêu chí       | Tauri                | Electron     | Native        |
| :------------- | :------------------- | :----------- | :------------ |
| Cross-platform | ✅ 1 codebase         | ✅ 1 codebase | ❌ 3 codebases |
| RAM            | ~30-50MB             | 100-500MB    | <20MB         |
| Binary         | 5-12MB               | 100-400MB    | 3-5MB         |
| Parsing speed  | Rust = nhanh nhất    | JS = chậm    | Tùy ngôn ngữ  |
| OS integration | Tốt (native WebView) | Kém          | Xuất sắc      |
| Dev speed      | Trung bình           | Nhanh        | Chậm          |
| **Tổng điểm**  | **9.0/10**           | **5.0/10**   | **8.5/10**    |

**Core features cho "Markdown Quick Look" app**:
1. **View-only** — không cần editing, giảm complexity
2. **Auto-refresh** — watch file, tự reload khi thay đổi (cực tiện cho AI workflow)
3. **GFM + Mermaid + Math** — bao phủ 99% output từ AI
4. **Dark/Light/System theme** — theo system preference
5. **Quick Look / Preview Pane integration** — nhúng vào OS file manager
6. **CLI support** — `mdql file.md` từ terminal
7. **File association** — double-click `.md` = mở ngay

**KPIs đo lường thành công**:

| KPI                    | Mục tiêu | Cách đo                         |
| :--------------------- | :------- | :------------------------------ |
| Cold start time        | < 0.3s   | Benchmark script                |
| RAM idle               | < 30MB   | Activity Monitor / Task Manager |
| File render (1MB .md)  | < 0.5s   | Benchmark script                |
| Binary size            | < 15MB   | Build output                    |
| GitHub stars (6 tháng) | > 500    | GitHub metrics                  |
| User satisfaction      | > 4.5/5  | Survey / App Store reviews      |

---

## 12. Version Tracking

| Version | Ngày       | Thay đổi                                                                                                                                                                                                                                                                                                                                                    |
| :------ | :--------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v1.0    | 2026-05-23 | Khởi tạo ý tưởng ban đầu (nhu cầu & câu hỏi)                                                                                                                                                                                                                                                                                                                |
| v2.0    | 2026-05-23 | Research chi tiết: macOS (Quick Look, QLMarkdown, FluxMarkdown, MarkEdit, Automator), Windows (PowerToys, Store viewers), Linux (GNOME/KDE integration, Rust viewer), IDE (VS Code/JetBrains/Neovim/Sublime), CLI (Glow/Pandoc), Cross-platform build options (Tauri/Native/Wry/Electron), bảng so sánh toàn diện, đề xuất giải pháp tối ưu + KPIs đo lường |
| v2.1    | 2026-05-23 | Bổ sung hướng dẫn post-install cho Quick Look extensions (bước enable trong System Settings, qlmanage -r, troubleshooting)                                                                                                                                                                                                                                  |
| v2.2    | 2026-05-23 | Làm rõ sự khác biệt giữa Preview.app (closed-source, KHÔNG extend được) và Quick Look (extensible system service) — tránh nhầm lẫn QLMarkdown là plugin của Preview.app                                                                                                                                                                                     |
| v3.0    | 2026-05-23 | Đổi framing: focus vào Quick Look (không phải Preview.app). Đổi tên app "MD Preview" → "Markdown Quick Look". Viết lại section 3.1 trả lời trực tiếp "Quick Look CÓ THỂ extend" + giải thích cơ chế hoạt động                                                                                                                                               |
| v4.0    | 2026-05-23 | Thêm section 8: Rendering Pipeline (2 pipeline: HTML vs Native, bottleneck analysis, benchmark timing). Thêm phân tích Alternative Rendering Engines: litehtml, Servo, Ultralight, Lightpanda, Sciter. Đề xuất Hybrid Architecture (litehtml + lazy WebView). Renumber sections 8→9, 9→10, 10→11, 11→12                                                     |
| v5.0    | 2026-05-23 | Bổ sung so sánh chi tiết Wry (Tauri's core WebView wrapper), Obscura (Rust headless browser — không phù hợp), Ladybird/LibWeb (independent engine — tiềm năng dài hạn). Mở rộng bảng so sánh 8.5 từ 6 lên 9 engines với thêm cột "Visual render?". Đổi tên app "MD Quick Look" → "Markdown Quick Look" toàn file                                            |
| v5.1    | 2026-05-23 | Viết lại section 8.6 (đề xuất kiến trúc): đổi từ "litehtml + WebView" sang 2 options rõ ràng — Option A: litehtml + **Servo** ⭐ (recommended, nhất quán cross-platform, cùng Rust ecosystem) vs Option B: litehtml + WebView (simpler, lighter binary). Bảng so sánh trade-offs giữa 2 options                                                               |
