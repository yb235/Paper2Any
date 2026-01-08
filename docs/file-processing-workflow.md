# File Processing Workflow - Deep Dive

## Overview

This document explains **how Paper2Any reads uploaded files and produces consistent analysis**. We'll trace the complete journey from file upload to structured output generation.

---

## The Complete File Processing Pipeline

```
┌─────────────┐
│  User       │
│  Uploads    │
│  PDF/Image  │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. UPLOAD & STORAGE                                          │
│    FastAPI receives file → Save to unique directory          │
│    outputs/{task_type}/{timestamp}_{uuid}/input/file.pdf    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. FILE TYPE DETECTION & ROUTING                            │
│    • PDF → Extract text + metadata                          │
│    • Image → Direct to image processing                     │
│    • Text → Direct to content generation                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
┌────────────┐   ┌────────────────┐   ┌───────────────┐
│  PDF Path  │   │  Image Path    │   │  Text Input   │
└─────┬──────┘   └──────┬─────────┘   └──────┬────────┘
      │                 │                    │
      ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. CONTENT EXTRACTION                                        │
│    PDF: PyMuPDF (fitz) → Extract first 10 pages             │
│    Image: PIL → Load image for processing                   │
│    Text: Direct use                                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. STRUCTURED ANALYSIS WITH LLM                              │
│    Agent: "paper_idea_extractor"                            │
│    Input: Raw text from pages 1-10                          │
│    Output: Structured summary of key contributions          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. VISUAL GENERATION                                         │
│    Agent: "figure_desc_generator"                           │
│    Input: Structured paper analysis                         │
│    Output: Visual description for image generation          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. IMAGE GENERATION                                          │
│    Tool: DALL-E / Stable Diffusion / Midjourney            │
│    Input: Visual description                                │
│    Output: Generated figure image                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. LAYOUT DETECTION (MinerU)                                │
│    • Vision-Language Model analyzes generated image         │
│    • Detects: text regions, image regions, tables           │
│    • Extracts: bounding boxes, OCR text, element types      │
│    • Recursive: Processes sub-elements up to depth N        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. POST-PROCESSING                                           │
│    • Background removal on detected elements                │
│    • Icon extraction and cleanup                            │
│    • Text refinement                                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 9. OUTPUT GENERATION                                         │
│    • PPT: python-pptx reconstructs layout                   │
│    • SVG: Inkscape processes vector graphics               │
│    • Metadata: JSON with all element information           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
                    ┌────────────┐
                    │  Results   │
                    │  Returned  │
                    │  to User   │
                    └────────────┘
```

---

## Deep Dive: Each Processing Stage

### Stage 1: Upload & Storage

**File**: `fastapi_app/routers/paper2any.py`

```python
@router.post("/paper2figure")
async def paper2figure_endpoint(
    pdf_file: UploadFile = File(...),
    input_type: str = Form(...),
    ...
):
    # Create unique directory for this request
    ts = datetime.utcnow().strftime("%Y%m%dT%H%M%S")
    rid = uuid.uuid4().hex[:6]
    run_dir = BASE_OUTPUT_DIR / "paper2figure" / f"{ts}_{rid}"
    
    # Create subdirectories
    (run_dir / "input").mkdir(parents=True, exist_ok=True)
    (run_dir / "output").mkdir(parents=True, exist_ok=True)
    
    # Save uploaded file
    pdf_path = run_dir / "input" / pdf_file.filename
    pdf_path.write_bytes(await pdf_file.read())
    
    # Structure:
    # outputs/
    #   paper2figure/
    #     20260108T123456_abc123/
    #       input/
    #         paper.pdf        ← User's uploaded file
    #       output/
    #         (generated files will go here)
```

**Key Design Decision**: Each request gets a unique directory. This:
- Prevents file collisions in concurrent requests
- Makes cleanup easy (delete entire directory)
- Preserves history for debugging
- Enables resumable processing

---

### Stage 2: File Type Detection & Routing

**File**: `dataflow_agent/workflow/wf_paper2figure.py`

```python
def set_entry_node(state: Paper2FigureState) -> str:
    """Conditional routing based on input type"""
    
    if state.input_type == "PDF":
        # PDF needs text extraction first
        log.critical('Entering PDF node...')
        return "paper_idea_extractor"
    
    elif state.input_type == "TEXT":
        # Text can go directly to figure description
        log.critical('Entering TEXT node...')
        return "figure_desc_generator"
    
    elif state.input_type == "FIGURE":
        # Existing figure goes to layout detection
        log.critical('Entering FIGURE node...')
        return "figure_mask_generator"
    
    else:
        log.error(f"Invalid input type: {state.input_type}")
        return "_end_"
```

