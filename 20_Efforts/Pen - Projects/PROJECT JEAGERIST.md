

Best Suite of OCR text recog and extraction
- https://github.com/PaddlePaddle/PaddleOCR (outperforms gemeni, qwen and chatgpt-4o in doc parsing)
	- 2 million params
	- https://www.paddleocr.ai/main/en/index.html
	![[Pasted image 20260215143436.png]]

Best Document parser
- https://github.com/docling-project/docling
- has quite a good pipeline in place
- native integrations into LangChain, LlamaIndex, and CrewAI


Choice for local LLM's in the pipeline 
	- reasoning document privacy
	- could be runnable on a 4090
	- No model over-picking


Bench marking platforms for parsers
- OmniDocBench


- olmOCR-2 for complex scans and fallback
- 10k pages or more


# Layout Analysis
	- https://github.com/opendatalab/DocLayout-YOLO
	- surpasses LayoutLMV3 and yolo


For table structure recognition specifically, the stack should combine multiple approaches:
- Granite-Docling-258M
- **TableTransformer v1.1** 
- **SPRINT**
- PubTables-v2

--------

Perfect pipeline

Initial phase - OCR
1. Scanned PDF's (PaddleOCR - GPU powered, excells in all benchmarks)
2. Born-Digital PDF - DOCLING (fast as fuck)
3. Fall back for degraded documents - ( Hard approach but VLM fallback )
	1. Qwen2.5-VL can do things better, but there is a chance of halucination
	2. there are far far far better models that can give 100% results but they are overkill
		1. https://arena.ai/leaderboard/vision
		2. might want to choose cleverly because these need H100's to run easily
		3. http://livebench.ai/#/?openweight=true
4. a document router would decide which intial pipeline should be triggered

( Next phase is only triggered only when this face passes with extraction metrics of CER - claim-evidence-Reasoning < 2 % ) - zero numerical errors on your test set


Phase 2 (Layout analysis)
- table structure recognition - knowing which cell is which in a Schedule III Balance Sheet
- section segmentation - building the document's hierarchy tree


phase 3 (Extraction)
- transforms OCR text + layout structure into clean, machine-readable financial JSON.
- Indian number parsing (hazar, carode, das, pachees)
- hierarchical financial statement extraction using LLM prompts
- XBRL cross-validation - eXtensible Business Reporting Language


Phase 4 (RAG Knowledge Base)
- chunking of IndAS standards
- Section-aware chunking of IndAS standards
- fine-tuned BGE-M3 embeddings ?????????
- Qdrant vector store with payload filtering, hybrid BM25+dense search
- Neo4j knowledge graph for cross-regulatory reasoning


Phase 5 (Compliance Reasoning)
- A clever model likw qwen 32B or Kimi k2.5
- Financial Data + retrieved regulatory context
- compliance judgements
	- multi step retrival + reasoning via langraph
	

Phase 6 (Integration + test)



Hardware requirements - development
- All other models are pretty cheap and easy peasy to run on regular hardware (8 gm ram, 200 gb storage, no gpu)
	- but phase 4 , 5 are critical
	- they need atleast 1 min and 2 max 4090's [ I have one so should be fine for experimenting]
	- runs easily - 7B VLM and 32B reasoning model
- 64 GM ram - RAG fast memory for LLM's
- 500 GB - just in case for data set 


Production scaling it to users
- 6× NVIDIA H100 80GB [ or any equivalents ] - 8× RTX 5090 32GB
- lot of RAM, 1 TB 
- lot of storage to keep storing data - 3 TB
- [ this is in assumption for scaling not inital requirement ]



Data collection
Annual
- mca.gov.in - ₹100/company
- bseindia.com
- nseindia.com
- https://www.kaggle.com/datasets/sameerprogrammer/detailed-financial-data-of-4456-nse-and-bse-company

Regulatory texts
- IndAS standards - 41 standards
	- icai.org
- - SEBI LODR regulations: sebi.gov.in
- - Companies Act 2013: legislative.gov.in
- - MCA notifications
- - - RBI guidelines for NBFC/banking companies: rbi.org.in
- XBRL filings
	- - MCA XBRL portal: Download IndAS taxonomy files

