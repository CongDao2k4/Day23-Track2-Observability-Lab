# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Đào Văn Công

**Mã học viên**: 2A202600031

**Submission date:** 2026-05-11

**Lab repo URL:** https://github.com/CongDao2k4/Day23-Track2-Observability-Lab.git

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.2)
Compose v2:    OK  (5.1.3)
RAM available: 7.61 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /mnt/d/AI - AI_VinUni_St2_Lab/Day23-Track2-Observability-Lab/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Tiện lợi và sức mạnh của **PromQL** với **Rate functions**. Tôi chỉ với một dòng query đơn giản, Prometheus có thể tính toán chính xác tốc độ lỗi (Error Rate) và lưu lượng (Throughput) từ các Counter thô. Ngoài ra, tính năng **Exemplars** (nút màu xanh trên biểu đồ Latency) cho phép tôi click trực tiếp từ một điểm dữ liệu bị chậm sang ngay Trace tương ứng trong Jaeger, giúp rút ngắn thời gian điều tra lỗi từ vài phút xuống còn vài giây.

_(2-3 sentences)_

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste one structured log line with `trace_id` (from Loki or `docker compose logs app`):

```json
{"model": "llama3-mock", "input_tokens": 8, "output_tokens": 8, "quality": 0.855, "duration_seconds": 0.1645, "trace_id": "bd991f39ef3f9e1a0ad653aea339c7c6", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T11:30:41.328364Z"}
```

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

**Calculation:**
- Keep 100% of Errors (assume 5% error rate)
- Keep 100% of Slow (assume 2% latency spikes > 2s)
- Keep 1% of Healthy (93% of total)

Total fraction = `(1.0 * 0.05) + (1.0 * 0.02) + (0.01 * 0.93)` = `0.05 + 0.02 + 0.0093` = **7.93%** of total traffic.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "feature_gpu_util": {"drift": "yes", "score": 0.32, "method": "PSI"},
  "feature_latency": {"drift": "no", "score": 0.05, "method": "KS"}
}
```

### Why use labels like `model` or `status`?

Dùng labels cho phép chúng ta thực hiện **Aggregation (Gom nhóm)** và **Filtering (Lọc)** dữ liệu mà không cần tạo ra hàng trăm metric riêng lẻ. 

Ví dụ: thay vì tạo `latency_llama3` và `latency_gpt4`, chúng ta chỉ cần một metric `inference_latency` với label `model`. Điều này giúp Dashboard trở nên linh hoạt (Dynamic Dashboard), người dùng có thể đổi model qua dropdown menu mà không cần sửa code Dashboard.

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

- `prompt_length`: **KS Test** Tốt nhất cho dữ liệu phân phối liên tục và phát hiện sự thay đổi hình dạng phân phối.

- `embedding_norm`: **MMD** Vì embedding là dữ liệu vector cao chiều, MMD hiệu quả hơn trong không gian feature phức tạp.

- `response_length`: **KS Test** Giống như prompt length, nhạy với các thay đổi nhỏ trong phân phối số lượng tokens.

- `response_quality`: **PSI** Chất lượng thường được băm thành các nhóm/bins ổn định, PSI giúp theo dõi sự dịch chuyển giữa các nhóm này.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Metric từ **Day 18 (Lakehouse/Spark)** là khó nhất. Vì Spark có cấu trúc metrics rất phức tạp và cần cấu hình Prometheus Servlet hoặc JMX Exporter riêng biệt, khác với các ứng dụng Python đơn giản chỉ cần một endpoint `/metrics`.

---

## 6. The single change that mattered most

Thay đổi quan trọng nhất là việc cấu hình lại **Port-binding (từ 127.0.0.1 sang 0.0.0.0)** trong `docker-compose.yml`. Trong môi trường WSL2, nếu chỉ bind vào localhost của máy ảo Linux, trình duyệt Windows sẽ không thể truy cập trực tiếp vào Dashboards hay Jaeger UI. Việc mở rộng bind giúp quy trình quan sát (Observability) trở nên khả thi từ máy host. Ngoài ra, việc fix lỗi **Instrumentation lifespan** giúp Trace bắt đầu chảy về hệ thống thay vì bị bỏ sót.

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.
