# Thiết kế: Coding/DevOps Agent bằng Go với Anthropic SDK

- **Ngày:** 2026-06-01
- **Trạng thái:** Đã duyệt thiết kế, chờ viết plan
- **Thư mục:** `ai-auto/`

## 1. Mục tiêu

Xây dựng một CLI agent viết bằng Go, dùng Anthropic SDK chính thức
(`github.com/anthropics/anthropic-sdk-go`), có khả năng tự động hoá các tác vụ
coding/DevOps trong workspace: đọc, sửa, tạo file, tìm kiếm và chạy lệnh shell —
theo cơ chế tool-use của Claude. Đây là một "mini coding agent".

### Phạm vi (MVP)

- Vòng lặp agent (tool-use loop) hoàn chỉnh.
- 6 tool: `read_file`, `write_file`, `edit_file`, `list_files`, `grep`, `run_command`.
- Hai chế độ chạy: REPL tương tác (không có prompt) và one-shot (`agent "task..."`).
- Sandbox theo workspace + cơ chế xác nhận (approval) 3 mức.

### Ngoài phạm vi (YAGNI cho MVP)

- Memory/persistence giữa các phiên.
- Framework agent bên thứ ba (eino, langchaingo).
- Đa agent / orchestration phức tạp.
- Streaming UI nâng cao.
- Gọi Claude thật trong unit test.

## 2. Phương án kỹ thuật

**Đã chọn: Agent loop thuần với SDK chính thức.** Tự viết vòng lặp tool-use trên
`anthropic-sdk-go`. Lý do: ít phụ thuộc, sát khuyến nghị của Anthropic, kiểm soát
hoàn toàn cơ chế tool-use, đủ cho coding agent.

Phương án loại bỏ: dùng framework agent có sẵn (nặng, che cơ chế tool-use); tự gọi
REST API không qua SDK (tốn công với schema/retry/streaming).

## 3. Cấu trúc thư mục

```
ai-auto/
├── go.mod
├── cmd/agent/main.go        # entrypoint: parse args, chọn REPL/one-shot
├── internal/
│   ├── agent/
│   │   ├── agent.go         # Agent struct + vòng lặp Run()
│   │   └── loop.go          # xử lý tool_use → tool_result
│   ├── tools/
│   │   ├── tool.go          # interface Tool + registry
│   │   ├── read_file.go
│   │   ├── write_file.go
│   │   ├── edit_file.go
│   │   ├── run_command.go
│   │   ├── list_files.go
│   │   └── grep.go
│   ├── llm/
│   │   └── client.go        # khởi tạo anthropic.Client, cấu hình model
│   ├── config/
│   │   └── config.go        # API key, model, workspace root, max turns
│   └── ui/
│       └── console.go       # in màu, prompt xác nhận, REPL input
└── README.md
```

Nguyên tắc thiết kế: mỗi tool là một unit độc lập (1 file, implement chung
interface `Tool`), test được riêng. Lõi loop không biết chi tiết tool — chỉ làm
việc qua registry.

## 4. Tool interface & Registry

```go
type Tool interface {
    Name() string                                 // vd "read_file"
    Description() string                          // mô tả cho Claude
    InputSchema() anthropic.ToolInputSchemaParam  // JSON schema tham số
    Execute(ctx context.Context, input json.RawMessage) (string, error)
}
```

- **Registry** giữ `map[string]Tool`, xuất ra `[]anthropic.ToolUnionParam` để đưa
  vào request và tra cứu tool theo tên khi xử lý `tool_use`.
- Tool trả về `(string, error)`. Lỗi được đóng gói vào `tool_result` với
  `is_error: true`, **không** làm sập chương trình.

### Mô tả 6 tool

| Tool | Tham số | Hành vi |
|---|---|---|
| `read_file` | `path` | Đọc nội dung file (chỉ-đọc). |
| `write_file` | `path`, `content` | Tạo/ghi đè file. Cần approval. |
| `edit_file` | `path`, `old_string`, `new_string` | Thay thế chuỗi chính xác, hiển thị diff. Cần approval. |
| `list_files` | `path` (tuỳ chọn) | Duyệt cây thư mục (chỉ-đọc). |
| `grep` | `pattern`, `path` (tuỳ chọn) | Tìm nội dung theo regex (chỉ-đọc). |
| `run_command` | `command` | Chạy lệnh shell, cwd=workspace, có timeout. Cần approval. |

## 5. Agent loop

`agent.Run(ctx, prompt)`:

```
1. messages = [user prompt]
2. Lặp (tối đa MaxTurns, mặc định 50):
   a. resp = client.Messages.New(model, messages, tools, system)
   b. append assistant response vào messages
   c. nếu resp KHÔNG có tool_use → in text cuối, kết thúc
   d. với mỗi tool_use block:
        - tìm tool trong registry (không thấy → tool_result is_error)
        - nếu tool cần approval và mode yêu cầu → xin xác nhận người dùng
          (từ chối → tool_result is_error "user declined")
        - Execute → thu kết quả/lỗi
        - tạo tool_result block (is_error nếu lỗi)
   e. append MỘT user message chứa TẤT CẢ tool_result → quay lại (a)
3. chạm MaxTurns → dừng an toàn, báo người dùng
```

