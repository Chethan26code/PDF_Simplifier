# PDF Decoder — Development Rules & Architectural Guardrails

## AI Orchestration
- Use separate, task-specific prompts for each feature (topic breakdown, diagram generation, numerical solving, web enrichment). Never combine them into a single LLM request.
- Each prompt must be optimized for its specific output type:
  - `model_calibrating_gemini`: Document comprehension & clean explanations.
  - `diagramRepresentation_gemini`: Strictly valid Mermaid/SVG syntax.
  - `webResearchApp_code`: Synthesis of Tavily search findings.
- Generate features on-demand (when requested by the student), rather than all upfront.

## OCR Processing
- Treat OCR (Tesseract / pytesseract) as an on-demand fallback for scanned pages only.
- Detect pages with sparse or empty extractable text using character-count thresholds before invoking OCR.
- Never run blanket OCR over an entire document if digital text is extractable.

## Web Enrichment (Tavily + Gemini)
- Tavily's role is to FIND external factual information.
- Gemini's role is to INTERPRET and contextualize it for the student.
- Never use the search API directly as the explanation engine.

## Math & Numerical Validation
- Do not rely on LLM token completion alone for numerical math, matrix arithmetic, or complex formulas.
- Validate steps and outputs using deterministic computation (SymPy on backend / math.js on frontend).
- Display explicit uncertainty indicators if the source document or validation cannot confirm an answer.

## Security & Sanitization
- All API keys must remain strictly on the backend (`.env`), never exposed to frontend client code.
- Sanitize all rendered Markdown/HTML with DOMPurify to prevent XSS.
- Never execute AI-generated code directly. Treat all external inputs (PDFs, web snippets) as untrusted.

## Performance & Caching
- Cache processed topics, diagram syntax, and search responses to conserve API quota and avoid redundant model calls.
- Support lazy on-demand retrieval for resource-intensive features.

