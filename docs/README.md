# Paper2Any Documentation

<div align="center">

**Transform Academic Papers into Professional Visuals with AI**

[Quick Start](#-quick-start) | [Documentation](#-documentation-index) | [Support](#-support)

</div>

---

## �� Documentation Index

### 🎯 For First-Time Users

- **[📖 User Guide](user-guide.md)** - Complete beginner tutorial

### 🏗️ For Developers & Technical Analysis

- **[🏛️ Architecture Overview](architecture.md)** - System design deep-dive
- **[📄 File Processing Workflow](file-processing-workflow.md)** - **⭐ ANSWERS YOUR QUESTION**
  - How uploaded files are read and processed
  - MinerU layout detection explained
  - Consistent analysis mechanisms
  - Design patterns you can learn from
- **[🔌 API Reference](api-reference.md)** - Complete API documentation

### 📖 More Resources

- [⚡ Quick Start](quickstart.md) | [💻 CLI Guide](cli.md) | [❓ FAQ](faq.md)
- [🤝 Contributing](contributing.md) | [📝 Changelog](changelog.md)

---

## 💡 What is Paper2Any?

An **AI-powered multi-agent system** that transforms research papers into professional visuals:

- 📊 **Paper2Figure** - Model architecture diagrams
- 🎬 **Paper2PPT** - Presentation slides
- 🖼️ **PDF2PPT** - Editable PowerPoint conversions

**Technology Stack:**
- LLM analysis (GPT-4/Claude)
- AI image generation (DALL-E/SD)
- Layout detection (MinerU VLM)
- Workflow orchestration (LangGraph)

---

## 🚀 Quick Start

```bash
# Web Interface
cd fastapi_app && uvicorn main:app --port 8000
cd frontend-workflow && npm run dev

# CLI
python script/run_paper2figure.py
```

Visit http://localhost:3000

---

## 📞 Support

- 📖 [FAQ](faq.md) | 🐛 [Issues](https://github.com/OpenDCAI/Paper2Any/issues)
- 💬 [Discussions](https://github.com/OpenDCAI/Paper2Any/discussions)

---

<div align="center">

**Made with ❤️ by OpenDCAI Team**

</div>
