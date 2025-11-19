# 🎯 Universal Conversation Splitter v2.1

**By Tri-Octa Harvest Designs (2025)**

A powerful, JSON-driven conversation parser that transforms AI chat exports into beautiful, readable formats. Supports ChatGPT, Claude, and Gemini with gorgeous HTML themes, enhanced Markdown, and plain text output.

---

## ✨ Features

### 🎨 Beautiful HTML Output
- **Light/Dark theme toggle** - Easy on your eyes, day or night
- **Font size controls** - Small, Medium, Large for accessibility
- **Emoji show/hide** - Professional mode when you need it
- **Speech bubble design** - Clear visual distinction (User = Blue, AI = Green)
- **Sticky controls** - Always accessible at the top
- **Print-friendly** - Optimized for PDF export
- **LocalStorage** - Remembers your preferences!
- **Responsive design** - Works beautifully on mobile

### 📝 Enhanced Markdown
- **Bold headers** - Quick navigation in Word/Google Docs
- **Smart code detection** - Auto-wraps code blocks
- **Rich frontmatter** - Metadata for organization
- **Clean hierarchy** - Professional document structure
- **Annotation-friendly** - Space for your notes

### 🔧 Technical Excellence
- **JSON-driven configs** - Easily extensible to new platforms
- **Auto-platform detection** - Identifies ChatGPT, Claude, or Gemini
- **Robust error handling** - Graceful fallbacks for edge cases
- **Progress indicators** - Know where you are in large batches
- **Unicode support** - Handles international characters

---

## 🚀 Quick Start

### Prerequisites
- Python 3.7 or higher
- No external dependencies! (Uses only standard library)

### Installation

1. **Download the script:**
   ```bash
   # Save UniversalSplitter_dev1.3.py to your working directory
   ```

2. **Set up config files (optional):**
   ```bash
   mkdir configs
   # Add your platform-specific JSON configs if needed
   # The script works with built-in fallback configs!
   ```

3. **Run it:**
   ```bash
   python UniversalSplitter_dev1.3.py
   ```

---

## 📖 Usage

### Interactive Mode

Simply run the script and follow the prompts:

```bash
$ python UniversalSplitter_dev1.3.py

============================================================
 Universal Conversation Splitter v2.1 (dev1.3)
 By Tri-Octa Harvest Designs
============================================================

Enter conversation file (default: conversations.json): 
📝 Choose output format:
  [1] Markdown (.md) - Great for editing and note-taking
  [2] HTML (.html) - Beautiful, interactive, theme-able
  [3] Plain Text (.txt) - Simple and universal
Enter choice (1/2/3, default: 1): 2

📁 Enter output folder (default: ConversationExports): MyChats
👤 Enter your name (default: User): James
🤖 Enter AI name (default: Assistant): Claude
```

### Supported Input Formats

#### ChatGPT
- **Format:** JSON export from ChatGPT settings
- **File:** `conversations.json`
- **How to export:**
  1. ChatGPT Settings → Data Controls
  2. Export Data
  3. Download the ZIP, extract `conversations.json`

#### Claude
- **Format:** JSON export from Claude settings
- **File:** `conversations.json`
- **How to export:**
  1. Claude Settings → Export Data
  2. Download JSON file

#### Gemini
- **Format:** HTML from Google Takeout
- **File:** `*.html`
- **How to export:**
  1. Google Takeout → Select Gemini
  2. Download the export
  3. Use the HTML file

---

## 🎨 Output Formats

### 1. Markdown (.md)
Perfect for documentation and note-taking!

**Features:**
- Bold headers with timestamps
- Smart code block detection
- Rich frontmatter metadata
- Clean separators between messages
- Easy to edit and annotate

**Best for:**
- Importing into Obsidian, Notion, or other note apps
- Creating documentation
- Sharing with collaborators
- Long-term archival with editability

### 2. HTML (.html)
Gorgeous, interactive reading experience!

**Features:**
- Light/Dark theme toggle
- Font size controls (Small/Medium/Large)
- Emoji show/hide button
- Beautiful speech bubbles
- Print to PDF functionality
- Responsive mobile design
- LocalStorage for preferences

**Best for:**
- Reading and reviewing conversations
- Sharing with non-technical users
- Professional presentations
- Archival with beautiful formatting

### 3. Plain Text (.txt)
Simple and universal!

**Features:**
- No formatting, just text
- Clean timestamps
- Simple separators

**Best for:**
- Maximum compatibility
- Processing with other tools
- Raw text analysis
- Minimal file size

---

## 🔧 Advanced Configuration

### Custom Platform Detection

Create `configs/platform_detection.json`:

```json
{
  "chatgpt": {
    "required_keys": ["mapping"],
    "optional_keys": ["title", "create_time"]
  },
  "claude": {
    "required_keys": ["chat_messages"],
    "optional_keys": ["name", "created_at"]
  },
  "detection_order": ["claude", "chatgpt"],
  "fallback_platform": "chatgpt"
}
```

### Custom Parser Rules

Create `configs/chatgpt_parser.json`:

```json
{
  "conversations_path": "",
  "message_container": "mapping",
  "title_path": "title",
  "author_path": "message.author.role",
  "content_path": "message.content.parts",
  "timestamp_path": "message.create_time",
  "timestamp_format": "unix",
  "user_role": "user",
  "ai_role": "assistant"
}
```

---

## 🐛 Troubleshooting

### "No conversations found in file!"
- **Check file format:** Ensure you're using the correct export format
- **Verify JSON:** Use a JSON validator to check for corruption
- **Try fallback:** The script uses built-in configs if yours are missing

