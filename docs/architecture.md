# Paper2Any Architecture Overview

## Introduction

Paper2Any is a sophisticated AI-powered system that transforms academic papers into various formats (figures, presentations, videos) using a **multi-agent workflow orchestration** architecture. This document provides a comprehensive technical overview for developers and first-time users.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Frontend Layer                            │
│  (React + TypeScript + Vite + Supabase Auth)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTP/REST API
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FastAPI Backend Layer                         │
│  • File Upload & Management                                      │
│  • Request Routing & Validation                                  │
│  • Workflow Adapters                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │ Async Execution
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   DataFlow-Agent Core                            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐      │
│  │   Workflow   │  │    Agents    │  │   Tool Manager  │      │
│  │ Orchestrator │←→│   (Roles)    │←→│    (Toolkits)   │      │
│  └──────────────┘  └──────────────┘  └─────────────────┘      │
│         ▲                 ▲                    ▲                 │
│         └─────────────────┴────────────────────┘                │
│                   LangGraph StateGraph Engine                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    External Services Layer                       │
│  • LLM APIs (OpenAI, Claude, etc.)                              │
│  • MinerU (PDF Layout Detection)                                │
│  • SAM (Segment Anything Model)                                 │
│  • Background Removal Service                                   │
│  • Image Generation APIs                                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Frontend Layer (`frontend-workflow/`)

**Technology Stack**: React, TypeScript, Vite, TailwindCSS, Supabase

**Purpose**: Provides the user interface for interacting with Paper2Any workflows.

**Key Features**:
- File upload interface (PDF, images, text)
- Workflow configuration forms
- Real-time progress tracking
- Result preview and download
- User authentication via Supabase

**Example Flow**:
```typescript
// User uploads a PDF → Frontend sends to FastAPI
POST /api/paper2figure
{
  "pdf_file": <binary>,
  "input_type": "PDF",
  "mask_detail_level": 2,
  "api_key": "...",
  "model": "gpt-4o"
}
```

---

### 2. FastAPI Backend Layer (`fastapi_app/`)

**Purpose**: HTTP API gateway that handles requests, manages file uploads, and triggers workflows.

**Key Files**:
- `main.py`: Application entry point, CORS, middleware setup
- `routers/paper2any.py`: Main API endpoints for Paper2Figure/Paper2PPT
- `routers/pdf2ppt.py`: PDF-to-PPT conversion endpoints
- `schemas.py`: Pydantic models for request/response validation
- `workflow_adapters.py`: Adapters that bridge HTTP requests to workflow execution

**Request Flow**:
```
1. User Upload → FastAPI receives file
2. Create unique run directory: outputs/{task_type}/{timestamp}_{uuid}/
3. Save uploaded file to input/ subdirectory
4. Trigger workflow execution (async)
5. Monitor workflow state
6. Return result URLs when complete
```

**Example: Paper2Figure Endpoint**
```python
@router.post("/paper2figure")
async def paper2figure_endpoint(
    pdf_file: UploadFile,
    input_type: str,
    mask_detail_level: int,
    api_key: str,
    ...
):
    # 1. Create run directory
    run_dir = create_run_dir("paper2figure")
    
    # 2. Save uploaded file
    pdf_path = run_dir / "input" / pdf_file.filename
    pdf_path.write_bytes(await pdf_file.read())
    
    # 3. Build state object
    state = Paper2FigureState(
        paper_file=str(pdf_path),
        input_type=input_type,
        mask_detail_level=mask_detail_level,
        ...
    )
    
    # 4. Execute workflow
    result = await run_paper2figure_wf_api(state)
    
    # 5. Return result URLs
    return Paper2FigureResponse(
        ppt_url=_to_outputs_url(result.ppt_path),
        ...
    )
```

---

### 3. DataFlow-Agent Core (`dataflow_agent/`)

This is the **heart** of Paper2Any - a modular, agent-based workflow orchestration system.

#### 3.1 Workflow System (`dataflow_agent/workflow/`)

**Purpose**: Defines multi-step processing pipelines as directed acyclic graphs (DAGs) using LangGraph.

