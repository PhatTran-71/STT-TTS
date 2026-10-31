# Kế hoạch thực nghiệm: trợ lý giọng nói local tiếng Việt cho HUIT

## 1. Định vị paper và đóng góp

### Đóng góp 1 — hệ thống local cho HUIT

Chúng tôi hiện thực một trợ lý giọng nói tiếng Việt tự chủ cho tra cứu thông tin HUIT: ASR, RAG trên tài liệu chính thức HUIT, LLM, TTS, browser client và cơ chế ngắt lời. Artifact công bố phải có cấu hình triển khai, revision model, scripts dựng index, manifest dữ liệu và log từng trial.

### Đóng góp 2 — đánh giá **đa model, đa benchmark**

Chúng tôi đánh giá nhiều lựa chọn model/cấu hình cho từng thành phần, trên benchmark công khai tiếng Việt và benchmark miền HUIT được xây dựng có kiểm soát. Mục tiêu là chỉ ra trade-off chất lượng, độ trễ

, bộ nhớ và độ tin cậy cho một hệ thống local; không tuyên bố thuật toán mới hoặc SOTA.

Đây không thể là “nhiều benchmark” nếu mỗi thành phần chỉ chạy đúng một model. Bài tối thiểu cần có ba mức so sánh:

1. **benchmark công khai cho từng năng lực nền:** VIVOS cho ASR, VMLU cho LLM text-only;
2. **benchmark miền HUIT:** retrieval, grounded QA và refusal trên tài liệu chính thức HUIT;
3. **benchmark hệ thống:** voice end-to-end và barge-in trên các truy vấn HUIT.

---

## 2. Câu hỏi nghiên cứu

| Mã | Câu hỏi | Bằng chứng |
|---|---|---|
| RQ1 | Model ASR nào đạt trade-off WER–RTF phù hợp cho hệ thống local? | VIVOS và tập audio HUIT nhỏ, 3 model ASR. |
| RQ2 | Retriever/reranker nào truy hồi đúng bằng chứng trong tài liệu HUIT với trễ và footprint chấp nhận được? | HUIT-RAGBench retrieval, ít nhất 3 embedding model và BM25. |
| RQ3 | Cấu hình RAG nào tạo câu trả lời đúng, có citation hỗ trợ, và biết từ chối câu ngoài corpus? | HUIT-RAGBench QA/refusal, nhiều retriever và ít nhất 2 LLM. |
| RQ4 | LLM local có năng lực tiếng Việt nền tảng ra sao ngoài miền HUIT? | VMLU, protocol chính thức, ít nhất 2 LLM. |
| RQ5 | TTS/chunking nào có trade-off tốt giữa audio-first latency, intelligibility và chất lượng nghe? | Ít nhất 2 TTS backend × 3 policy chunking. |
| RQ6 | Cấu hình tốt nhất của các component có thực sự giúp trải nghiệm hội thoại HUIT đầu-cuối? | Voice-E2E-HUIT: latency, success, citation, interruption. |

---

## 3. Cấu hình đã chốt

Mục tiêu phần cứng: một GPU NVIDIA 16 GB (Colab T4 là đủ), RAM 32 GB. Mọi thành phần đã dùng trong báo cáo là baseline chính, bắt buộc giữ trong bảng. Model mới chỉ là baseline bổ sung. Chạy các benchmark thành phần **từng model một**; end-to-end chỉ nạp cấu hình đã thắng ở development split.

