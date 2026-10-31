# STT-TTS Benchmark & Evaluation for HUIT Voice Assistant

Kho lưu trữ mã nguồn, tập dữ liệu kiểm thử và kết quả thực nghiệm cho hệ thống trợ lý giọng nói tiếng Việt tại HUIT.

## Cấu trúc Repository

### 1. [Speech To Text](./Speech%20To%20Text) (Thực nghiệm E1)
- Đánh giá 3 mô hình ASR: inai/PhoWhisper-base, inai/PhoWhisper-small, openai/whisper-small.
- Tập dữ liệu: VIVOS test và HUIT-Audio (120 câu hỏi miền HUIT).
- Chỉ số: WER (%), CER (%), RTF, Latency p50/p95, Peak VRAM.

### 2. [Text to speech](./Text%20to%20speech) (Thực nghiệm E5)
- Đánh giá 2 backend TTS: VieNeu-v3-Turbo vs MMS-TTS-VIE.
- 3 chính sách lập lịch phát âm: P0 (Full-sentence), P1 (Punctuation flush), P2 (Streaming buffer 60-120 chars).
- Tập dữ liệu: 200 câu (100 câu HUIT nghiệp vụ + 100 câu UIT-VSFC).
- Chỉ số: TTFA p50/p95 (Time-To-First-Audio), RTF, Underrun rate (%), Total latency.
