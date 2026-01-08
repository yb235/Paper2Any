# Paper2Any User Guide for First-Time Users

## Welcome to Paper2Any! 🎉

This guide will help you understand what Paper2Any does, how to use it, and what makes it special. No technical background required!

---

## What is Paper2Any?

Paper2Any is an **AI-powered tool that transforms academic papers** into various useful formats:

- 📊 **Paper2Figure**: Convert your paper into beautiful model architecture diagrams
- 🎬 **Paper2PPT**: Generate presentation slides from your paper
- 🖼️ **PDF2PPT**: Convert PDF presentations while preserving layout (editable!)
- 🎨 **PPT Beautification**: Make your slides look professional with AI

Think of it as your **AI research assistant** that reads papers and creates visuals for you!

---

## How Does It Work? (Simple Explanation)

```
Your PDF Paper → AI Reads It → AI Creates Visuals → You Get Editable Results
```

**The Magic Behind the Scenes:**

1. **Upload**: You give Paper2Any your research paper (PDF)
2. **Read**: AI reads the paper and understands the main ideas
3. **Analyze**: AI figures out what visual would best represent your work
4. **Generate**: AI creates diagrams, charts, or slides
5. **Refine**: AI detects elements in the generated images and makes them editable
6. **Deliver**: You get a PowerPoint file you can edit and use!

---

## Quick Start Guide

### Option 1: Use the Web Interface (Easiest!)

1. **Visit the Website**: Open `http://localhost:3000` in your browser
2. **Upload Your Paper**: Click "Upload PDF" and select your paper
3. **Choose What You Want**: 
   - Model Diagram? → Select "Paper2Figure"
   - Presentation? → Select "Paper2PPT"
   - Convert PDF? → Select "PDF2PPT"
4. **Configure Settings** (optional):
   - Detail level (how detailed should the analysis be?)
   - Style preferences
   - Output format
5. **Click Generate**: Wait 1-3 minutes while AI works its magic
6. **Download Results**: Get your PowerPoint file!

### Option 2: Use Command Line (For Developers)

```bash
# Generate a model architecture diagram
python script/run_paper2figure.py

# Generate a presentation from paper
python script/run_paper2ppt.py

# Convert PDF to editable PPT
python script/run_pdf2ppt_with_paddle_sam_mineru.py
```

---

## Understanding the Results

### Paper2Figure Output

You'll get a **PowerPoint file with 2 slides**:

**Slide 1: Editable Diagram**
- All text is editable (click and type!)
- Images have transparent backgrounds
- Elements positioned exactly as in the original
- Can change colors, fonts, layout

**Slide 2: Original AI-Generated Image**
- High-quality reference image
- Shows what the AI originally created
- Good for comparison

**Why two slides?**
- Slide 1 is for editing and customization
- Slide 2 is your "source of truth" if you want to start over

### Example Use Case

**Scenario**: You wrote a paper about a new neural network architecture.

**Input**: Your 20-page PDF paper

**Output**: 
- A beautiful diagram showing your network's layers
- Each component labeled and positioned correctly
- Editable in PowerPoint for your defense presentation

---

## Common Questions

### Q: Do I need to install anything?

**A**: It depends on how you use it:

**Web Version** (Recommended for beginners):
- Only need a web browser
- Backend should be running (ask your admin)

**Local Installation**:
- Python 3.11+
- Some system tools (see installation guide)
- About 15 minutes to set up

### Q: What file formats are supported?

**A**: 
- **Input**: PDF, images (PNG/JPG), plain text
- **Output**: PowerPoint (.pptx), SVG, PNG, JSON

### Q: How long does it take?

**A**: 
- **Paper2Figure**: 1-3 minutes
- **Paper2PPT**: 2-5 minutes (longer papers take more time)
- **PDF2PPT**: 30 seconds - 2 minutes per page

### Q: Does it work with any paper?

**A**: Works best with:
- ✅ Computer Science papers
- ✅ Machine Learning papers
- ✅ Engineering papers with diagrams
- ⚠️ Papers with clear structure (abstract, intro, methods, results)

May struggle with:
- ❌ Pure mathematics (heavy equations)
- ❌ Very old papers (poor scan quality)
- ❌ Papers without English text

### Q: Can I customize the output?

**A**: Yes! Multiple ways:

1. **Before Generation**:
   - Adjust detail level (1-3, higher = more detailed)
   - Choose style preferences
   - Specify what you want to focus on

2. **After Generation**:
   - Edit text in PowerPoint
   - Move elements around
   - Change colors and fonts
   - Add your own images

3. **Iterative Refinement**:
   - Upload the generated image
   - Ask AI to modify specific parts
   - "Make the encoder boxes bigger"
   - "Add labels to the arrows"

### Q: Is my paper kept private?

**A**: 
- Papers are stored temporarily during processing
- Deleted after results are generated (configurable)
- If using the public demo, check the privacy policy
- For sensitive papers, install locally!

### Q: What if the result isn't perfect?

