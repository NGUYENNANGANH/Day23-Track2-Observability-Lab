# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Năng Anh
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/NGUYENNANGANH/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.0.1)
Compose v2:    OK  (2.40.3-desktop.1)
RAM available: 3.66 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: D:\thucchienai\Day23-Track2-Observability-Lab\00-setup\setup-report.json
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
| _T1+60s_ | Slack posted an explicit `RESOLVED: ServiceDown` follow-up after service recovery | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Sự đa dạng của các plugin và khả năng tùy biến mạnh mẽ trên Grafana, đặc biệt là cách Prometheus dùng labels để lọc và gộp dữ liệu đa chiều rất tiện lợi khi troubleshooting các dịch vụ AI.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.2125, "trace_id": "81eacda2778b73628e42e7de90561dac", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T16:47:39.475351Z"}
```

### Tail-sampling math

Nếu ứng dụng tạo ra N traces/sec, chính sách probabilistic-1pct sẽ giữ lại `N * 1%` traces khỏe mạnh. Đồng thời chính sách keep-errors và keep-slow sẽ giữ lại 100% traces có lỗi hoặc chậm. Vì vậy tổng số trace giữ lại là `(N * 0.01) + E + S` (với E, S là lượng traces lỗi và chậm). Tỉ lệ giữ lại = `(0.01*N + E + S) / N`.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 0.25,
    "kl": 0.1,
    "ks_stat": 0.15,
    "ks_pvalue": 0.001,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

- `prompt_length`: **KS Test** vì phân phối liên tục và KS rất nhạy cảm với sự dịch chuyển phân phối (shift in mean/variance).
- `embedding_norm`: **MMD** vì khoảng cách của các embedding (vectors) nhiều chiều sẽ phản ánh trung thực nhất qua kernel-based MMD.
- `response_length`: **PSI** vì thường ta chia length thành các bucket (bin) để so sánh tính ổn định của độ dài phản hồi.
- `response_quality`: **KL Divergence** vì chất lượng thường là phân phối dạng Beta, KL sẽ đo lường lượng thông tin khác biệt giữa 2 phân phối.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Llama.cpp từ Day 20 khó expose nhất. Lý do là vì HTTP server mặc định của llama.cpp không expose sẵn định dạng metrics cho Prometheus, mà cần phải cấu hình thông qua một log-tail sidecar (như script stub) để đếm số lượng tokens/sec hay queue depth, yêu cầu thêm công sức parse log và chuyển đổi.

---

## 6. The single change that mattered most

> **Grader reads this closest.**
Sự khác biệt lớn nhất giữa một hệ thống "chạy được" và "hữu ích" là việc bổ sung **Custom Metrics (như `inference_quality`, `input_tokens`, và `output_tokens`)** thay vì chỉ đo đếm latency hoặc request count thuần túy (như các web server truyền thống).

Nhờ có các metrics mang tính đặc thù của Generative AI này (cùng với việc nhóm theo nhãn `model`), ta có thể thiết lập các cảnh báo SLI/SLO Burn-rate chính xác hơn. Nó cho phép đội vận hành (SRE) phát hiện ngay lập tức tình trạng suy giảm chất lượng sinh text hoặc đột biến chi phí Token API — vốn là những vấn đề đau đầu nhất khi triển khai AI ra Production. Điều này liên kết chặt chẽ với khái niệm "RED/USE cho AI" từ bài giảng.