**Why This Matters**:
- Avoids unnecessary processing (don't extract text from images)
- Allows users to start at different pipeline stages
- Makes workflow reusable for different use cases

---

### Stage 3: Content Extraction (PDF Case)

**File**: `dataflow_agent/workflow/wf_paper2figure.py` (pre_tool)

```python
@builder.pre_tool("paper_content", "paper_idea_extractor")
def _get_abstract_intro(state: Paper2FigureState):
    """
    Robustly extract content from PDF.
    This function runs BEFORE the agent executes.
    """
    
    # 1. Extract metadata title
    try:
        with open(state.paper_file, 'rb') as f:
            reader = PyPDF2.PdfReader(f)
            paper_title = reader.metadata.get('/Title', 'Unknown Title')
    except Exception:
        paper_title = "Unknown Title"
    
    # 2. Extract text from first 10 pages using PyMuPDF
    pdf_document = fitz.open(state.paper_file)
    text = ""
    for page_num in range(min(10, len(pdf_document))):
        page = pdf_document.load_page(page_num)
        text += page.get_text("text")
    
    # 3. Clean and format
    content = text.strip()
    
    # 4. Build structured input for agent
    final_text = (
        f"The title of the paper is {paper_title}\n\n"
        f"Here's first ten page content: {content}"
    )
    
    return final_text
```

**Why PyMuPDF (fitz)?**
- **Fast**: C++ backend, faster than pure Python parsers
- **Accurate**: Preserves text layout and reading order
- **Versatile**: Can extract images, annotations, metadata
- **Cross-platform**: Works on Linux, Windows, macOS

**Why First 10 Pages?**
- Contains most important content (abstract, intro, methods)
- Reduces LLM token usage
- Speeds up processing
- Still captures key contributions

---

### Stage 4: Structured Analysis with LLM

**File**: `dataflow_agent/agentroles/paper2any_agents/paper_idea_extractor.py`

```python
@register("paper_idea_extractor")
class PaperIdeaExtractorAgent(BaseAgent):
    """Extracts key contributions from academic papers"""
    
    @property
    def role_name(self) -> str:
        return "PaperIdeaExtractor"
    
    @property
    def system_prompt_template_name(self) -> str:
        return "paper_idea_extractor_system"
    
    @property
    def task_prompt_template_name(self) -> str:
        return "paper_idea_extractor_task"
```

**Prompt Template** (`dataflow_agent/promptstemplates/paper_idea_extractor_task.jinja2`):
```jinja2
You are an expert at reading academic papers and extracting their core contributions.

Given the following paper content:
{{ paper_content }}

Please analyze this paper and extract:
1. The main research question or problem being addressed
2. The key innovation or contribution
3. The methodology or approach used
4. The main results or findings

Return your analysis in JSON format:
{
  "research_question": "...",
  "key_innovation": "...",
  "methodology": "...",
  "findings": "..."
}
```

**Execution Flow**:
```python
async def execute(self, state: MainState):
    # 1. Load templates
    system_prompt = load_template("paper_idea_extractor_system")
    task_prompt = load_template("paper_idea_extractor_task")
    
    # 2. Inject data from state (via pre_tool)
    task_prompt = task_prompt.render(
        paper_content=get_paper_content(state)  # From pre_tool
    )
    
    # 3. Build messages
    messages = [
        SystemMessage(content=system_prompt),
        HumanMessage(content=task_prompt)
    ]
    
    # 4. Call LLM
    llm = ChatOpenAI(
        model=state.request.model,
        api_key=state.request.api_key,
        base_url=state.request.chat_api_url
    )
    response = await llm.ainvoke(messages)
    
    # 5. Parse response (JSON)
    result = robust_parse_json(response.content)
    
    # 6. Store in state
    state.paper_idea = result["key_innovation"]
    state.agent_results["paper_idea_extractor"] = {
        "results": result,
        "timestamp": datetime.now().isoformat()
    }
    
    return state
```

**Why This Approach Works**:
1. **Structured Output**: JSON ensures consistent parsing
2. **Prompt Engineering**: Clear instructions → Better results
3. **Template Separation**: Easy to modify prompts without code changes
4. **Error Handling**: `robust_parse_json()` fixes common LLM JSON issues

---

### Stage 5: Visual Description Generation

**File**: `dataflow_agent/agentroles/paper2any_agents/fig_desc_generator.py`

```python
@register("figure_desc_generator")
class FigureDescGeneratorAgent(BaseAgent):
    """Converts paper analysis into visual descriptions"""
```

**Pre-Tool** (provides context):
```python
@builder.pre_tool("paper_idea", "figure_desc_generator")
def _get_paper_idea(state: Paper2FigureState):
    """Return structured paper analysis from previous agent"""
    return state.paper_idea
```

**Prompt Template**:
```jinja2
You are an expert at creating visual descriptions for scientific figures.

Based on this paper's key contribution:
{{ paper_idea }}

Generate a detailed description for a model architecture diagram that:
1. Visually represents the key innovation
2. Shows the flow of data/information
3. Highlights the novel components
4. Uses appropriate scientific visualization conventions

Return a JSON with:
{
  "fig_desc": "A detailed visual description...",
  "style": "diagram|flowchart|architecture",
  "key_elements": ["element1", "element2", ...]
}
```

**Output Example**:
```json
{
  "fig_desc": "A neural network architecture diagram with three main components: an encoder (left, blue boxes stacked vertically), a novel attention mechanism (center, green diamond with multi-head connections), and a decoder (right, orange boxes). Arrows show information flow from input (bottom) to output (top). The attention mechanism connects to both encoder and decoder with dotted lines.",
  "style": "architecture",
  "key_elements": ["encoder", "attention", "decoder", "residual_connections"]
}
```

---

### Stage 6: Image Generation

**File**: `dataflow_agent/toolkits/imtool/req_img.py`

```python
async def generate_or_edit_and_save_image_async(
    prompt: str,
    save_path: str,
    aspect_ratio: str = "16:9",
    api_url: str = None,
    api_key: str = None,
    model: str = "dall-e-3",
    image_path: str = None,
    use_edit: bool = False
):
    """
    Generate or edit an image using various APIs
    Supports: DALL-E, Stable Diffusion, Midjourney, Custom endpoints
    """
    
    if use_edit and image_path:
        # Image editing mode (for refinement)
        response = await client.post(
            f"{api_url}/images/edits",
            files={"image": open(image_path, "rb")},
            data={
                "prompt": prompt,
                "model": model,
                "size": map_aspect_ratio_to_size(aspect_ratio)
            },
            headers={"Authorization": f"Bearer {api_key}"}
        )
    else:
        # Image generation mode
        response = await client.post(
            f"{api_url}/images/generations",
            json={
                "prompt": prompt,
                "model": model,
                "size": map_aspect_ratio_to_size(aspect_ratio),
                "quality": "hd",
                "n": 1
            },
            headers={"Authorization": f"Bearer {api_key}"}
        )
    
    # Download and save image
    image_url = response.json()["data"][0]["url"]
    image_data = await client.get(image_url)
    Path(save_path).write_bytes(image_data.content)
    
    return save_path
```

**Key Features**:
- **Async**: Non-blocking, handles multiple requests
- **Flexible**: Supports multiple image generation APIs
- **Editing Support**: Can refine existing images
- **Error Handling**: Retries on failure

---

### Stage 7: Layout Detection with MinerU (The Magic Part!)

**File**: `dataflow_agent/utils.py` + `dataflow_agent/toolkits/imtool/mineru_tool.py`

This is **the most critical part** for consistent analysis. MinerU uses a Vision-Language Model (VLM) to understand the visual structure.

#### 7.1 MinerU HTTP API Call

```python
async def recursive_run_mineru_http(
    img_path: Path,
    out_dir: Path,
    port: int = 8010,
    max_depth: int = 2,
    current_depth: int = 0
) -> List[Dict[str, Any]]:
    """
    Recursively process image through MinerU VLM service
    
    Args:
        img_path: Path to image to analyze
        out_dir: Output directory for results
        port: MinerU service port (load-balanced if multiple)
        max_depth: How many levels of sub-elements to detect
        current_depth: Current recursion level
    
    Returns:
        List of detected elements with structure:
        [
          {
            "type": "text"|"image"|"table",
            "text": "...",  # for text elements
            "img_path": "...",  # for image elements
            "bbox": [x1, y1, x2, y2],  # bounding box
            "text_level": 1|2|3,  # heading level
            "page_idx": 0
          },
          ...
        ]
    """
    
    if current_depth > max_depth:
        return []  # Stop recursion
    
    log.info(f"[MinerU] Depth {current_depth}, processing: {img_path}")
    
    # 1. Call MinerU VLM API
    async with aiohttp.ClientSession() as session:
        with open(img_path, 'rb') as f:
            form = aiohttp.FormData()
            form.add_field('file', f, filename=img_path.name)
            
            async with session.post(
                f"http://localhost:{port}/extract",
                data=form
            ) as response:
                result = await response.json()
    
    # 2. Parse MinerU output
    items = parse_mineru_response(result, out_dir)
    
    # 3. Find sub-images in the result
    vlm_images_dir = out_dir / img_path.stem / 'vlm' / 'images'
    sub_images = list(vlm_images_dir.glob("*.jpg"))
    
    # 4. Recursively process sub-images
    if current_depth < max_depth:
        for sub_img_path in sub_images:
            # Recursive call
            sub_items = await recursive_run_mineru_http(
                sub_img_path,
                out_dir,
                port,
                max_depth,
                current_depth + 1
            )
            
            # 5. Coordinate transformation: sub-image coords → parent image coords
            items = transform_and_merge_subitems(
                items,
                sub_items,
                sub_img_path
            )
    
    return items
```

#### 7.2 MinerU Response Parsing

```python
def parse_mineru_response(result: Dict, out_dir: Path) -> List[Dict]:
    """
    Parse MinerU JSON response into standardized element list
    """
    elements = []
    
    # MinerU returns structure like:
    # {
    #   "pdf_info": [
    #     {
    #       "para_blocks": [
    #         {
    #           "type": "title"|"text"|"image"|"table",
    #           "bbox": [x1, y1, x2, y2],
    #           "lines": [...],
    #           "blocks": [...]  # nested elements
    #         }
    #       ]
    #     }
    #   ]
    # }
    
    for pdf_info in result.get("pdf_info", []):
        for block in pdf_info.get("para_blocks", []):
            block_type = block.get("type")
            
            if block_type in ["title", "text", "paragraph"]:
                # Extract text content
                text = extract_text_from_block(block)
                if text:
                    elements.append({
                        "type": "text",
                        "text": text,
                        "bbox": block.get("bbox", []),
                        "text_level": 1 if block_type == "title" else None,
                        "page_idx": 0
                    })
            
            elif block_type in ["image", "table"]:
                # Extract image path and metadata
                img_elements = extract_image_elements(block, out_dir)
                elements.extend(img_elements)
    
    return elements
```

#### 7.3 Coordinate Transformation (Critical!)

When processing sub-images, coordinates are relative to the sub-image. We need to transform them back to the parent image coordinate system.

```python
async def replace_item_with_sub_items(
    items: List[Dict],
    sub_items: List[Dict],
    sub_img_path: str
) -> List[Dict]:
    """
    Replace a parent image element with its detected sub-elements,
    transforming coordinates to parent's coordinate system.
    """
    
    # Find the parent item that corresponds to this sub-image
    parent_item = None
    parent_idx = None
    for i, item in enumerate(items):
        if item.get("img_path") == sub_img_path:
            parent_item = item
            parent_idx = i
            break
    
    if not parent_item:
        return items  # Sub-image not found in parent
    
    # Get parent's bounding box
    parent_bbox = parent_item["bbox"]
    px1, py1, px2, py2 = parent_bbox
    
    # Transform each sub-item's coordinates
    transformed_items = []
    for sub_item in sub_items:
        sub_bbox = sub_item["bbox"]
        sx1, sy1, sx2, sy2 = sub_bbox
        
        # Normalize sub-coordinates (0-1 range)
        sub_img = Image.open(sub_img_path)
        sub_w, sub_h = sub_img.size
        norm_sx1 = sx1 / sub_w
        norm_sy1 = sy1 / sub_h
        norm_sx2 = sx2 / sub_w
        norm_sy2 = sy2 / sub_h
        
        # Transform to parent coordinate space
        parent_w = px2 - px1
        parent_h = py2 - py1
        new_x1 = px1 + norm_sx1 * parent_w
        new_y1 = py1 + norm_sy1 * parent_h
        new_x2 = px1 + norm_sx2 * parent_w
        new_y2 = py1 + norm_sy2 * parent_h
        
        # Create transformed sub-item
        transformed_item = sub_item.copy()
        transformed_item["bbox"] = [new_x1, new_y1, new_x2, new_y2]
        transformed_items.append(transformed_item)
    
    # Remove parent item, add transformed sub-items
    items.pop(parent_idx)
    items.extend(transformed_items)
    
    return items
```

**Why Recursive + Coordinate Transform?**
- **Multi-scale analysis**: Captures both overview and details
- **Accurate positioning**: Elements positioned correctly in final output
- **Flexible depth**: Users can control detail level (max_depth=1,2,3...)

---

### Stage 8: Post-Processing

#### 8.1 Background Removal

**File**: `dataflow_agent/toolkits/imtool/bg_tool.py`

```python
def local_tool_for_bg_remove_batch(params: Dict) -> List[str]:
    """
    Batch remove backgrounds from images
    Uses RMBG (Remove Background) model
    """
    image_path_list = params["image_path_list"]
    model_path = params["model_path"]
    output_dir = params["output_dir"]
    
    # Load background removal model
    model = load_rmbg_model(model_path)
    
    output_paths = []
    for img_path in image_path_list:
        # Load image
        img = Image.open(img_path)
        
        # Remove background
        img_no_bg = model.remove_background(img)
        
        # Save with transparency
        output_path = Path(output_dir) / f"{Path(img_path).stem}_nobg.png"
        img_no_bg.save(output_path, "PNG")
        
        output_paths.append(str(output_path))
    
    return output_paths
```

**Why Remove Backgrounds?**
- **Cleaner visuals**: Icons/diagrams look more professional
- **Easy recomposition**: Transparent PNGs can be layered
- **Better for PPT**: Matches slide themes without clashing backgrounds

---

### Stage 9: Output Generation (PPT)

**File**: `dataflow_agent/workflow/wf_paper2figure.py`

```python
async def figure_ppt_generation_node(state: Paper2FigureState):
    """
    Generate PowerPoint presentation from detected layout
    """
    from pptx import Presentation
    from pptx.util import Inches, Pt
    from pptx.dml.color import RGBColor
    
    # 1. Create presentation
    prs = Presentation()
    
    # 2. Set slide size to match generated image
    img = Image.open(state.fig_draft_path)
    width_px, height_px = img.size
    prs.slide_width = Inches(width_px / 96)   # 96 DPI
    prs.slide_height = Inches(height_px / 96)
    
    # 3. Create Slide 1: Reconstructed layout
    slide1 = prs.slides.add_slide(prs.slide_layouts[6])  # Blank layout
    
    # 3a. Set background color
    background = slide1.background
    fill = background.fill
    fill.solid()
    fill.fore_color.rgb = RGBColor(188, 224, 254)  # Light blue
    
    # 3b. Add each detected element
    for element in state.fig_mask:
        if element["type"] == "text":
            # Add text box
            bbox = element["bbox"]
            left = Inches(bbox[0] / 96)
            top = Inches(bbox[1] / 96)
            width = Inches((bbox[2] - bbox[0]) / 96)
            height = Inches((bbox[3] - bbox[1]) / 96)
            
            textbox = slide1.shapes.add_textbox(left, top, width, height)
            text_frame = textbox.text_frame
            text_frame.text = element["text"]
            
            # Style based on text_level
            if element.get("text_level") == 1:
                # Title styling
                text_frame.paragraphs[0].font.size = Pt(24)
                text_frame.paragraphs[0].font.bold = True
            else:
                # Body text styling
                text_frame.paragraphs[0].font.size = Pt(12)
        
        elif element["type"] in ["image", "table"]:
            # Add image
            bbox = element["bbox"]
            img_path = element["img_path"]
            
            if Path(img_path).exists():
                left = Inches(bbox[0] / 96)
                top = Inches(bbox[1] / 96)
                width = Inches((bbox[2] - bbox[0]) / 96)
                height = Inches((bbox[3] - bbox[1]) / 96)
                
                slide1.shapes.add_picture(
                    img_path,
                    left, top,
                    width=width,
                    height=height
                )
    
    # 4. Create Slide 2: Original generated image (full-screen)
    slide2 = prs.slides.add_slide(prs.slide_layouts[6])
    
    # Calculate dimensions to fit slide
    page_w = prs.slide_width
    page_h = prs.slide_height
    ratio_w = page_w / Inches(width_px / 96)
    ratio_h = page_h / Inches(height_px / 96)
    ratio = min(ratio_w, ratio_h)
    
    final_w = Inches(width_px / 96) * ratio
    final_h = Inches(height_px / 96) * ratio
    left = (page_w - final_w) / 2
    top = (page_h - final_h) / 2
    
    slide2.shapes.add_picture(
        state.fig_draft_path,
        left, top,
        width=final_w,
        height=final_h
    )
    
    # 5. Save PPT
    output_path = state.result_path / f"presentation_{int(time.time())}.pptx"
    prs.save(str(output_path))
    
    state.ppt_path = output_path
    return state
```

**Why Two Slides?**
1. **Slide 1 (Editable)**: Reconstructed from layout detection
   - User can edit text, move elements
   - Professionally styled with backgrounds
   - Elements are native PPT objects
   
2. **Slide 2 (Original)**: Full generated image
   - Preserves original AI-generated quality
   - Good for reference/comparison
   - High-resolution export option

---

## What Makes This Processing Consistent

### 1. **State-Driven Execution**
All nodes read/write from a shared state object. This ensures:
- Data flows predictably through pipeline
- No hidden dependencies or global variables
- Easy to debug (inspect state at any point)
- Resumable processing (can save/load state)

### 2. **Structured Prompts**
Prompts use Jinja2 templates with JSON output specifications:
- LLMs return predictable structure
- Parsing is consistent
- Easy to version control prompts
- A/B testing different prompts

### 3. **Robust JSON Parsing**
The `robust_parse_json()` utility handles:
- Markdown code blocks (```json ... ```)
- Trailing commas (invalid JSON)
- Comments in JSON
- Multiple JSON objects in response
- LaTeX escaping issues
- Control characters

```python
def robust_parse_json(text: str):
    # Remove markdown fences
    text = re.sub(r'```[\w-]*\s*([\s\S]*?)```', r'\1', text)
    
    # Strip comments
    text = re.sub(r'//.*$', '', text, flags=re.MULTILINE)
    text = re.sub(r'/\*.*?\*/', '', text, flags=re.DOTALL)
    
    # Remove trailing commas
    text = re.sub(r',\s*([}\]])', r'\1', text)
    
    # Fix control characters
    text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]', '', text)
    
    # Escape unescaped backslashes (LaTeX formulas!)
    text = fix_latex_escaping(text)
    
    # Parse JSON
    return json.loads(text)
```

### 4. **Error Recovery**
Each processing stage has fallbacks:
- PDF extraction fails → Try pdfplumber as backup
- LLM times out → Retry with exponential backoff
- Image generation fails → Use placeholder or retry
- MinerU service down → Graceful degradation

### 5. **Validation Layers**
- **Input validation**: Pydantic models check request structure
- **File validation**: Check file types, sizes before processing
- **Output validation**: Verify generated files exist and are valid
- **State validation**: Ensure required fields exist at each stage

---

## Key Learnings & Design Patterns

### 1. **Separation of Concerns**
- **FastAPI**: HTTP handling only, no business logic
- **Workflows**: Orchestration only, delegate to agents
- **Agents**: Task execution only, use tools
- **Tools**: Atomic operations only, no complex logic

### 2. **Async/Await Throughout**
```python
# All I/O operations are async
async def process_pdf(path: str):
    text = await extract_text_async(path)
    analysis = await llm_analyze_async(text)
    image = await generate_image_async(analysis)
    layout = await detect_layout_async(image)
    return await generate_ppt_async(layout)
```

Benefits:
- Non-blocking I/O → higher throughput
- Concurrent processing of multiple requests
- Efficient use of GPU resources

### 3. **Idempotent Operations**
Each stage can be re-run safely:
- Files saved with unique timestamps
- Intermediate results cached
- No side effects beyond defined outputs

### 4. **Observability**
Comprehensive logging at every stage:
```python
log.info(f"[stage_name] Starting with input: {input_summary}")
log.debug(f"[stage_name] Intermediate result: {result}")
log.error(f"[stage_name] Failed: {error}", exc_info=True)
```

Users can trace exactly what happened during processing.

---

## Performance Optimizations

1. **Batch Processing**: Background removal processes all images at once
2. **Caching**: LLM responses cached by prompt hash
3. **Load Balancing**: MinerU/SAM services run on multiple GPUs
4. **Lazy Loading**: Only load heavy models when needed
5. **Parallel Execution**: Independent workflow branches run concurrently

---

## Next Steps

- Read [API Reference](./api-reference.md) for complete endpoint documentation
- Read [Developer Guide](./developer-guide.md) to create custom processors
- Read [Architecture Overview](./architecture.md) for system design details