initial pipeline development
- 10 annual reports
	- 5 scanned, 5 digital

| Note: For fine tuning VLM models we would eed like quite a lot of anotated images - expected 30k page images

Fine tuning is not possible in my 4090's

![[Pasted image 20260215171233.png]]


30 - 40 hours per single fine tune run - just on LORA, not full fine tune (QLoRA)
	- atleast 5 - 10 runs for good trail and error best results -LlamaFactory/Unsloth
	- Freeze the vision encoder (ViT) and tune the projector + LLM layers with LoRA adapters. This keeps VRAM under ~20-22GB.
	- QLoRA (4-bit quantized LoRA) or LoRA.


Critical problems
- Text extraction - - from multi-format documents (PDF, scanned images, XBRL)
- Table extraction - 
- Hyperlink extraction - URLs, cross-references to notes/schedules, comments
- Section segmentation - - based on headers (Balance Sheet, P&L, Cash Flow, Notes)
- Metadata tagging - company name, CIN, financial year, auditor, etc.
- Compliance validation - - against IndAS, Schedule III, SEBI, RBI frameworks
- Structured output - in a defined JSON/XML schema



Programming Document Router
- try python extractor for 50 characts if success then pass it to born - digital pdf pipeline
- else if less than 10 - go for scanned
- else if more than 50 char and has images then mixed


OCR
- PaddleOCR
	- 106 languages including Devanagari/Telugu/Tamil
	- CPU inference
	- 2M parameters — tiny and fast
	- reports confidence as well if confidence is < 5%
	- go for VLM based fall back
		- If average confidence drops below 0.80 or more than 20% of text blocks are low-confidence, re-process the page with olmOCR-2 or LightOnOCR-2. This catches degraded scans, handwritten annotations, and complex stamp overlays that PaddleOCR struggles with.

validation and verification
- Download SROIE test set - *Character Error Rate**
- compute CER
- **FinTabNet**
	- Table extraction accuracy
	- TEDS > 0.95.
- Manual verification, once in a while, [aggresive in development, once in a while in production]
- If CER > 3% on any document type, or if numerical extraction has ANY errors in your spot-check. Go back to the VLM fallback threshold tuning and the document router classification heuristics.
- proceed - CER < 2% on benchmarks AND 100% numerical accuracy


Layout Detection
-  DocLayout-YOLO 
	- Outperforms LayoutLMv3 and DiT on diverse document
- Table Structure Recognition
	- most important and could be our USP
	- Granite-Docling-258M as the primary engine (best accuracy on financial tables), with TableTransformer as validation.
	- Section Segmentation
		- PURE DSA


verification and validation
- **Layout detection mAP** — target > 0.75 on DocLayNet-style evaluation. Run detector on the DocLayNet test set.
- **Table structure TEDS** — target > 0.95 on FinTabNet.
- manually annotate 10 annual reports with correct section boundaries. Compute precision/recall
- Amount unit detection accuracy - human work - 100% accuracy


If anything has issues, we can increase context and re run this pipeline, and we can use VLM assisted guided learning


Phase 3: Structured Data Extraction - transforms messy OCR output into machine-readable financial data
- PROVIDE ALL FUNCTION CALLING [important]
- Indian Number Format Parsing
- Financial Statement Extraction
	- USE LLM's here, 
	- there would be meta components like
		- numbers, amounts which can be staticly checked against OCR outputs and we can cross verify LLM accuracy
		- multiple runs is best
		- file write pattern, and verision controll updates
- Metadata Extraction
- XBRL Processing with Arelle
	- - All companies with paid-up capital ≥₹5 crore or turnover ≥₹100 crore MUST file in XBRL format with MCA - XBRL filings are STRUCTURED DATA 
	- no OCR needed, exact values guaranteed 
	- If a company has both a PDF annual report AND XBRL filing, we can cross-validate: extract from PDF, compare with XBRL, flag discrepancies


