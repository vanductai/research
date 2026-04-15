# Kiến Trúc Triển Khai AI Roundtable (Agent-Agnostic Design)

Để hệ thống **AI Roundtable** có thể hoạt động linh hoạt, liền mạch và tương thích với *bất kỳ AI Agent nào* (từ các công cụ dòng lệnh như `claude-cli`, `gemini-cli` cho đến các AI IDE Editor tiên tiến như `Antigravity`, `Cursor`, `OpenCode`), chúng ta cần từ bỏ việc trói buộc (hard-code) hệ thống vào một bộ SDK API cụ thể. 

Thay vào đó, hệ thống cần được thiết lập xoay quanh **4 Nguyên lý Kiến trúc Cốt lõi** sau:

---

## 1. Filesystem-as-API (Biến Thư Mục Thành Hệ CSDL)

Bất kỳ AI Agent nào hiện nay, dù khác biệt về nhà phát triển, đều chia sẻ một khả năng cốt lõi: **Đọc văn bản thuần (Text) và Ghi file (Markdown)**. Do đó, state (trạng thái) của hệ thống sẽ không lưu ở Database, mà lưu hoàn toàn trên ổ cứng cục bộ.

**Cấu trúc thư mục tiêu chuẩn (Template):**
```text
/roundtable-core
│
├── /prompts/                 # Nơi chứa trí thông minh (System Prompts)
│   ├── system_moderator.md   # Nhiệm vụ của Moderator
│   ├── system_debater.md     # Logic lập luận (Dùng chung cho cả PRO/CON)
│   ├── system_judge_u.md     # Quy tắc chấm điểm Utilitarian
│   └── persona_matrix.json   # Thư viện 12 Personas
│
├── /workspace/               # Nơi chứa Data đang chạy (Context)
│   └── /session_001_M&A/
│       ├── 00_input.md                 # Ý tưởng gốc của khách hàng
│       ├── 01_brief.md                 # Output của Moderator
│       ├── 02_side_a_opening.md        # Output của Analyst A
│       ├── 03_referee_report.md        # Quyết định của Trọng tài
│       └── ...
│
└── orchestration.sh          # Mã kịch bản điều phối các CLI
```

**Cách hoạt động**: Khi một Agent tham gia vào một bước, nó chỉ được cấp quyền đọc các hệ quả từ bước ngay trước đó trong `/workspace/`, và ghi đè kết quả của nó ra một file Markdown hoàn toàn tĩnh.

---

## 2. Tiêu Chuẩn Hóa Input/Output (Plain Text Piping)

Đối với các AI Agent chạy bằng CLI (như `claude-cli`, `gemini-cli`), việc kết nối các "chuyên gia" này diễn ra thông qua Text Piping cơ bản của hệ điều hành.

**Luồng làm việc mẫu cho Step 1 (Moderator):**
```bash
# Sử dụng biến môi trường $AI_CLI để linh hoạt đổi Model
export AI_CLI="claude-cli --model claude-3.5-sonnet"
# Hoặc export AI_CLI="gemini-cli --model gemini-1.5-pro"

cat prompts/system_moderator.md workspace/00_input.md | $AI_CLI > workspace/01_brief.md
```

Đối với các AI dạng Editor (Cursor, Antigravity, OpenCode):
User chỉ cần tạo một prompt ngay trong trình duyệt chat của màn hình workspace:
>*"@workspace Đọc /prompts/system_moderator.md, áp dụng yêu cầu vào file /workspace/00_input.md và xuất kết quả lưu tại /workspace/01_brief.md"*

Nhờ định dạng chuẩn Markdown, AI Editor dễ dàng hoàn thành Tasktool một cách chính xác mà không cần biết nó đang nằm trong Pipeline nào.

---

## 3. Cô Lập Ràng Buộc (Context Isolation)

Điểm nghẽn lớn nhất để duy trì tính độc lập (tránh trường hợp chuyên gia A đọc lén suy nghĩ của chuyên gia B) đối với các AI tích cực đọc Workspace (như Cursor/Antigravity) là cơ chế chặn (Isolation).

**Cách setup hiệu quả:**
- **Sử dụng `.roundtablerules` hoặc `.cursorrules` linh động**: Tại mỗi bước (Step), trình điều phối sẽ ghi đè file `.rules` ở gốc nhằm giới hạn tầm nhìn của AI Agent hiện tại.
- Ví dụ, khi Antigravity đóng vai `Debater PRO` ở Vòng 2, Rule sẽ ghi rõ:
    ```markdown
    CRITICAL RULE: 
    You are acting as Debater PRO. 
    You are ONLY allowed to read `01_brief.md` and `02_side_b_opening.md`.
    DO NOT use native search to find other side_a or side_b files.
    Write your output EXACTLY to `03_side_a_rebuttal.md`.
    ```
- Điều này ép Agent hành xử độc lập như một phiên luân phiên thực sự.

---

## 4. Lớp Điều Phối Tự Động (Orchestration Layer)

Để hệ thống chạy mượt từ Bước 0 đến Bước 10 trên bất kỳ nền tảng nào, ta cần một Orchestrator siêu nhẹ. Tốt nhất là sử dụng **Makefile** hoặc mã **Python Script** không yêu cầu thư viện mạnh.

**Ví dụ một đoạn `Makefile` điều phối Agnostic:**

```makefile
# Khai báo Model tùy chọn
MODERATOR_MODEL = claude-cli
ANALYST_A_MODEL = gemini-cli
JUDGE_U_MODEL   = chatgpt-cli

# Chạy tự động toàn bộ tiến trình
run-all: step-1 step-2 step-3 ... finish

step-1-moderator:
	@echo "Moderator đang phân tích Input..."
	@cat prompts/system_moderator.md in.md | $(MODERATOR_MODEL) > workspace/01_brief.md

step-2-analysts:
	@echo "Team Analyst đang mổ xẻ song song..."
	@# Trực tiếp kích hoạt Parallel Bash job
	@(cat prompts/system_debater.md workspace/01_brief.md | $(ANALYST_A_MODEL) > workspace/02_side_a_open.md) & \
	 (cat prompts/system_debater.md workspace/01_brief.md | $(ANALYST_B_MODEL) > workspace/02_side_b_open.md) & \
	 wait
	@echo "Hoàn thành trình bày vòng 1."
```

## Kết Luận: Kiến Trúc Mở Không Phụ Thuộc (Vendor Lock-in Free)

Thiết lập theo cấu trúc bên trên mang lại 3 ưu điểm sống còn:
1. **Dễ dàng Plug-and-Play**: Muốn đưa Antigravity vào làm Tester, chỉ cần assign task cho Antigravity thực hiện thao tác tương đương. Muốn nhúng Gemini làm Meta-Judge, đổi CLI command trong 1 giây.
2. **Không tốn chi phí hạ tầng lớn**: Không cần Node.JS backend nặng nề, không cần DB. Folder Workspace chính là Project state, nén lại thành File .ZIP là giao ngay được cho Khách hàng.
3. **Audit Trail tuyệt đối minh bạch**: Khách hàng (doặc User) xem lại thư mục `/workspace/` là thấy rõ AI nào nói vào lúc nào, dùng bằng chứng gì (do các file Markdown tĩnh lưu trữ tuần tự thời gian). Mọi ngụy biện bị phơi bày hoàn toàn ở file `referee.md`.