**A**: Several options:

1. **Try again with different settings**:
   - Increase detail level
   - Try a different style
   - Use more specific prompts

2. **Edit in PowerPoint**:
   - Fix small mistakes manually
   - Adjust positioning
   - Change colors/fonts

3. **Iterative refinement**:
   - Upload the generated image
   - Use the "edit" feature to refine
   - Give specific instructions

---

## Step-by-Step Tutorial: Paper2Figure

Let's walk through a complete example!

### Scenario
You have a paper about a Transformer model and want a diagram for your presentation.

### Step 1: Prepare Your Paper

✅ **Do**:
- Use a clear, well-formatted PDF
- Ensure the first few pages contain the abstract and introduction
- Check that the paper is readable (not a bad scan)

❌ **Don't**:
- Use password-protected PDFs
- Use corrupted or damaged files
- Use non-PDF formats (convert first!)

### Step 2: Upload and Configure

1. Go to the Paper2Figure page
2. Click "Upload PDF"
3. Select your paper
4. **Input Type**: Choose "PDF" (not "Text" or "Figure")
5. **Detail Level**: Start with "2" (medium detail)
6. **Aspect Ratio**: Choose based on your needs:
   - `16:9` for presentations
   - `1:1` for papers/posters
   - `4:3` for traditional slides

### Step 3: Configure AI Settings

**API Configuration**:
- **API URL**: Your LLM provider's endpoint
  - OpenAI: `https://api.openai.com/v1`
  - Custom: Your own endpoint
- **API Key**: Your API key (stored securely)
- **Model**: Choose model:
  - `gpt-4o` (best quality, slower)
  - `gpt-4o-mini` (faster, cheaper)
  - `claude-3-opus` (good alternative)

**Generation Settings**:
- **Mask Detail Level**: How detailed should element detection be?
  - `1`: Fast, basic detection
  - `2`: Balanced (recommended)
  - `3`: Very detailed, slower

### Step 4: Generate!

Click "Generate Figure" and wait. You'll see progress:

```
📄 Uploading paper... ✓
📖 Extracting paper content... ✓
💡 Analyzing key contributions... ✓
🎨 Generating visual description... ✓
🖼️ Creating diagram... ✓
🔍 Detecting elements... ✓
🎨 Cleaning up backgrounds... ✓
📊 Building PowerPoint... ✓
✅ Done!
```

### Step 5: Download and Review

**What you get**:
- `presentation_123456.pptx` - Your editable PowerPoint
- `figure_draft.jpg` - The original AI-generated image
- `mineru_result.json` - Technical data (for developers)

**Open PowerPoint and explore**:
- Slide 1: Click on any text to edit
- Slide 2: See the original high-res image
- Experiment with layouts and styles!

### Step 6: Refine (Optional)

Not perfect? No problem!

**Option A: Manual Edits**
- Open PowerPoint
- Edit text, move boxes, change colors
- Add your own elements

**Option B: AI Refinement**
1. Save Slide 2 as an image
2. Upload it back to Paper2Any
3. **Input Type**: Choose "Figure" (not PDF!)
4. **Edit Prompt**: "Make the attention mechanism more prominent"
5. Generate again!

---

## Understanding Different Input Types

Paper2Any can start from different points in the pipeline:

### Input Type: PDF
```
PDF → Extract Text → Analyze → Generate → Detect Layout → Output
```
**Use when**: You have a complete paper PDF

**Best for**: Creating diagrams from scratch

### Input Type: Text
```
Text → Analyze → Generate → Detect Layout → Output
```
**Use when**: You have text description of your work

**Best for**: Quick prototypes without a full paper

**Example text**:
```
This paper proposes a novel attention mechanism that combines 
self-attention with cross-attention in a hierarchical manner. 
The encoder processes input sequences, and the decoder generates 
outputs while attending to both the input and previous outputs.
```

### Input Type: Figure
```
Existing Image → Detect Layout → Output
```
**Use when**: You have an image that needs to be made editable

**Best for**: Converting hand-drawn diagrams or screenshots into editable PPT

---

## Tips for Best Results

### 📝 Paper Quality Matters

**Good Papers for Paper2Any**:
```
✅ Clear structure (abstract, intro, methods, results)
✅ Well-formatted PDF (not scanned)
✅ Contains visual descriptions in text
✅ Recent papers (2015+)
```

**Challenging Papers**:
```
⚠️ Pure text, no visual descriptions
⚠️ Heavily mathematical
⚠️ Poor scan quality
⚠️ Non-English
```

### 🎨 Prompt Engineering

When providing text input, be descriptive:

**Bad Prompt**:
```
A neural network
```

**Good Prompt**:
```
A three-layer neural network architecture with:
- Input layer (green, 784 neurons)
- Hidden layer (blue, 128 neurons) 
- Output layer (red, 10 neurons)
- Fully connected with ReLU activation
- Dropout between layers
```

### ⚙️ Settings Optimization