| Thành phần | Model/cấu hình phải chạy | Thiết lập cố định |
|---|---|---|
| ASR | **Báo cáo:** `vinai/PhoWhisper-base`. **Bổ sung:** `openai/whisper-small`; `vinai/PhoWhisper-small` | FP16 CUDA; beam size 5; language `vi`; cùng text normalizer; không VAD trong VIVOS. |
| Sparse retrieval | **Báo cáo:** BM25 + SQLite FTS5 `unicode61`. **Bổ sung:** BM25L trên cùng index; BGE-M3 learned sparse vectors. | BM25/BM25L cùng tokenizer của báo cáo. BM25L: `k1=1.2`, `b=0.75`, `delta=0.5`. BGE-M3 sparse lấy token weights từ model. |
| Dense retrieval | **Báo cáo:** `intfloat/multilingual-e5-small`. **Bổ sung:** `intfloat/multilingual-e5-base`; `BAAI/bge-m3` | Cosine similarity; L2-normalize embeddings; cùng chunk corpus. E5 luôn dùng prefix `query:`/`passage:`. |
| Fusion | **Báo cáo:** BM25 + E5-small bằng RRF. **Bổ sung:** relative-score fusion BM25 + E5-small; BGE-M3 dense + sparse fusion. | RRF `k=60`; top-50 từ mỗi nhánh. Relative-score fusion min-max normalize từng danh sách top-50, rồi `0.5 × sparse + 0.5 × dense`. BGE-M3 cũng dùng trọng số dense/sparse 0.5/0.5. |
| Reranker | **Báo cáo:** `Alibaba-NLP/gte-multilingual-reranker-base`. **Bổ sung:** `BAAI/bge-reranker-v2-m3` | Rerank top-50 hybrid; batch size 8; max length 512; FP16 CUDA. |
| LLM | **Báo cáo:** `Qwen/Qwen3.5-4B` qua Ollama, dùng đúng digest/Modelfile đã ghi trong báo cáo. **Bổ sung:** `Qwen/Qwen3-4B-Instruct-2507` | Cùng Ollama/llama.cpp Q4_K_M; context 8,192; temperature 0 cho QA; max new tokens 256; tắt thinking mode nếu runtime hỗ trợ. |
| TTS | **Báo cáo:** `pnnbao-ump/VieNeu-TTS-v3-Turbo`. **Bổ sung:** `facebook/mms-tts-vie` | Một voice cố định/model; 24 kHz PCM mono; CPU inference, cùng text normalization. |
| Orchestration | S0: answer hoàn chỉnh rồi TTS; S1: token-stream + punctuation chunking; S2: S1 + interruption/epoch guard | Cùng ASR/retriever/reranker/LLM/TTS đã thắng development split. |

Không thay model vì kết quả test. Với Qwen3.5-4B, file manifest phải lưu digest, Modelfile, quantization và context window thực tế; không đủ bốn thông tin này thì không chạy benchmark cho đến khi khôi phục từ environment của báo cáo.

## 4. Benchmark đã chốt

| Benchmark | Mục đích | Split/runs cần báo cáo |
|---|---|---|
| VIVOS test | ASR tiếng Việt công khai | PhoWhisper-base của báo cáo + 2 ASR bổ sung; WER/CER/RTF/latency. |
| VMLU test | Năng lực LLM tiếng Việt ngoài miền HUIT | Qwen3.5-4B của báo cáo + Qwen3-4B-Instruct-2507; zero-shot, temperature 0. |
| HUIT-RAGBench retrieval | Chọn sparse, dense, fusion, reranker cho tài liệu HUIT | R0--R10 cố định dưới đây, trong đó R0/R3/R6/R9 giữ nguyên stack báo cáo; Recall/MRR/nDCG/latency. |
| HUIT-RAGBench QA/refusal | Đánh giá grounded answer và từ chối | Qwen3.5-4B + Qwen3-4B-Instruct-2507, mỗi model chạy 4 contexts. |
| HUIT-Audio | ASR trên câu hỏi miền HUIT | 120 audio clips, 3 ASR models. |
| HUIT-TTS | TTS và scheduler | 200 câu: 100 câu HUIT + 100 câu UIT-VSFC; 2 TTS × 3 policies. |
| Voice-E2E-HUIT | Trải nghiệm hệ thống | 100 scenarios, S0/S1/S2. |

Không dùng VLUE, ViLegalExpert hay benchmark pháp luật trong bài này: chúng không trả lời câu hỏi về RAG HUIT và sẽ làm loãng phần thực nghiệm.

---

## 5. HUIT-RAGBench: benchmark miền bắt buộc phải xây dựng

Đây là phần trung tâm của RAG. Corpus “question–answer về HUIT” tự nó **chưa đủ** làm benchmark, vì thiếu bằng chứng, split và câu không trả lời được.

### 4.1 Corpus

Chỉ lấy tài liệu HUIT được phép dùng: quy chế đào tạo, học phí, tuyển sinh, học vụ, lịch, biểu mẫu, website/FAQ chính thức và tài liệu đơn vị có ngày hiệu lực rõ. Mỗi document/chunk phải có:

```text
doc_id, source_url, title, issuing_unit, document_type,
published_or_effective_date, crawled_at, license_or_permission,
content_hash, version, section_path, chunk_id, text
```

- Lưu snapshot HTML/PDF hoặc checksum + script tải nếu được phép.
- Tài liệu có hiệu lực khác nhau phải giữ version/date; không trả lời một quy định đã hết hạn như hiện hành.
- Chunk theo tiêu đề/điều/mục. Nếu một mục dài hơn 350 syllable-token (tách theo whitespace), chia thành cửa sổ 350 với overlap 50; không chạy chunk-size ablation trong paper này.

### 4.2 Bộ câu hỏi