### "Unknown" timestamps in output
- **Platform issue:** Some platforms don't include timestamps in certain contexts
- **Update needed:** Check for updated parser configs

### Character encoding issues
- **Emoji problems:** Ensure your terminal supports UTF-8
- **File encoding:** Save files as UTF-8 with BOM if needed

### KeyError on parsing
- **Config mismatch:** Check that your parser configs match the data structure
- **Use defaults:** Remove custom configs to use built-in fallbacks

---

## 🎯 Tips & Tricks

### Batch Processing
Process hundreds of conversations at once! The script automatically:
- Shows progress every 25-50 conversations
- Handles large files efficiently
- Creates numbered output files

### HTML Theme Customization
Open any HTML output and use the built-in controls:
- **Font Size:** Adjust for readability
- **Theme:** Switch between light/dark
- **Emojis:** Hide for professional contexts

All preferences are saved in your browser's LocalStorage!

### Markdown Workflow
1. Export to Markdown
2. Import into your favorite note-taking app
3. Add your own notes between messages
4. The bold headers create a clickable table of contents!

### PDF Archival
1. Export to HTML format
2. Open in your browser
3. Adjust font size and theme as desired
4. Press Ctrl+P (or Cmd+P)
5. Select "Save as PDF"
6. You now have a beautiful, permanent archive!

---

## 📊 Performance

**Tested with:**
- ✅ 234 ChatGPT conversations (2 minutes)
- ✅ 500+ messages per conversation
- ✅ Large HTML exports (10+ MB)
- ✅ Unicode and emoji-heavy content
- ✅ Edge cases (empty messages, malformed data)

**Processing Speed:**
- ~100 conversations per minute (typical)
- Parallel processing not needed (fast enough!)
- Memory efficient (processes incrementally)

---

## 🛠️ Technical Architecture

### Core Components

```
UniversalSplitterEngine
├── Platform Detection (auto-identifies source)
├── JSON Config System (data-driven rules)
├── Parser Modules
│   ├── ChatGPT Parser
│   ├── Claude Parser
│   └── Gemini HTML Parser
└── Output Generators
    ├── Markdown Generator
    ├── HTML Generator
    └── Text Generator
```

### Design Philosophy

**Why this approach?**
- **Extensible:** Add new platforms with just JSON configs
- **Maintainable:** Logic separated from data
- **Robust:** Graceful fallbacks for edge cases
- **User-friendly:** Clean CLI with sensible defaults

---

## 🚧 Known Limitations

### Gemini Exports
- Timestamp extraction can be tricky with some formats
- Very long conversations may have parsing edge cases
- Working on improved HTML structure detection

### Platform-Specific
- ChatGPT: Some system messages are filtered out
- Claude: Requires specific export format
- Gemini: HTML parsing is more complex than JSON

### Future Enhancements
- Search/filter functionality in HTML
- Export to PDF button (instead of print)
- Conversation merging for fragmented exports
- Batch mode for multiple files at once

---

## 📝 Changelog

### v2.1 (dev1.3) - Current
- ✅ Fixed config key compatibility issues
- ✅ Fixed emoji/copyright HTML encoding
- ✅ Filter empty messages
- ✅ Safe fallbacks for missing config keys

### v2.1 (dev1.2)
- ✨ Beautiful HTML with themes and controls
- ✨ Enhanced Markdown with code detection
- 🧹 Cleaned up all debug output
- 🎨 Professional styling throughout

### v2.0
- 🚀 Complete JSON-driven rewrite
- 🎯 Multi-platform support
- 📦 Modular architecture

---

## 🤝 Contributing

This is a personal project by **Tri-Octa Harvest Designs**, but if you find bugs or have suggestions:
- Document the issue with example data
- Note your Python version and platform
- Share your use case

---

## 📄 License

**One-Time Purchase Model** (WinRAR-style)
- Personal use: Free forever
- Commercial use: Contact for licensing
- No subscription, no SaaS, no data collection
- Your data stays on your machine

---

## 🎓 Philosophy

> "Complex but well-documented beats simple but limited."

This tool is built for **power users** who value:
- **Privacy:** All processing happens locally
- **Control:** You own your data and outputs
- **Quality:** Professional results worth the setup
- **Extensibility:** Adapt it to your needs

---

## 💬 Support

Having issues? Check:
1. This README (most questions answered here!)
2. Your Python version (3.7+ required)
3. File format (correct export type?)
4. Error messages (usually self-explanatory)

For complex issues, provide:
- Error message (full traceback)
- Input file format (ChatGPT/Claude/Gemini)
- Python version
- Operating system

---

## 🎯 Project Goals

**What this tool IS:**
- A robust, local conversation parser
- A privacy-respecting archival tool
- A beautiful output generator
- An extensible platform for future formats

**What this tool ISN'T:**
- A cloud service (no servers!)
- A data collection tool (no analytics!)
- A subscription product (pay once, use forever!)
- A simplified toy (complexity with documentation!)

---

## 🙏 Acknowledgments

Built with:
- Python's standard library (no external dependencies!)
- CSS Variables for theming
- LocalStorage API for preferences
- Love for well-crafted tools ❤️

Developed by **James** at **Tri-Octa Harvest Designs**
- Philosophy: Privacy-first, user-controlled, one-time purchase
- Inspired by the WinRAR model of software ethics

---

**Made with ❤️ by Tri-Octa Harvest Designs**

*"Because your conversations deserve better than plain text dumps."*
