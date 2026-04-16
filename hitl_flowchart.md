# Human-In-The-Loop (HITL) Workflow Flowchart

Sơ đồ quy trình dưới đây khắc họa hệ thống **Defense-in-Depth Pipeline kết hợp HITL**, đảm nhiệm việc cân bằng giữa tự động hóa (AI) và rủi ro tuân thủ bằng cách sử dụng **3 Điểm Quyết Định (Decision Points)** có sự can thiệp của con người.

## Sơ đồ Quy trình (Mermaid)

```mermaid
flowchart TD
    %% Tùy chỉnh màu sắc
    classDef userReq fill:#E1F5FE,stroke:#0288D1,stroke-width:2px;
    classDef aiProcess fill:#F3E5F5,stroke:#8E24AA,stroke-width:2px;
    classDef secCheck fill:#FFF3E0,stroke:#F57C00,stroke-width:2px;
    classDef decision fill:#E8EAF6,stroke:#3F51B5,stroke-width:2px,stroke-dasharray: 3 3;
    classDef human fill:#FFEBEE,stroke:#D32F2F,stroke-width:3px;
    classDef endNode fill:#E8F5E9,stroke:#388E3C,stroke-width:2px;

    %% Các Node Chính
    Start([Bắt đầu: User Gửi Request]) ::: userReq
    InputGuard[Lớp Bảo Vệ Input: Quét Regex & Độc hại] ::: secCheck
    
    %% Quyết định 1: Yêu cầu nhạy cảm
    Decision1{Decision 1:\nYêu cầu có thuộc danh mục\nnhạy cảm hành chính không?\n(Khoá thẻ, Vay vốn, Chuyển khoản lớn)} ::: decision
    
    LLM_Core[LLM Xử lý: Gemini 2.5 Sinh Câu Trả Lời] ::: aiProcess
    
    %% Quyết định 2: Confidence
    Decision2{Decision 2:\nĐiểm số tự tin (Confidence Score)\ncủa Token Answer > 85%?} ::: decision

    OutputGuard[Lớp Bảo Vệ Output: LLM-as-Judge & Redact PII] ::: secCheck

    %% Quyết định 3: Chặn nhiều lần
    Decision3{Decision 3:\nCâu trả lời có bị Judge chặn\n> 2 lần liên tục không?} ::: decision

    %% Các Node Human
    HumanEscalation1[[Escalation 1: Trực tiếp luân chuyển User\nsang Box Chat với Giao dịch viên]] ::: human
    HumanEscalation2[[Escalation 2: Chuyển Ticket Review\ncho Quản lý Ngân hàng Kênh số]] ::: human
    HumanEscalation3[[Escalation 3: Đưa câu hội thoại vào Console\ncho Reviewer chỉnh sửa & Trả lời]] ::: human

    Finish([Kết thúc: Trả phản hồi an toàn cho User]) ::: endNode
    BlockR([Kết thúc: Báo lỗi Blocked / Rate Limit]) ::: endNode

    %% Luồng đi
    Start --> InputGuard
    InputGuard -- "Failed / Injection" --> BlockR
    InputGuard -- "PASS" --> Decision1

    Decision1 -- "Có (Nhạy cảm pháp lý)" --> HumanEscalation1
    Decision1 -- "Không (Hỏi đáp bình thường)" --> LLM_Core

    LLM_Core --> Decision2
    Decision2 -- "Scores < 85% (Hallucination Risk)" --> HumanEscalation2
    Decision2 -- "Scores >= 85%" --> OutputGuard

    OutputGuard -- "Judge FAIL / Chứa PII" --> Decision3
    OutputGuard -- "PASS & Redacted" --> Finish

    Decision3 -- "Vượt quá ngưỡng chặn" --> HumanEscalation3
    Decision3 -- "Chưa vượt (Gọi lại LLM / Fix)" --> LLM_Core

    %% Trả về người dùng sau Human Support
    HumanEscalation1 --> Finish
    HumanEscalation2 --> Finish
    HumanEscalation3 --> Finish
```

---

## Giải thích chi tiết 3 Điểm Quyết Định (Decision Points)

**1. Decision 1: Phân luồng Danh mục Pháp lý (Regulatory routing)**
- *Logic:* Hệ thống kiểm tra ý định (Intent) của người dùng hoặc check Keyword xem tác vụ đang làm là gì.
- *Leo thang (Escalation):* Nếu tác vụ mang tính pháp lý/tài chính cao (ví dụ: Huỷ tài khoản, vay tiền khẩn cấp) buộc phải có chữ ký hoặc xác thực của người thật, AI sẽ DỪNG hoàn toàn việc suy luận. Trạng thái live-chat được *escalate* ngay lập tức sang màn hình của Giao dịch viên (Call Center Agent).

**2. Decision 2: Routing dựa trên sự tự tin của Model (Confidence Score Routing)**
- *Logic:* Đôi khi AI trả lời nhưng với Log-Probabilities / Confidence Score rất thấp (dưới 85%). Nếu đáp án là về Lãi suất huy động nhưng AI không chắc chắn, nguy cơ "Hallucination" (bịa đặt số liệu ngân hàng) là cực kỳ thảm họa.
- *Leo thang (Escalation):* AI không gửi câu trả lời hiển thị cho User. Thay vào đó nó đính kèm tag `Review-Needed` và chuyển thẳng Ticket đó cho Backend Staff. Staff này sẽ cung cấp số liệu thực, duyệt câu trả lời và hệ thống mới gửi về người dùng.

**3. Decision 3: Vượt quá giới hạn QA Fail (Infinite-Loop Prevention)**
- *Logic:* Lớp bảo vệ LLM-as-Judge hoặc Output PII lọc quá nghiêm ngặt. Hệ thống tạo ra câu trả lời, nhưng bị Judge đánh hỏng 3 lần liên tiếp do ngữ cảnh quá tế nhị.
- *Leo thang (Escalation):* Nếu cứ bắt AI sinh lại sẽ tốn tiền và User sẽ phải chờ quá lâu (Timeout). Tại điểm thứ 3, hệ thống sẽ ngắt truy vấn LLM, báo cho User "Hàng chờ xin phản hồi đang được xử lý..." và ném hội thoại đó vào 1 Console Tool dành cho Technical Data Reviewer để trực tiếp gỡ rối (Sửa câu lệnh sinh hỏng hoặc điều chỉnh Judge).