Tạo ba split độc lập theo **document source**, không split ngẫu nhiên theo câu hỏi:

| Split | Quy mô tối thiểu | Mục đích |
|---|---:|---|
| Development | 100 câu answerable + 50 unanswerable | Chọn chunk size, RRF, `top_k`, threshold, prompt. |
| Test retrieval/QA | 300 câu answerable | Báo cáo chính; mỗi câu có 1+ evidence chunk/document gold. |
| Test refusal | 150 câu unanswerable/out-of-scope/outdated | Đo refusal; không dùng khi tuning. |

Mỗi câu answerable gồm `question_id`, câu hỏi tự nhiên, intent/domain, `gold_answer` ngắn, `gold_evidence_ids`, ngày hiệu lực. Mỗi câu unanswerable phải có loại: ngoài phạm vi HUIT, không có trong corpus, mơ hồ/thiếu thông tin, hoặc đòi hỏi quyết định cá nhân. Hai annotator độc lập gán evidence; bất đồng được adjudicate. Công bố agreement và guideline.

**Chống leakage:** không sinh test question bằng cách chỉnh câu trong FAQ rồi để chính FAQ đó trong train/dev; không dùng test question trong prompt exemplars; ưu tiên paraphrase do người viết độc lập, sau đó một người khác tìm evidence gold.

### 4.3 Ba mức khó

Gán nhãn để phân tích lỗi:

1. **Single-hop:** một mục/FAQ trả lời trực tiếp.
2. **Multi-section:** phải kết hợp hai hoặc nhiều mục cùng văn bản.
3. **Temporal/procedural:** phụ thuộc thời điểm, quy trình hoặc điều kiện.

Báo cáo riêng từng mức; tránh để benchmark chỉ gồm FAQ keyword-match dễ.

---

## 6. Thí nghiệm cụ thể

### E0 — Manifest và tái lập

Mỗi trial ghi `run_id`, code commit, dữ liệu/split hash, model revisions, hardware, CUDA/driver, CPU/RAM, threads, quantization/offload, prompt hash, seed, cache/warm-up, status/error.

Dùng cùng máy chính; chạy component isolated và “all services resident” để thấy tranh chấp GPU/RAM. Warm-up 10 mẫu, không tính vào số liệu.

### E1 — ASR: VIVOS + HUIT-Audio

- **VIVOS test:** benchmark công khai, WER/CER/RTF với 3 model ASR.
- **HUIT-Audio:** 120 truy vấn HUIT, 20 người nói tiếng Việt, mỗi người đọc 6 truy vấn được phân công ngẫu nhiên. Thu 16 kHz mono WAV, ghi vùng miền/giới tính tự nguyện và không ghi định danh cá nhân. Đây là test miền, không gọi là benchmark chuẩn quốc gia.
- **Metrics:** WER, CER, RTF, endpoint-to-final transcript p50/p95, peak VRAM/RAM, lỗi/timeout. Bootstrap CI 95% theo utterance.
- **Fairness:** cùng audio, decode options, normalizer và language `vi`; chỉ model/revision thay đổi.

### E2 — Retrieval trên HUIT-RAGBench

Chạy toàn bộ 11 cấu hình dưới đây trên development để kiểm tra pipeline, sau đó chạy đúng một lần trên test; không thay cấu hình theo kết quả test.

| ID | Cấu hình |
|---|---|
| R0 | BM25 + SQLite FTS5 (**sparse trong báo cáo**) |
| R1 | BM25L trên cùng index |
| R2 | BGE-M3 learned sparse |
| R3 | E5-small dense (**dense trong báo cáo**) |
| R4 | E5-base dense |
| R5 | BGE-M3 dense |
| R6 | BM25 + E5-small + RRF (**hybrid trong báo cáo**) |
| R7 | BM25 + E5-small + relative-score fusion |
| R8 | BGE-M3 dense + BGE-M3 sparse fusion |
| R9 | R6 + `Alibaba-NLP/gte-multilingual-reranker-base` (**hybrid + reranker trong báo cáo**) |
| R10 | R8 + `BAAI/bge-reranker-v2-m3` |

**Metrics:** Recall@1/3/5/10, MRR@10, nDCG@10, Hit@5; p50/p95 latency, indexing time, index size, peak RAM/VRAM, QPS. Paired bootstrap CI 95% theo query. RRF constant là 60, candidate pool 50, reranker top-N là 50.

### E3 — Grounded QA và refusal HUIT

Dùng hai LLM × bốn RAG contexts: no-RAG, dense tốt nhất, hybrid tốt nhất, hybrid+reranker tốt nhất. Chạy trên test đã khóa. Prompt bắt buộc yêu cầu citation theo `chunk_id` và trả lời đúng câu ``Không tìm thấy thông tin trong tài liệu HUIT được cung cấp.'' khi evidence không đủ.