**Key Concepts**:
- **Workflow = StateGraph**: Each workflow is a state machine with nodes (processing steps) and edges (transitions)
- **State**: Carries data between nodes (e.g., `Paper2FigureState`, `Paper2PPTState`)
- **Nodes**: Async functions that perform specific tasks (e.g., extract text, generate images)
- **Edges**: Define execution order (sequential or conditional)

**Example: Paper2Figure Workflow** (`wf_paper2figure.py`):
```python
@register("paper2fig")
def create_p2fig_graph() -> GenericGraphBuilder:
    builder = GenericGraphBuilder(
        state_model=Paper2FigureState,
        entry_point="_start_"
    )
    
    # Define nodes
    nodes = {
        '_start_': lambda state: state,
        "paper_idea_extractor": paper_idea_extractor_node,
        "figure_desc_generator": figure_desc_generator_node,
        "figure_generator": figure_generator_node,
        "figure_mask_generator": figure_mask_generator_node,
        "figure_icon_bg_remover": figure_icon_bg_remover_node,
        "figure_ppt_generator": figure_ppt_generation_node,
        '_end_': lambda state: state,
    }
    
    # Define edges
    edges = [
        ("paper_idea_extractor", "figure_desc_generator"),
        ("figure_desc_generator", "figure_generator"),
        ("figure_generator", "figure_mask_generator"),
        ("figure_mask_generator", "figure_icon_bg_remover"),
        ("figure_icon_bg_remover", "figure_ppt_generator"),
        ("figure_ppt_generator", "_end_"),
    ]
    
    # Conditional entry based on input type
    builder.add_conditional_edge("_start_", set_entry_node)
    
    return builder.add_nodes(nodes).add_edges(edges)
```

**Workflow Registry**:
- Workflows auto-register via `@register("workflow_name")` decorator
- Naming convention: `wf_*.py` files in `workflow/` directory
- CLI/API can invoke by name: `dfa run --wf paper2fig`

---

#### 3.2 Agent System (`dataflow_agent/agentroles/`)

**Purpose**: Intelligent agents that perform specific tasks using LLMs and tools.

**Architecture**:
```
BaseAgent (Abstract Base Class)
    ↓
Common Agents (dataflow_agent/agentroles/cores/)
    ↓
Domain-Specific Agents (dataflow_agent/agentroles/paper2any_agents/)
    → paper_idea_extractor
    → figure_desc_generator
    → icon_generator
    → outline_agent
    → chart_type_recommender
    → ...
```

**Key Features**:
1. **Auto-Registration**: Agents register via `@register("agent_name")` decorator
2. **Prompt Templates**: Agents use Jinja2 templates for system/task prompts
3. **Tool Integration**: Agents can use tools from the ToolManager
4. **Execution Strategies**: Simple, ReAct, Parallel, Graph modes
5. **Message History**: Tracks conversation context with LLMs

**Example: Paper Idea Extractor Agent**
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
    
    async def execute(self, state: MainState, use_agent: bool = True):
        # 1. Build prompt from template + state data
        # 2. Call LLM with prompt
        # 3. Parse response
        # 4. Update state with results
        return state
```

**Agent Execution Flow**:
```
1. Load prompt templates (system + task)
2. Inject state data into templates (pre_tools)
3. Build messages for LLM
4. Call LLM API (OpenAI, Claude, etc.)
5. Parse LLM response (JSON, XML, text)
6. Run post-processing tools
7. Update state with results
```

---

#### 3.3 Tool System (`dataflow_agent/toolkits/`)

**Purpose**: Reusable utilities that agents and workflows can call.

**Toolkit Categories**:
- **filetool**: File reading, directory listing
- **imtool**: Image generation, editing, background removal
- **optool**: Operator-specific tools
- **pipetool**: Pipeline utilities
- **p2vtool**: Paper-to-video tools
- **model_servers**: Integration with MinerU, SAM, OCR services

**Tool Manager**:
```python
tool_manager = ToolManager()
tool_manager.register_tool("read_file", read_text_file)
tool_manager.register_tool("generate_image", generate_image_async)