**For Speed**:
- Detail level: 1
- Model: gpt-4o-mini
- Skip background removal

**For Quality**:
- Detail level: 3
- Model: gpt-4o or claude-3-opus
- Enable background removal
- Use iterative refinement

**For Balance** (Recommended):
- Detail level: 2
- Model: gpt-4o
- Enable background removal

---

## Troubleshooting

### Problem: "Upload Failed"

**Possible Causes**:
- File too large (>10MB)
- Invalid file format
- Network issue

**Solutions**:
1. Compress your PDF (use Adobe Acrobat or online tools)
2. Ensure it's a valid PDF (open in PDF reader to verify)
3. Try again (might be temporary network issue)

### Problem: "Generation Failed"

**Possible Causes**:
- API key invalid or expired
- LLM service down
- Model server (MinerU) not running

**Solutions**:
1. Verify API key is correct
2. Check API URL is reachable
3. Contact admin if using shared instance
4. Try with a different model

### Problem: "Results Look Wrong"

**Possible Causes**:
- Paper's content ambiguous
- Not enough context in first 10 pages
- Wrong input type selected

**Solutions**:
1. Try increasing detail level
2. Use "Text" input type with explicit description
3. Manually edit the output in PowerPoint
4. Try iterative refinement with specific instructions

### Problem: "Text is Blurry in Output"

**Cause**: Layout detection OCR quality

**Solutions**:
1. Increase mask detail level to 3
2. Manually replace text in PowerPoint
3. Use higher resolution input image

---

## Advanced Features

### Batch Processing (CLI)

Process multiple papers at once:

```bash
# Create a JSON file with paper list
cat > papers.json << EOF
[
  {"paper": "paper1.pdf", "output": "results/paper1/"},
  {"paper": "paper2.pdf", "output": "results/paper2/"},
  {"paper": "paper3.pdf", "output": "results/paper3/"}
]
EOF

# Run batch processing
python script/batch_process.py --input papers.json
```

### API Integration

Use Paper2Any in your own applications:

```python
import requests

# Upload paper
with open("paper.pdf", "rb") as f:
    response = requests.post(
        "http://localhost:8000/api/paper2figure",
        files={"pdf_file": f},
        data={
            "input_type": "PDF",
            "mask_detail_level": 2,
            "api_key": "your-key",
            "model": "gpt-4o"
        }
    )

result = response.json()
ppt_url = result["ppt_url"]
print(f"Download your PPT: {ppt_url}")
```

### Custom Styles

Create your own style templates:

1. Generate a figure with default style
2. Edit in PowerPoint to your liking
3. Save as template
4. Apply to future generations

---

## Real-World Examples

### Example 1: Conference Presentation

**Need**: Create slides for a 15-minute conference talk

**Workflow**:
1. Use Paper2Figure for architecture diagram
2. Use Paper2PPT for content slides
3. Manual editing for final touches

**Time Saved**: ~4 hours of manual diagram creation!

### Example 2: Paper Submission

**Need**: High-quality figures for paper submission

**Workflow**:
1. Generate diagram with Paper2Figure
2. Export as SVG for scalability
3. Import to Inkscape for final adjustments

**Result**: Publication-quality vector graphics

### Example 3: Teaching Materials

**Need**: Explain complex models to students

**Workflow**:
1. Generate diagrams for multiple papers
2. Create comparative slide decks
3. Add annotations in PowerPoint

**Benefit**: Visual aids that clarify complex concepts

---

## Best Practices

### 1. Start Simple
- Use default settings first
- Understand what each option does
- Gradually experiment with advanced features

### 2. Iterate
- First generation rarely perfect
- Use iterative refinement
- Combine AI generation with manual editing

### 3. Organize Your Work
```
projects/
  my_paper/
    input/
      paper.pdf
    outputs/
      v1_draft/
        figure.pptx
      v2_refined/
        figure.pptx
      final/
        figure.pptx
```

### 4. Version Control
- Save each iteration
- Note what settings you used
- Keep track of what works for different paper types

### 5. Learn from Examples
- Browse the gallery (if available)
- Study well-generated figures
- Understand what makes them good

---

## Getting Help

### Documentation
- 📖 [Architecture Guide](./architecture.md) - Technical details
- 🔧 [API Reference](./api-reference.md) - Complete API documentation
- 💻 [Developer Guide](./developer-guide.md) - Extend Paper2Any

### Community
- 💬 GitHub Discussions: Ask questions, share results
- 🐛 GitHub Issues: Report bugs
- 📧 Email: Contact maintainers

### FAQ
- Check [FAQ.md](../faq.md) for common questions
- Search GitHub Issues for similar problems
- Read the changelog for recent updates

---

## What's Next?

Now that you understand Paper2Any basics:

1. **Try it out**: Generate your first diagram!
2. **Experiment**: Try different settings and input types
3. **Share**: Show your results to colleagues
4. **Contribute**: Help improve Paper2Any (report bugs, suggest features)

**Happy Creating! 🚀**

