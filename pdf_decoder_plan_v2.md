# 📚 PDF Decoder — Study Companion App (Updated Work Plan v2)

> An AI-powered web app that takes any PDF, breaks it down into digestible study material with explanatory diagrams, simplified explanations, step-by-step numerical solving, and supplementary web info.

> [!IMPORTANT]
> **This is the updated plan** incorporating all handwritten change notes from the week-long review. Changes are marked with 🔄 throughout.

---

## 🎯 What Problem Does This Solve?

You upload a 6-page or 100+ page PDF (textbook chapters, notes, papers), and instead of losing focus reading dense text, the app:
- **Breaks it into topics** automatically
- **Explains each topic simply** (like a tutor talking to you)
- **Draws diagrams** that actually help you *understand* (process flows, cause-effect chains, comparison tables, step-by-step breakdowns)
- **Solves numericals step-by-step** with visual walkthroughs
- **Pulls extra info from the web** to fill gaps (via Tavily, not raw search)
- **Shows how much content was condensed** (e.g., "100 pages → 12 key topics")

---

## 🧩 Core Features

### 1. 📄 PDF Upload & Parsing
| Detail | Description |
|--------|-------------|
| **Input** | Drag & drop or click to upload any PDF (1–500+ pages) |
| **Parsing** | Extracts text, tables, equations, and images from the PDF |
| **Smart Chunking** | Splits content into logical topics/sections (not just by page) |
| **Page Stats** | Shows original page count, detected topics, and compression ratio |
| 🔄 **File Validation** | Validate uploaded files, limit file sizes, treat PDFs as untrusted input |
| 🔄 **OCR Fallback** | Detect pages with little/no extractable text → render to image → Tesseract.js OCR → only on pages that need it |

### 🔄 OCR Pipeline (NEW — Scanned PDF Support)

> [!NOTE]
> Tesseract.js is an **OCR fallback for scanned PDFs**, NOT something that runs on every PDF. For a mixed PDF (e.g., 100-page textbook where pages 1–80 are digital text and pages 81–100 are scanned handwritten notes), we detect which pages have little or no extractable text and OCR **only those pages**.

**Pipeline Steps:**
1. Extract text from each page using PDF.js
2. Detect pages with minimal extractable text (below a threshold)
3. Render those sparse pages to images using PDF.js canvas rendering
4. Pass each image to Tesseract.js for OCR
5. Collect recognized text and merge with digitally-extracted text
6. Feed the combined text into the chunking pipeline

**Limitations to be aware of:**
- Tesseract.js recognizes text in images but **doesn't reliably reconstruct** complex mathematical notation, circuit diagrams, or page layouts
- May need specialized processing for those types of content
- OCR can be CPU-intensive for long PDFs — use background Web Workers and track processing time
- Consider a progress indicator showing OCR status per page

### 2. 🧠 AI-Powered Topic Breakdown
| Detail | Description |
|--------|-------------|
| **Topic Detection** | Identifies distinct topics/concepts from the PDF |
| **Simple Explanations** | Rewrites each topic in plain, easy-to-understand language |
| **Key Takeaways** | Bullet-point summary of what you need to remember |
| **Difficulty Tagging** | Labels topics as Easy / Medium / Hard |
| **Estimated Read Time** | How long each simplified section takes to read |

### 3. 📊 Explanatory Diagrams (The Core Feature)

> [!IMPORTANT]
> These are NOT generic mind maps. These are **topic-specific, explanatory diagrams** that help you actually *understand* the material.

| Diagram Type | When It's Used | Example |
|--------------|----------------|---------|
| **Process Flowcharts** | Sequential processes, algorithms, procedures | "How does DNA replication work?" — step-by-step flow |
| **Cause-Effect Chains** | Understanding why things happen | "What causes inflation?" — chain diagram |
| **Comparison Tables** | Comparing 2+ concepts | "TCP vs UDP" — side-by-side visual |
| **Concept Relationship Diagrams** | How ideas connect to each other | "Newton's Laws" — how they relate |
| **Step-by-Step Problem Solving** | Numericals & calculations | "Solve this integral" — each step visualized |
| **Timeline / Sequence Diagrams** | Historical events, processes in order | "History of computing" — timeline |
| **Hierarchy / Tree Diagrams** | Classification, taxonomy | "Types of chemical bonds" — tree |
| **Annotated Illustrations** | Visual explanations with labels | Circuit diagrams, anatomy, etc. |

🔄 **Diagram Rendering Strategy (CHANGED):**

| Renderer | Used For | Rationale |
|----------|----------|-----------|
| **Mermaid.js** | Flowcharts, sequence diagrams, trees, cause-effect chains | Standard diagramming — reliable for structured diagrams |
| **Custom SVG generation** | Unusual/technical diagrams (circuits, anatomy, etc.) | Mermaid may not be enough for complex technical visuals |
| **Image Generation API** | Illustration visuals where code-generated diagrams don't suffice | For rich annotated illustrations |