# Agents can access tools
agent = create_agent("my_agent", tool_manager=tool_manager)
```

**Example: MinerU Integration** (Layout Detection):
```python
# MinerU extracts layout information from PDF/images
items = await recursive_run_mineru_http(
    img_path=image_path,
    out_dir=output_dir,
    port=8010,
    max_depth=2  # Recursive depth for sub-elements
)

# Returns structured layout data:
# [
#   {
#     "type": "text",
#     "text": "Introduction",
#     "bbox": [100, 200, 300, 250],
#     "text_level": 1
#   },
#   {
#     "type": "image",
#     "img_path": "/path/to/figure.png",
#     "bbox": [100, 300, 500, 600]
#   }
# ]
```

---

### 4. State Management

**Purpose**: Carry data through the workflow pipeline.

**State Hierarchy**:
```
MainState (base)
    ├── request: MainRequest
    ├── messages: List[BaseMessage]
    ├── agent_results: Dict[str, Any]
    └── temp_data: Dict[str, Any]

DFState (extends MainState)
    └── request: DFRequest (extended)

Paper2FigureState (domain-specific)
    ├── paper_file: str
    ├── input_type: str
    ├── paper_idea: str
    ├── fig_draft_path: str
    ├── fig_mask: List[Dict]
    ├── ppt_path: Path
    └── ...
```

**State Flow Example**:
```python
# Initial state
state = Paper2FigureState(
    paper_file="/path/to/paper.pdf",
    input_type="PDF"
)

# Node 1: Extract ideas
state = await paper_idea_extractor_node(state)
# → state.paper_idea = "This paper proposes..."

# Node 2: Generate figure description
state = await figure_desc_generator_node(state)
# → state.agent_results["figure_desc_generator"] = {...}

# Node 3: Generate image
state = await figure_generator_node(state)
# → state.fig_draft_path = "/tmp/output.jpg"

# ... continues through pipeline
```

---

## Detailed Workflow Execution

### Example: Paper2Figure Complete Flow

**Input**: User uploads a PDF paper → Request model architecture diagram

**Step-by-Step Execution**:

#### Step 1: File Upload & Initialization
```python
# FastAPI receives request
pdf_file = await request.files["pdf_file"]
run_dir = create_run_dir("paper2figure")
pdf_path = run_dir / "input" / "paper.pdf"

# Initialize state
state = Paper2FigureState(
    paper_file=str(pdf_path),
    input_type="PDF",
    mask_detail_level=2,
    request=MainRequest(
        api_key="sk-...",
        model="gpt-4o",
        language="en"
    )
)
```

#### Step 2: Extract Paper Content (Node 1)
```python
async def paper_idea_extractor_node(state):
    # Read PDF content (first 10 pages)
    pdf_document = fitz.open(state.paper_file)
    text = ""
    for page_num in range(min(10, len(pdf_document))):
        page = pdf_document.load_page(page_num)
        text += page.get_text("text")
    
    # Create agent
    agent = create_graph_agent("paper_idea_extractor")
    
    # Agent calls LLM with prompt:
    # "Extract the key contributions from this paper: {text}"
    
    state = await agent.execute(state, use_agent=True)
    
    # State now contains:
    # state.paper_idea = "This paper introduces a novel attention mechanism..."
    
    return state
```

#### Step 3: Generate Figure Description (Node 2)
```python
async def figure_desc_generator_node(state):
    agent = create_graph_agent("figure_desc_generator")
    
    # Agent uses paper_idea to create visual description
    # Prompt: "Based on this paper idea: {state.paper_idea},
    #          generate a detailed description for a model architecture diagram"
    
    state = await agent.execute(state, use_agent=True)
    
    # Results stored in:
    # state.agent_results["figure_desc_generator"]["results"]["fig_desc"]
    # → "A neural network diagram with encoder-decoder architecture..."
    
    return state