**Quy tắc:**
- Tất cả `tool_result` của một lượt gộp vào **một** user message (đúng yêu cầu API).
- Lỗi tool → `is_error: true` để Claude tự xử lý/thử lại, không crash.
- Mỗi vòng in ra hành động (tool nào, tham số gì) để người dùng theo dõi.
- `agent.Run` phụ thuộc **interface `LLMClient`** (không phụ thuộc trực tiếp struct
  SDK) để mock được trong test.

```go
type LLMClient interface {
    CreateMessage(ctx context.Context, messages []anthropic.MessageParam) (*anthropic.Message, error)
}
```

## 6. An toàn & Sandbox

### 6.1 Sandbox theo workspace root

- Mọi tool thao tác file resolve đường dẫn về tuyệt đối, kiểm tra **phải nằm trong
  `WorkspaceRoot`**. Thoát ra ngoài (vd `../../etc/passwd`) → trả lỗi, không thực thi.
- `WorkspaceRoot` mặc định là thư mục hiện tại, override qua `--workspace`.

### 6.2 Cơ chế approval (3 mức, qua `--approval`)

- `suggest` (**mặc định**): hỏi xác nhận trước mỗi `write_file`, `edit_file`,
  `run_command`. Tool chỉ-đọc (`read/list/grep`) chạy thẳng.
- `auto-edit`: tự cho phép sửa file, vẫn hỏi với `run_command`.
- `full-auto`: không hỏi (CI/one-shot tin tưởng) — in cảnh báo rõ khi bật.

Prompt xác nhận hiển thị tên tool, tham số đầy đủ (đường dẫn/lệnh), và với edit thì
**diff** trước khi áp dụng. Người dùng chọn `y/n`.

### 6.3 Chặn lệnh nguy hiểm (phòng thủ nhẹ)

- `run_command` có danh sách pattern chặn cứng (vd `rm -rf /`, fork bomb, ghi đè
  ngoài workspace). Mục tiêu: bắt lỗi vô ý, không phải chống mọi tấn công.
- Lệnh chạy với timeout (mặc định 120s) và `cwd = WorkspaceRoot`.

### 6.4 One-shot mặc định an toàn

One-shot mặc định vẫn `suggest` (hỏi). Muốn chạy không tương tác phải chủ động
`--approval full-auto`.

## 7. Cấu hình

Ưu tiên: flag > env > mặc định.

| Mục | Nguồn | Mặc định |
|---|---|---|
| API key | env `ANTHROPIC_API_KEY` | bắt buộc, lỗi nếu thiếu |
| Model | `--model` / env `ANTHROPIC_MODEL` | `claude-opus-4-8` |
| Workspace root | `--workspace` | thư mục hiện tại |
| Approval mode | `--approval` | `suggest` |
| Max turns | `--max-turns` | 50 |
| Command timeout | `--cmd-timeout` | 120s |

- Hỗ trợ `.env` (godotenv) cho dev. Validate API key sớm để báo lỗi rõ ràng.
- System prompt cố định mô tả vai trò agent + nhắc dùng tool, đặt trong
  `internal/agent`.

## 8. Testing

Chạy `go test ./... -race`.

- **Tool tests (trọng tâm)** — mỗi tool test trên `t.TempDir()`:
  - happy path + lỗi (file không tồn tại, schema sai).
  - **Sandbox**: đường dẫn thoát workspace (`../`) bị từ chối.
  - `run_command`: lệnh hợp lệ chạy được; pattern nguy hiểm bị chặn; timeout hoạt động.
- **Registry test** — đăng ký/tra cứu tool, sinh schema đúng.
- **Loop test** — dùng **fake `LLMClient`** (không gọi mạng) trả về kịch bản
  `tool_use` định sẵn → kiểm tra loop thực thi tool, gộp tool_result, dừng đúng khi
  có text cuối và khi chạm MaxTurns.
- **Approval logic** — tách hàm thuần `needsApproval(tool, mode)` để test không cần I/O.

**Không test:** gọi Claude thật (chạy thủ công qua README), in màu console.

## 9. Tiêu chí hoàn thành (success criteria)

- `go build ./...` và `go test ./... -race` xanh.
- Chạy `agent "tạo file hello.go in ra Hello"` (one-shot) tạo được file đúng, có
  hỏi approval ở mức `suggest`.
- Chạy `agent` không tham số vào được REPL, hội thoại nhiều lượt, agent gọi tool
  và trả lời.
- Đường dẫn ngoài workspace và lệnh nguy hiểm bị từ chối có kiểm chứng bằng test.
- README hướng dẫn cài đặt, cấu hình API key, và ví dụ sử dụng.
