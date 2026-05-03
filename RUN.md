# Hướng dẫn chạy RAG Chatbot (rag-project)

Dự án: **Gradio** + **LlamaIndex**, LLM qua **Ollama**, embedding/rerank qua **Hugging Face**. Trong `pyproject.toml` chỉ ghi **`requires-python = ">=3.11"`** — nhưng với các gói hiện tại trong `uv.lock`, **không nên** dùng Python quá mới trên máy dev; bên dưới có **phiên bản khuyến nghị** để ít lỗi nhất.

---

## 1. Chuẩn môi trường (khuyến nghị cố định)

Để giống “toolchain mà lockfile và wheel đã được thử phổ biến”, dùng **một trong hai**:

| Cách | Ý nghĩa |
|------|---------|
| **Khuyến nghị** | **CPython 3.12.x** (ví dụ 3.12.13): đủ `>=3.11`, các gói kiểu `onnxruntime` / ML thường có wheel sẵn, vẫn còn module **`audioop`** trong stdlib (tránh lỗi Gradio/`pydub` hay gặp trên **3.13+**). |
| Tối thiểu theo manifest | **3.11.x** hoặc **3.12.x** — miễn là **không phải 3.14+** và nếu dùng **3.13** thì có thể cần thêm xử lý (mục 6). |

**Không** dựa vào `python3` hệ thống nếu đó là bản **3.14**: `uv`/PyPI có thể **không có wheel** cho một số dependency (ví dụ `onnxruntime`), và bạn sẽ lỗi khi `uv sync`.

**Cách làm gọn với `uv` (đồng nhất máy của bạn với nhau):**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
# mở shell mới nếu cần để có lệnh `uv`

cd /đường/dẫn/rag-project
uv python install 3.12
echo "3.12" > .python-version    # không bắt buộc nếu bạn chỉ thích gõ tay; có thì uv tự nhận
rm -rf .venv
uv sync --locked
uv run python --version           # mong đợi: Python 3.12.x
```

Mọi lệnh **`uv run ...`** trong project sau đó dùng đúng `.venv` của bản Python đó — **không cần** “cái mới nhất” trên distro.

---

## 2. Lỗi build: “Readme file does not exist: README.md”

Nếu `pyproject.toml` vẫn có **`readme = "README.md"`** nhưng repo không có **`README.md`**, Hatchling sẽ lỗi khi **`uv sync`**. Cách xử lý: tạo `README.md` tối thiểu, **hoặc** bỏ dòng `readme = ...` trong `pyproject.toml` (đã chỉnh sẵn trong bản này nếu bạn không reset lại upstream).

---

## 3. Ollama (LLM local)

```bash
curl -fsSL https://ollama.com/install.sh | sh
# đảm bảo API chạy, ví dụ:
curl -s http://127.0.0.1:11434/api/tags >/dev/null && echo OK
```

Model mặc định trong code thường là `llama3:8b-instruct-q8_0`; có thể `pull` model khác và chọn đúng tên trong UI.

```bash
ollama pull llama3:8b-instruct-q8_0
# hoặc model nhẹ hơn tuỳ máy
```

---

## 4. Chạy app (thuần lệnh, giống `scripts/run.sh`)

```bash
uv run python -m rag_chatbot --host localhost
```

Trình duyệt mở URL Gradio trong log (thường port **7860**).

Tuỳ chọn: `./scripts/run.sh` (cần `chmod +x` nếu chưa).

**Lưu ý:** Trong `Makefile`, `make run` gọi `main.py` — file đó **không có trong repo**. Dùng lệnh `python -m rag_chatbot` như trên.

---

## 5. Lần đầu ingest / embedding

- Model Hugging Face (embedding, rerank) **tự tải** lần đầu, cache thường dưới `data/huggingface/` (theo `rag_chatbot/setting/setting.py`).
- Cần mạng; máy không GPU vẫn chạy được (chậm hơn khi index PDF).

---

## 6. Python 3.13 và 3.14 (chỉ khi bạn bắt buộc)

- **3.13:** trong stdlib **không còn** `audioop`; Gradio kéo `pydub` có thể báo **`No module named 'audioop'` / `pyaudioop`**. Giải pháp đơn giản nhất là **dùng 3.12** như mục 1. Nếu vẫn dùng 3.13, cần cài/backport được tài liệu gói thay cho `audioop` (tuỳ thời điểm, có gói tương thích trên PyPI).
- **3.14:** nhiều wheel (ví dụ `onnxruntime`) có thể **chưa hỗ trợ** — `uv sync` dễ thất bại. **Khuyến nghị không dùng** cho đến khi toàn bộ dependency có wheel cho 3.14.

---

## 7. Docker (tuỳ chọn)

Image dùng `pytorch/pytorch` + `uv sync --locked`. Trên Linux, app gọi Ollama qua `host.docker.internal` có thể cần cấu hình thêm (`extra_hosts` hoặc dùng service `ollama` trong `docker-compose.yml`). Chi tiết chỉnh theo máy của bạn; cách làm quen nhất là chạy **Ollama + `uv run` trên host** như các mục trên.

---

## 8. Checklist ngắn

1. Cài **`uv`**.
2. **`uv python install 3.12`**, (**tùy chọn**) `echo 3.12 > .python-version`, **`uv sync --locked`**.
3. Không có lỗi readme: **`pyproject.toml` không bắt buộc `readme =`** (hoặc phải có `README.md` nếu bạn giữ dòng đó sau khi pull upstream khác).
4. Cài và chạy **Ollama**, `ollama pull` model.
5. **`uv run python -m rag_chatbot --host localhost`**.
