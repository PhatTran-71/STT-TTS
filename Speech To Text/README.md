# E1 — ASR Benchmark: VIVOS + HUIT-Audio

## Mô tả
Thí nghiệm đánh giá 3 model ASR tiếng Việt trên 2 tập dữ liệu:
- **VIVOS test** (benchmark công khai, ~760 utterances)
- **HUIT-Audio** (120 câu hỏi miền HUIT, 16kHz Mono WAV)

## 3 Model ASR
| Model | HuggingFace ID |
|---|---|
| PhoWhisper-base (Baseline) | `vinai/PhoWhisper-base` |
| Whisper-small | `openai/whisper-small` |
| PhoWhisper-small | `vinai/PhoWhisper-small` |

## Fairness Protocol
- FP16 CUDA trên Google Colab T4
- Beam size = 5, language = 'vi'
- Cùng text normalizer cho tất cả model
- Warm-up 10 samples, không tính vào kết quả
- Không dùng VAD trên VIVOS

## Metrics
- WER (%), CER (%) với Bootstrap 95% CI (1000 resamples)
- RTF (Real-Time Factor)
- p50/p95 Latency (ms)
- Peak VRAM (MB)

## Cấu trúc thư mục
```
Speech To Text/
├── README.md                             ← File này
├── EXPERIMENT_PLAN_VI.md                 ← Kế hoạch thí nghiệm đầy đủ
│
├── code/
│   ├── ASR_Benchmark_HUIT.ipynb          ← Notebook benchmark (chạy trên Colab)
│   ├── HUIT_Audio_Generator_VieNeu.ipynb ← Notebook tạo audio TTS
│   └── TTS.ipynb                         ← Notebook TTS gốc (tham khảo)
│
├── data/
│   ├── HUIT_Audio_120_Queries.xlsx       ← Metadata 120 câu (transcript, speaker, topic...)
│   ├── HUIT_Audio_120_Queries.docx       ← Bản Word phân phối thu âm
│   └── huit_audio_120/                   ← 116 file WAV (26 human + 90 VieNeu TTS)
│
└── results/                              ← Kết quả benchmark (sau khi chạy xong trên Colab)
    ├── huit_audio_results.csv
    ├── vivos_test_results.csv
    └── summary_table.csv
```

## Cách chạy lại
1. Upload `ASR_Benchmark_HUIT.ipynb` lên Google Colab
2. Chọn Runtime > T4 GPU
3. Upload thư mục `huit_audio_120/` (hoặc zip) lên Colab
4. Chạy lần lượt Cell 1 → Cell 6
5. Kết quả tự động tải về dạng ZIP

## Dataset HUIT-Audio
- 120 câu hỏi tiếng Việt về HUIT (tuyển sinh, học phí, quy chế, đặc thù HUIT)
- 26 file thu từ người thật + 90 file tạo bởi VieNeu-TTS-v3-Turbo
- Thiếu 4 file (huit_027 → huit_030): bạn thu âm chỉ gửi 26/30 file
- Format: WAV, 16kHz, Mono, PCM 16-bit