# RAG & Compliance Knowledge Base
- important thing - SECTION-AWARE chunking that preserves the regulatory hierarchy.
	- Parse the document into its natural section hierarchy
	- Each paragraph becomes one chunk
	- Attach rich metadata for filtering during retrieval
	- Generate contextual prefix

- Embedding with BGE-M3
	- 568M parameters, 1024 dimensions good balance of quality and speed
	- Supports dense, sparse, AND multi-vector retrieval in one model

We need to fine tune these embeddings at any cost, general purpose embeddings top generic benchmarks but perform flawed in real world compliance and legal embeddings

so fine tuning is must (Open to explore any other model than BGE-M3)

fine tune plan
1. Collect regulatory query-passage pairs (e.g., "What are the disclosure requirements for contingent liabilities?"  IndAS 37, Para 86) 
2. Use contrastive learning to fine-tune BGE-M3 on these pairs
	1. self supervised type shit, learns from its own data
3. Expected improvement: +10-30% retrieval accuracy on regulatory queries
4. After fine tune encode all the chunks


Qdrant 
- Payload filtering
- Real-time updates
- When checking IndAS 24 compliance, we dont want to search across ALL regulations
	- filter to IndAS 24 chunks first, then do vector similarity within that subset.


Hybrid Search -  (BM25 + Dense)
- BM25 - exact keyword search
- Dense - in depth search for generic questions like : what are the rules for recognizing revenue from long-term contracts?
- compute Reciprocal Rank Fusion


Knowledge Graph for Cross-Regulatory Reasoning
- get results + reasoning from multiple compliance dbs
-

Phase 5: Reasoning 

GOALS
1. Understand regulatory language precisely (not approximately)
2. Apply rules to specific financial data (quantitative reasoning)
3. Handle edge cases and exemptions (multi-hop logic)
4. Explain its reasoning in auditable detail (not a black box)

checks
- presense checks
- completeness checks
- consistensy checks
- threshold checks
- format checks
- cross refernce checks


[ CHAIN THE RULE SET HARD AND THIS GOES INTO SYSTEM PROMPT ]


Compliance Reasoning Agent
- Retrieves relevant regulatory text from the knowledge base
- Retrieves relevant sections from the target annual report
- Reasons about whether the requirements are met
- Generates a structured compliance verdict with evidence


[ HUGE SYSTEM PROMPT AGAIN with compating ]



### Multi-Agent Orchestration with LangGraph

we have couple of models
- ocr
- layout
- VLM
- extraction
- compliance
- validation


langraph
- state machine with cycles
- Built in persistence
- tool calling
- Document → OCR → Layout → Extraction → Compliance → Report
- handles ERROR RECOVERY
	- OCR confidence is low → retry with VLM fallback
	- re run the flows



all models
- PaddleOCR PP-OCRv5 - 2M params
- Docling - 258M
- olmOCR-2 / LightOnOCR-2 - 1B

- DocLayout-YOLO - ~100M
- Granite-Docling-258M - 258M

- Qwen2.5-VL-7B (QLoRA) - 7B

- BGE-M3 (fine-tuned) - 568M

- Qwen3-32B-AWQ - 32B (4-bit)

- Reasoning DeepSeek-R1-Distill-32B 32B



Other options
TABLES
- TableTransformer v1.1
- **SPRINT**
- PubTables-v2
- **MultiDocFusion** (EMNLP 2025) constructs hierarchical trees from section headers using LLMs, then applies DFS-based grouping for coherent section chunks.
- **UniHDSA** handles cross-page table grouping and TOC extraction at the document level.




if using chinese open sourced models is prohiobited 
- llama 3 or mistral
- LLaVA or Pixtral - for vision
- we can start with chinese models, since model architectures are same, we can swap them in and out



DEEP REASONING 
-  `PyMuPDF` or `pdfplumber` to extract URI annotations and link them to the text.
- for scanned docs, hyperlink detectin is not possible haha


Summarisation Agent
- as part of reasoning agent



Deepresearch agent
- - Add a "External Signal Ingestion" module.
- _Tech:_ A simple scraper (BeautifulSoup/Selenium) that feeds into the RAG pipeline.
- news, enforcement actions... and whistle-blower allegations