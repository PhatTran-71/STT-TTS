# E1 — ASR Benchmark: VIVOS + HUIT-Audio

## 1. Mô tả
Thí nghiệm đánh giá 3 model ASR tiếng Việt trên 2 tập dữ liệu:
- **HUIT-Audio:** 120 câu truy vấn âm thanh miền HUIT (16kHz Mono WAV, 16-bit PCM).
  - 30 câu người thật (đã đủ từ `huit_001.wav` đến `huit_030.wav`).
  - 90 câu tổng hợp từ `VieNeu-TTS-v3-Turbo` (từ `huit_031.wav` đến `huit_120.wav`).
- **VIVOS test:** Benchmark tiếng Việt công khai chuẩn (760 utterances).

## 2. 3 Model ASR Bắt Buộc
| Model | HuggingFace ID | Vai trò |
|---|---|---|
| PhoWhisper-base | `vinai/PhoWhisper-base` | Baseline báo cáo chính |
| Whisper-small | `openai/whisper-small` | Baseline bổ sung 1 |
| PhoWhisper-small | `vinai/PhoWhisper-small` | Baseline bổ sung 2 |

## 3. Fairness Protocol
- FP16 CUDA trên Google Colab GPU T4
- Beam size = 5, language = 'vi', task = 'transcribe'
- Dùng chung text normalizer tiếng Việt
- Warm-up 10 samples (không tính vào chỉ số đo)
- Bootstrap 95% Confidence Interval (1000 resamples)

## 4. Cấu trúc thư mục
```
Speech To Text/
├── README.md                             ← File mô tả
├── EXPERIMENT_PLAN_VI.md                 ← Kế hoạch thí nghiệm gốc
├── code/
│   ├── ASR_Benchmark_HUIT.ipynb          ← Notebook benchmark chính (chạy trên Colab)
│   ├── HUIT_Audio_Generator_VieNeu.ipynb ← Notebook sinh audio tổng hợp
│   └── TTS.ipynb                         ← Notebook TTS tham khảo
├── data/
│   ├── HUIT_Audio_120_Queries.xlsx       ← File metadata 120 câu
│   ├── HUIT_Audio_120_Queries.docx       ← Bản Word phân phối người đọc
│   ├── huit_audio_120.zip                ← File nén 120 WAV để upload Colab
│   └── huit_audio_120/                   ← Thư mục chứa đủ 120 file WAV 16kHz
└── results/                              ← Kết quả xuất ra từ Colab (CSV, JSONL)
```