> All three rendering approaches are presented inside the same interactive UI.

### 4. 🔢 Numerical / Problem Solver
| Detail | Description |
|--------|-------------|
| **Detection** | Auto-detects numerical problems in the PDF |
| **Step-by-Step Solution** | Shows every step with explanations |
| **Formula Highlighting** | Highlights which formula is used and why |
| **Visual Walkthrough** | Diagram showing the solution process |
| **Similar Problems** | Generates practice problems on the same concept |
| 🔄 **Deterministic Math** | For straightforward calculations (vector math, simple algebra), use a deterministic math library — don't rely on Gemini alone |
| 🔄 **Validation** | Validate numerical results; show uncertainty when the source doesn't support an answer |

### 5. 🌐 Web Enrichment 🔄 (REDESIGNED — Tavily-Powered)

> [!IMPORTANT]
> **Use Tavily as the web enrichment layer, NOT as the LLM tool.**
> Tavily finds the information → Gemini interprets it and makes it fit the student.
> This separation makes the architecture easier to maintain and test.

| Component | Description |
|-----------|-------------|
| **Tavily Search** | Find relevant external information for each topic |
| **Tavily Extract** | Retrieve additional content when search snippets aren't sufficient |
| **Gemini Interpretation** | AI interprets Tavily results and presents them in student-friendly language |
| **Source Citations** | Shows where the extra info came from |
| **Gap Filling** | If the PDF explanation is incomplete, Tavily + Gemini fills the gaps |
| **Efficiency** | Only fetches when the student requests it — keeps it lean |

### 6. 📉 Content Analytics Dashboard
| Metric | Description |
|--------|-------------|
| **Original Pages** | How many pages the PDF has |
| **Topics Found** | Number of distinct topics detected |
| **Compression Ratio** | e.g., "100 pages → 15 key topics (85% reduction)" |
| **Diagrams Generated** | Count of visual aids created |
| **Study Time Estimate** | Estimated time to go through simplified content |

---

## 🏗️ Technical Architecture

### 🔄 AI Orchestration Strategy (CRITICAL CHANGE)

> [!WARNING]
> **Do NOT ask Gemini to generate all four sections (topic breakdown, diagrams, numerical solving, web enrichment) in one request.**
> Each section has different requirements:
> - Topic breakdown needs **good explanations**
> - Diagram generation needs **valid Mermaid syntax**
> - Numerical solving needs **correct mathematics**
> - Web enrichment needs **actual search results from external sources**

**Solution:** Use separate task-specific prompts:
1. First prompt: Understand the document — get topic breakdown and structure
2. Separate prompts per feature: Invoke task-specific prompts/services only when the student **requests** a feature
3. This gives more control, avoids hallucinated errors in Mermaid, and avoids generating nonsense content the student doesn't need

🔄 **Multi-Orchestration Model — Task Routing:**

Use different AI "services" (prompt templates + model configurations) for different features. This saves free-tier token usage by routing each task to the most appropriate configuration:

| Task Route | Purpose | Model Config |
|-----------|---------|-------------|
| `model_calibrating_gemini` | Initial document understanding, topic detection, explanation generation | Gemini (lower temperature, focused on accuracy) |
| `diagramRepresentation_gemini` | Generating valid Mermaid syntax, SVG code, diagram structures | Gemini (structured output, strict syntax) |
| `webResearchApp_code` | Web enrichment orchestration — Tavily search + Gemini interpretation | Tavily API + Gemini (RAG pipeline) |

> Each route is a separate prompt template with its own system instructions, temperature settings, and output format requirements. This prevents one task from contaminating another and conserves free-tier tokens.

### Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Frontend** | HTML + CSS + Vanilla JS | Simple, fast, no build tools needed |
| **PDF Parsing** | pdf.js (Mozilla) | Best open-source PDF parser, runs in-browser |
| 🔄 **OCR Fallback** | Tesseract.js | Browser-based OCR for scanned PDF pages only |
| **AI/LLM (Primary)** | Google Gemini API | Free tier, great at explanations, generates diagram code |
| 🔄 **AI/LLM (Secondary)** | ChatGPT API (OpenAI) | Secondary LLM where testing shows a real benefit (Version 3) |
| **Diagrams** | Mermaid.js | Renders flowcharts, sequence diagrams, etc. from text |
| 🔄 **Technical Diagrams** | Custom SVG generation | For complex diagrams Mermaid can't handle |
| 🔄 **Illustration Visuals** | Image Generation API | For rich annotated illustrations |
| **Math Rendering** | KaTeX | Fast LaTeX math rendering for equations |
| 🔄 **Deterministic Math** | math.js or custom functions | For straightforward calculations |
| 🔄 **Web Search** | Tavily Search API + Tavily Extract | AI-enriched search |
| **Styling** | Custom CSS with dark mode | Premium study-friendly UI |
| **Markdown Rendering** | Marked.js + DOMPurify | Render AI-generated markdown safely |

### 🔄 Security Requirements (NEW)

