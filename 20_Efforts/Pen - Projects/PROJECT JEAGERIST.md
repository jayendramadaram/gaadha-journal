

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
1. A 