```

#### Step 4: Generate Image (Node 3)
```python
async def figure_generator_node(state):
    # Extract description from previous agent
    prompt = state.agent_results["figure_desc_generator"]["results"]["fig_desc"]
    
    # Call image generation API (DALL-E, Midjourney, etc.)
    save_path = f"tmps/figure_{timestamp}.jpg"
    
    await generate_or_edit_and_save_image_async(
        prompt=prompt,
        save_path=save_path,
        aspect_ratio=state.aspect_ratio,
        api_url=state.request.chat_api_url,
        api_key=state.request.api_key,
        model="dall-e-3"
    )
    
    state.fig_draft_path = save_path
    return state
```

#### Step 5: Layout Detection (Node 4)
```python
async def figure_mask_generator_node(state):
    img_path = Path(state.fig_draft_path)
    out_dir = build_output_directory(img_path)
    
    # Call MinerU service to detect layout elements
    # MinerU uses VLM (Vision-Language Model) to:
    # - Detect text regions
    # - Detect image/diagram regions
    # - Extract bounding boxes
    # - OCR text content
    
    items = await recursive_run_mineru_http(
        img_path,
        out_dir,
        port=8010,
        max_depth=2  # Recurse into sub-elements
    )
    
    # items = [
    #   {"type": "text", "text": "Encoder", "bbox": [10, 20, 100, 40]},
    #   {"type": "image", "img_path": "/sub_img_1.jpg", "bbox": [...]},
    #   ...
    # ]
    
    state.fig_mask = items
    return state
```

#### Step 6: Background Removal (Node 5)
```python
async def figure_icon_bg_remover_node(state):
    # Collect all images from layout detection
    image_paths = [item["img_path"] for item in state.fig_mask 
                   if item["type"] in ["image", "table"]]
    
    # Batch remove backgrounds
    output_paths = local_tool_for_bg_remove_batch({
        "image_path_list": image_paths,
        "model_path": state.request.bg_rm_model,
        "output_dir": state.result_path + "/icons"
    })
    
    # Update state with cleaned images
    for item, output_path in zip(state.fig_mask, output_paths):
        if item["type"] in ["image", "table"]:
            item["img_path"] = output_path
    
    return state
```

#### Step 7: Generate PPT (Node 6)
```python
async def figure_ppt_generation_node(state):
    prs = Presentation()
    
    # Slide 1: Reconstruct layout from mask data
    slide1 = prs.slides.add_slide(prs.slide_layouts[6])
    for element in state.fig_mask:
        if element["type"] == "text":
            add_text_element(slide1, element)
        elif element["type"] == "image":
            add_image_element(slide1, element)
    
    # Slide 2: Original generated image (full-screen)
    slide2 = prs.slides.add_slide(prs.slide_layouts[6])
    slide2.shapes.add_picture(state.fig_draft_path, ...)
    
    # Save PPT
    ppt_path = state.result_path / f"presentation_{timestamp}.pptx"
    prs.save(str(ppt_path))
    
    state.ppt_path = ppt_path
    return state
```

#### Step 8: Return Results
```python
# Workflow completes, FastAPI returns response
return Paper2FigureResponse(
    ppt_url="http://localhost:8000/outputs/paper2figure/20260108_abc123/output/presentation.pptx",
    draft_image_url="http://localhost:8000/outputs/.../figure_draft.jpg",
    status="success"
)
```

---

## Key Design Patterns & Learnings

### 1. **Agent-as-Tool Pattern**
Agents can be wrapped as tools for other agents to use, enabling hierarchical problem-solving.

```python
# Agent A can call Agent B as a tool
agent_b_tool = wrap_agent_as_tool(AgentB())
agent_a = AgentA(tools=[agent_b_tool])
```

### 2. **Pre-Tool / Post-Tool Pattern**
Inject context before LLM calls, process results after:

```python
@builder.pre_tool("paper_content", "paper_idea_extractor")
def _get_paper_content(state):
    # Executed BEFORE agent runs
    # Returns data that gets injected into prompt template
    return extract_pdf_text(state.paper_file)

@builder.post_tool("save_result", "figure_generator")
def _save_figure(state):
    # Executed AFTER agent runs
    # Process agent's output
    shutil.copy(state.fig_draft_path, state.result_path)