Mỗi output log câu trả lời, citation `chunk_id`, refusal flag/reason, retrieved contexts, token count, TTFT, total latency.

Hai annotator chấm mù 200 câu answerable test và toàn bộ/ít nhất 100 câu refusal test:

| Nhãn | 0 | 1 | 2 |
|---|---|---|---|
| Correctness | sai | đúng một phần | đúng, không thiếu ý quan trọng |
| Evidence support | không hỗ trợ | hỗ trợ một phần | evidence trực tiếp hỗ trợ claim chính |
| Temporal validity | sai thời điểm | không chắc | đúng version/ngày hiệu lực |

Báo answer score, evidence support, citation precision/recall, refusal precision/recall/F1, latency, failure rate và inter-annotator agreement. Không dùng LLM-as-judge làm metric chính.

### E4 — LLM tiếng Việt ngoài miền HUIT

Chạy VMLU evaluator chính thức cho hai LLM, zero-shot và decoding tất định. Báo accuracy overall/macro theo nhóm môn. Đây chỉ là năng lực nền của LLM, không trộn với RAG HUIT.

### E5 — TTS và scheduling

Chạy hai TTS backend đã chốt với đúng ba chính sách: **P0** chờ toàn bộ answer; **P1** flush tại `.` `?` `!` `;` `:`; **P2** streaming, chỉ flush ở dấu câu sau khi buffer có ít nhất 60 ký tự, ép flush ở 120 ký tự. Dùng 200 câu replay cố định: 100 câu HUIT hold-out và 100 câu UIT-VSFC. Thêm live mode với token stream từ cùng LLM.

Đo từ client: time-to-first-rendered-audio (TTFA), p50/p95, gap/underrun, RTF, total response time, round-trip ASR WER. Nghe mù 20–30 người: naturalness, continuity, intelligibility; report CI và paired analysis.

### E6 — Voice-E2E-HUIT và barge-in

100 kịch bản test: 25 FAQ/single-hop, 25 multi-section, 15 temporal/procedural, 15 refusal, 10 câu dài/TTS, 10 barge-in. So sánh S0/S1/S2 sau khi E1–E5 chọn ứng viên tốt. S0 dùng P0, S1 dùng P2, S2 dùng P2 và cancellation/epoch guard.

Client monotonic timestamps: user-end → first rendered audio; barge-in onset → detected → old audio stops; new user-end → new reply audio. Báo p50/p95, success/failure, stale audio leakage, history-consistency errors, citation completion và underrun. Lặp 5 lần trên mỗi timing barge-in.

---

## 7. Thiết kế bảng paper

1. Runtime/model manifest.
2. ASR: VIVOS và HUIT-Audio, ba model.
3. HUIT-RAGBench retrieval: sparse/dense/hybrid/reranker trade-off.
4. HUIT grounded QA/refusal: hai LLM × RAG contexts.
5. VMLU: hai LLM.
6. TTS: hai backend × policy.
7. Voice-E2E-HUIT: S0/S1/S2 và distribution latency/barge-in.
8. Error taxonomy: retrieval miss, outdated document, unsupported generation, incorrect refusal, ASR error, stale audio, timeout.

---

## 8. Trình tự làm việc

| Phase | Việc | Điều kiện xong |
|---|---|---|
| P0 | Chốt mục tiêu hardware, licenses, corpus HUIT và permission | Data manifest + document snapshots/chunks. |
| P1 | Xây HUIT-RAGBench, guideline, double annotation, document-level splits | Dev/test/refusal tách biệt, evidence gold hoàn chỉnh. |
| P2 | E1, E2 và chọn model bằng dev | Test chưa bị dùng để tuning. |
| P3 | E3, E4, E5 | Raw JSONL, CI và human-rating agreement. |
| P4 | E6 | Trace client, baselines S0–S2, failure rate. |
| P5 | Viết paper | Chỉ giữ claim có bảng và raw evidence. |

## 9. Điều kiện submit

Có code/artifact dựng index HUIT, corpus manifest và giấy phép/permission; split + qrels/evidence; ít nhất 2 LLM, 3 ASR, 3 dense retriever hoặc lý do hạn chế rõ; 2 TTS nếu muốn khái quát hoá TTS; end-to-end baseline serial/streaming; CI, failure rate và limitations. Nếu không phát hành được HUIT-RAGBench vì quyền dữ liệu, gọi nó là “internal HUIT evaluation set”, mô tả quy trình và không nhận đóng góp benchmark công khai.