| Requirement | Details |
|-------------|---------|
| **API Key Storage** | Store API keys on the server side (or secure input, never persist to localStorage in plaintext) |
| **File Validation** | Validate uploaded files — check MIME type, limit file sizes |
| **No Code Execution** | Never execute AI-generated code directly; treat all AI output as data |
| **Input Sanitization** | Treat PDFs and web content as untrusted input; sanitize with DOMPurify |
| **Page References** | Manage page references carefully; validate numerical results |
| **Uncertainty Display** | Show uncertainty when the source doesn't support an answer |

### 🔄 Performance & Cost Optimization (NEW)

| Strategy | Details |
|----------|---------|
| **Caching** | Avoid sending the same document to Gemini repeatedly; cache results |
| **Selective Retrieval** | Retrieve only relevant sections, not the whole document every time |
| **Lazy Processing** | Don't generate all content upfront — generate per feature on demand |
| **Background Workers** | Use Web Workers for OCR to avoid blocking the UI |
| **OCR Optimization** | Only OCR pages with minimal extractable text |
| **Avoid Long-Running** | Avoid long-running processing; show progress indicators |

---

## 📋 Build Phases — VERSIONED RELEASES

### Version 1 (Day 1): Static + PDF.js + Gemini + Mermaid.js

#### Phase 1.1: Core Foundation
- [ ] Project setup (HTML/CSS/JS structure)
- [ ] PDF upload & parsing with pdf.js
- [ ] Text extraction and display
- [ ] File validation (MIME type check, size limits)
- [ ] Dark mode UI with sidebar layout

#### Phase 1.2: AI Integration (Gemini Only)
- [ ] Gemini API integration (API key input with secure handling)
- [ ] Topic detection & chunking (Prompt 1: understand the document)
- [ ] Simple explanations generation (separate prompt per topic)
- [ ] Content analytics (page count, compression ratio)

#### Phase 1.3: Diagrams & Visuals (Mermaid Only)
- [ ] Mermaid.js integration
- [ ] AI-generated diagram code for each topic (separate prompt)
- [ ] Diagram rendering and display
- [ ] Multiple diagram types support

#### Phase 1.4: Basic Numerical Solver
- [ ] Problem detection in PDF text
- [ ] Step-by-step solution generation (separate prompt)
- [ ] KaTeX math rendering
- [ ] Visual solution walkthroughs

#### Phase 1.5: Polish v1
- [ ] Responsive design
- [ ] Loading states and error handling
- [ ] Progress tracking
- [ ] Dark mode toggle

### Version 2 (Day 2): Numerical Validation + Web Search + Persistent Storage

#### Phase 2.1: OCR Fallback
- [ ] Detect pages with minimal extractable text
- [ ] Render sparse pages to images via PDF.js
- [ ] Tesseract.js integration (Web Worker)
- [ ] Merge OCR text with digital text
- [ ] Progress indicator for OCR processing

#### Phase 2.2: Numerical Validation
- [ ] Integrate math.js for deterministic calculations
- [ ] Validate Gemini's numerical outputs against math.js
- [ ] Show uncertainty indicators
- [ ] Formula highlighting and step verification

#### Phase 2.3: Web Enrichment (Tavily)
- [ ] Tavily Search API integration
- [ ] Tavily Extract for deeper content retrieval
- [ ] Gemini interprets Tavily results
- [ ] Source citations and gap filling
- [ ] On-demand fetching

#### Phase 2.4: Persistent Storage & Caching
- [ ] Cache Gemini responses
- [ ] Store processed documents for quick reload
- [ ] IndexedDB or localStorage for persistence

#### Phase 2.5: SVG Diagram Support
- [ ] Custom SVG generation for technical diagrams
- [ ] SVG rendering within the same UI as Mermaid

#### Phase 2.6: Polish v2
- [ ] Animations and transitions
- [ ] Export/print functionality
- [ ] Error recovery and retry logic

### Version 3 (Day 3+): Fine AI Orchestration + Secondary LLM

#### Phase 3.1: Multi-LLM Orchestration
- [ ] ChatGPT API integration as secondary LLM
- [ ] A/B test which LLM produces better results per feature
- [ ] Route specific tasks to the best-performing LLM
- [ ] Cost optimization

#### Phase 3.2: Image Generation for Illustrations
- [ ] Integrate image generation API
- [ ] Present generated images in the same UI

#### Phase 3.3: Advanced Features
- [ ] Similar problem generation for practice
- [ ] Study session tracking and progress analytics
- [ ] Spaced repetition suggestions

---

## ⚙️ What You'll Need

| Requirement | Details |
|-------------|---------|
| **Gemini API Key** | Free at aistudio.google.com — 1,500 requests/day |
| **ChatGPT API Key** | For Version 3 — platform.openai.com (paid) |
| **Tavily API Key** | For web enrichment — tavily.com (1,000 free credits/month) |
| **Browser** | Any modern browser (Chrome, Edge, Firefox) |
| **Internet** | For AI calls and web enrichment |