```

### 3. **Recursive Processing Pattern**
Handle nested structures (e.g., figures with sub-figures):

```python
async def recursive_run_mineru(img_path, out_dir, max_depth=2, current_depth=0):
    if current_depth > max_depth:
        return []
    
    # Process current image
    items = await run_mineru(img_path, out_dir)
    
    # Find sub-images
    sub_images = find_sub_images(out_dir)
    
    # Recursively process each sub-image
    for sub_img in sub_images:
        sub_items = await recursive_run_mineru(sub_img, out_dir, max_depth, current_depth + 1)
        items = merge_with_coordinate_transform(items, sub_items)
    
    return items
```

### 4. **State-Based Workflow Pattern**
State object flows through the pipeline, accumulating results:

```python
# State carries data + configuration
state.paper_file = "/path/to/paper.pdf"
state.request.api_key = "sk-..."

# Each node reads from and writes to state
state = await node1(state)  # Adds state.paper_idea
state = await node2(state)  # Adds state.fig_desc
state = await node3(state)  # Adds state.fig_draft_path
```

### 5. **Conditional Routing Pattern**
Dynamic workflow based on input type:

```python
def set_entry_node(state: Paper2FigureState) -> str:
    if state.input_type == "PDF":
        return "paper_idea_extractor"  # Start with text extraction
    elif state.input_type == "TEXT":
        return "figure_desc_generator"  # Skip extraction
    elif state.input_type == "FIGURE":
        return "figure_mask_generator"  # Skip generation
```

---

## Technology Stack Summary

| Layer | Technologies |
|-------|-------------|
| **Frontend** | React, TypeScript, Vite, TailwindCSS, Supabase |
| **Backend** | FastAPI, Python 3.11+, Pydantic, asyncio |
| **Workflow Engine** | LangGraph, LangChain, StateGraph |
| **LLM Integration** | OpenAI API, Claude, Custom endpoints |
| **Document Processing** | PyMuPDF (fitz), pdfplumber, PyPDF2 |
| **Layout Detection** | MinerU (Vision-Language Model) |
| **Image Processing** | Pillow, SAM (Segment Anything), Background Removal |
| **PPT Generation** | python-pptx |
| **Vector Graphics** | Inkscape (SVG processing) |
| **LaTeX Rendering** | Tectonic |

---

## Deployment Architecture

### Development Setup
```bash
# Backend
cd fastapi_app
uvicorn main:app --host 0.0.0.0 --port 8000

# Frontend
cd frontend-workflow
npm install && npm run dev  # Port 3000

# Model Services (optional, for high concurrency)
./script/start_model_servers.sh  # MinerU, SAM, OCR
```

### Production Considerations

1. **Load Balancing**: Use multiple MinerU/SAM instances on different GPUs
2. **Caching**: Cache LLM responses for common queries
3. **Queue Management**: Task semaphore prevents resource exhaustion
4. **Storage**: S3/MinIO for generated files
5. **Monitoring**: Log aggregation (ELK), metrics (Prometheus)

---

## What Makes This System Unique

### 1. **Modular Agent Architecture**
- Agents are self-contained, reusable units
- Easy to add new agents without modifying core
- Auto-registration eliminates boilerplate

### 2. **Declarative Workflows**
- Workflows defined as code, not configuration
- Easy to version control and test
- Conditional logic built-in

### 3. **Multi-Modal Processing**
- Seamlessly handles PDF, images, text
- Integrates multiple AI services (LLM + VLM + diffusion models)
- Preserves layout information

### 4. **Extensibility**
- CLI tool generates scaffold code: `dfa create --agent_name my_agent`
- Plugin architecture for tools/agents/workflows
- Prompt templates externalized (Jinja2)

### 5. **Production-Ready**
- Async/await throughout for high concurrency
- Proper error handling and logging
- Security boundaries (file access limited to project root)

---

## Next Steps

- Read [File Processing Workflow](./file-processing-workflow.md) for detailed PDF/upload handling
- Read [API Reference](./api-reference.md) for endpoint documentation
- Read [Developer Guide](./developer-guide.md) to create custom agents/workflows

