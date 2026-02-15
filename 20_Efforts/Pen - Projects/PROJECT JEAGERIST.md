

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
	- they need atleast 1 min and 2 max 4090's [ I have one so should be fine for experimenting ]
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