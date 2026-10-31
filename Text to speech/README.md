# E5 - TTS Benchmark & Scheduling: VieNeu vs. MMS-TTS

## 1. Mô tả
Thực nghiệm đánh giá và so sánh hiệu năng của 2 mô hình Text-to-Speech (TTS) tiếng Việt kết hợp 3 chiến lược lập lịch đệm (chunking/scheduling policies) trên tập dữ liệu chuẩn 200 câu.

## 2. Mô hình TTS (Backends)
| Backend | Model ID / Engine | Thiết lập |
|---|---|---|
| **VieNeu-v3-Turbo** (Đề xuất chính) | pnnbao-ump/VieNeu-TTS-v3-Turbo | Voice 'Mai Anh', ONNX CUDA/CPU, 24kHz |
| **MMS-TTS-VIE** (Baseline so sánh) | acebook/mms-tts-vie | VITS architecture, 24kHz |

## 3. Chính sách lập lịch (Scheduling Policies)
- **P0 (Full-Sentence)**: Chờ hoàn thành toàn bộ câu văn bản mới đưa vào TTS sinh âm thanh.
- **P1 (Punctuation Flush)**: Gửi sang TTS ngay khi gặp các dấu ngắt câu (., ?, !, ;, :).
- **P2 (Streaming Buffer)**: Gom token vào buffer; chỉ flush tại dấu câu khi buffer >= 60 ký tự; ép flush khi buffer chạm 120 ký tự.

## 4. Tập dữ liệu kiểm thử (200 câu)
- **100 câu HUIT** (data/benchmark_200.json): Dữ liệu miền nghiệp vụ đại học (học phí, quy chế đào tạo, chuẩn đầu ra, tuyển sinh, ký túc xá...).
- **100 câu UIT-VSFC** (data/benchmark_200.json): Dữ liệu phản hồi sinh viên tự nhiên (ngôn ngữ đời thường, cảm xúc).

## 5. Cấu trúc thư mục
`
Text to speech/
├── README.md                            # Tài liệu tóm tắt thực nghiệm
│
├── code/
│   ├── E5_Notebook1_Prepare_Data.ipynb  # Chuẩn bị và làm sạch 200 câu test
│   └── E5_Notebook2_Run_Benchmark.ipynb # Chạy benchmark 2 backend x 3 policy x 200 câu
│
├── data/
│   ├── benchmark_200.json               # Bộ dữ liệu 200 câu (định dạng JSON)
│   ├── benchmark_200.csv                # Bộ dữ liệu 200 câu (định dạng CSV)
│   └── audio_samples/                   # Mẫu âm thanh WAV (24kHz, PCM 16-bit)
│
└── results/
    ├── table6_e5.csv                    # Bảng 6 kết quả tổng hợp (Metrics)
    ├── e5_chart.png                     # Biểu đồ so sánh trực quan TTFA, RTF, Underrun
    ├── e5_full_results.csv              # Dữ liệu kết quả chi tiết từng trial
    └── e5_raw_results.jsonl             # Log thô chi tiết 1.200 trials (chuẩn E0)
`

## 6. Kết quả thực nghiệm (Bảng 6)
| Backend | Policy | N | TTFA p50 (s) | TTFA p95 (s) | TTFA CI95 | RTF p50 | Underrun (%) | Total lat p50 (s) |
|---|---|---|---|---|---|---|---|---|
| **VieNeu-v3-Turbo** | **P0** | 200 | **0.921** | 1.244 | [0.864, 0.949] | **0.143** | **0.0** | **0.921** |
| **VieNeu-v3-Turbo** | **P1** | 200 | **0.917** | 1.221 | [0.852, 0.946] | **0.143** | **0.0** | **0.917** |
| VieNeu-v3-Turbo | P2 | 200 | 4.236 | 5.396 | [4.148, 4.443] | 0.795 | 28.0 | 4.444 |
| MMS-TTS-VIE | P0 | 200 | 4.358 | 6.508 | [4.012, 4.464] | 0.516 | 0.0 | 4.358 |
| MMS-TTS-VIE | P1 | 200 | 4.021 | 6.334 | [3.797, 4.146] | 0.477 | 0.0 | 4.021 |
| MMS-TTS-VIE | P2 | 199 | 3.789 | 4.931 | [3.613, 3.936] | 0.480 | 27.6 | 3.957 |

## 7. Kết luận chính
1. **Model**: VieNeu-v3-Turbo vượt trội hoàn toàn so với MMS-TTS-VIE: tốc độ nhanh hơn **3.6 lần** (RTF 0.143 so với 0.516) và thời gian phản hồi âm thanh đầu tiên nhanh hơn **4.7 lần** (0.92s so với 4.35s).
2. **Policy tối ưu**: Với mô hình nhẹ như VieNeu, chính sách **P0/P1** đạt hiệu năng lý tưởng nhất (TTFA < 1s, Underrun = 0.0%), đảm bảo âm thanh liên tục không gián đoạn cho trợ lý giọng nói.
