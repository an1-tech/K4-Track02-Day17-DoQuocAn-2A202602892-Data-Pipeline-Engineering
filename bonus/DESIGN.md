# Pipeline dữ liệu cho trợ lý chăm sóc khách hàng thương mại điện tử

**Học viên:** Đỗ Quốc An — **MSSV:** 2A202602892.

Đây là thiết kế đề xuất cho bonus B2, phát triển từ bài toán CSKH của lab.
Các số về quy mô, SLA và ngân sách bên dưới là **giả định thiết kế**, chưa phải
số đo production. Prototype đã chạy là pipeline local và hai extensions của lab;
không tuyên bố đã triển khai hệ thống nguồn, vector database hay model thật.

## Bài toán và ràng buộc

Một doanh nghiệp bán hàng trên website và sàn cần trợ lý trả lời tình trạng đơn,
chính sách đổi trả, lỗi thanh toán và chuyển ticket tới bộ phận phù hợp. Dữ liệu
đến từ ERP chứa đơn hàng, hệ thống ticket, chính sách sản phẩm và trace hội thoại.
Nguồn thay đổi không đồng thời: đơn hàng có cập nhật trạng thái, chính sách có
phiên bản hiệu lực, người dùng có thể gửi feedback sau nhiều ngày. Tiếng Việt có
dấu, tên sản phẩm viết tắt và nội dung chụp từ điện thoại làm chất lượng không đều.

Giả định ban đầu: 20.000 đơn mỗi ngày, 2.000 ticket mỗi ngày, tài liệu chính sách
tối đa 10.000 trang; một kỹ sư phụ trách pipeline. Mục tiêu là trạng thái đơn mới
trong năm phút, tài liệu chính sách mới trong một giờ và dataset huấn luyện mới
mỗi ngày. Ngân sách giả định cho xử lý dữ liệu là 3 triệu đồng một tháng, chưa tính
phí model phục vụ hội thoại. Cần kiểm chứng các giả định bằng đo tải và báo giá
trước triển khai. Trợ lý chỉ đề xuất câu trả lời/chuyển tuyến; thao tác hoàn tiền
vẫn qua dịch vụ nghiệp vụ có kiểm soát riêng.

## Quyết định 1 — Nguồn, khóa và schema drift

**Câu hỏi:** làm sao nối dữ liệu khi ERP và ticket cập nhật khác thời điểm?
Tôi chọn CDC theo khóa đơn/ticket cho bảng nghiệp vụ, export theo phiên bản cho
tài liệu và event bất biến cho trace/feedback. Bronze giữ payload gốc, source,
ingest time, offset và schema version. Silver chuẩn hóa khóa, timestamp UTC và
schema; ticket nhận trạng thái có LSN mới nhất. Khi mở rộng nhiều nguồn, thứ tự
thay đổi phải gắn với từng nguồn hoặc phân vùng, không dùng một max LSN toàn cục
để bỏ mọi bản ghi tới muộn. Trường mới được quan sát và kiểm tra trước khi dùng.

Đánh đổi: CDC chính xác hơn polling timestamp khi có delete nhưng đòi hỏi vận
hành connector và lưu log nguồn đủ lâu. Export tài liệu đơn giản hơn CDC, đổi lại
cần hash nội dung và manifest để phát hiện tài liệu bị gỡ. Dòng không khớp schema
vào quarantine có reason; không im lặng bỏ lỗi để tiếp tục xuất dữ liệu sai.

## Quyết định 2 — Batch hay streaming và độ tươi

**Câu hỏi:** phần nào thật sự cần cập nhật liên tục? Tôi chọn microbatch năm phút
cho trạng thái đơn/ticket, batch một giờ cho tài liệu và batch hằng ngày cho train.
Khi trả lời trạng thái đơn, agent gọi API nghiệp vụ theo quyền người dùng; chỉ mục
RAG không là nguồn quyết định cuối cho trạng thái giao hàng. Luồng ingest dùng
cùng hàm xử lý cho chạy ngày thường và backfill, như `run_day` trong lab.

Đánh đổi: microbatch dễ vận hành và dễ replay hơn streaming liên tục, nhưng chấp
nhận độ trễ trong SLA giả định. Streaming chỉ được bổ sung nếu số đo chứng minh
năm phút làm trải nghiệm kém. Tôi loại phương án dựng đồng thời Lambda batch và
streaming với hai bộ logic: nhóm một người khó bảo đảm hai đường tính giống nhau,
chi phí vận hành tăng trước khi có bằng chứng về nhu cầu.

## Quyết định 3 — Chất lượng, PII và yêu cầu xóa

**Câu hỏi:** dữ liệu nào được phép vào context của model? Tôi chọn gate ở
Bronze→Silver: schema validation, regex email/điện thoại và nhận diện tên/địa chỉ
tiếng Việt. Bronze có quyền truy cập hạn chế; log lỗi sử dụng mã tham chiếu thay
cho in toàn văn. Tập đánh giá PII gán nhãn thủ công đo precision và recall theo
loại thực thể; rà soát cả trường hợp tên sản phẩm bị che nhầm. Quarantine tăng
đột biến phải được người vận hành xem xét trước khi publish dataset mới.

Đánh đổi: che mạnh giảm rò rỉ nhưng có thể mất ngữ cảnh cần cho phân loại.
Tombstone giữ khóa và thứ tự thay đổi để chặn replay hồi sinh; đây chưa phải quy
trình xóa hoàn chỉnh. Yêu cầu xóa phải chạy qua raw payload, transcript, snapshot,
embedding/LLM cache, chỉ mục và backup theo chính sách lưu trữ. Version dataset
bị thu hồi có manifest trạng thái và metadata kiểm toán; không giữ nguyên văn
PII chỉ để tái lập. Lab chỉ kiểm chứng snapshot mới nhất và RAG loại ticket xóa.

## Quyết định 4 — Event time, late data và train/serve parity

**Câu hỏi:** feedback tới muộn được tính vào ngày nào và train có biết tương lai
không? Tôi chọn event time cho feature, ingest time để xác định dữ liệu đã biết
tại một thời điểm. Lookback được đo từ Bronze theo P99, rồi kiểm tra với full
recompute. Seed của lab đo được P99 bằng ba ngày; con số này chỉ áp dụng seed,
không mặc định dùng cho cửa hàng thật. Theo dõi phần đuôi ngoài cửa sổ và tạo
backfill riêng khi phát hiện event muộn hơn ngưỡng.

Đánh đổi: overwrite partition tránh cộng trùng nhưng tăng số partition phải tính
lại. Dataset training dùng ASOF/point-in-time để chọn feature sẵn có lúc dự đoán;
feedback ngày hôm sau không được đưa vào bản snapshot hôm trước. Cần lưu version
logic, khoảng thời gian và manifest input để tái lập. So sánh feature train và
serve trên cùng bộ tình huống trước khi đưa model mới vào sử dụng.

## Quyết định 5 — RAG, knowledge graph và flywheel

**Câu hỏi:** retrieval nào phù hợp và feedback được dùng ra sao? Tôi chọn tìm kiếm
kết hợp từ khóa/vector cho FAQ và chính sách, có bộ lọc tenant, version hiệu lực
và quyền truy cập trước khi nội dung vào context. Các câu cần nối nhiều quan hệ,
ví dụ sản phẩm → phụ kiện → kho giao, có thể dùng graph sau khi chứng minh giá trị
trên tập câu hỏi. Trace sinh eval holdout và cặp preference; decontamination loại
prompt trùng hoặc gần trùng eval khỏi train. Feedback phải gắn phiên bản model,
retrieval và prompt để tránh coi mọi lượt dislike là lỗi cùng nguyên nhân.

Đánh đổi: graph hỗ trợ multi-hop nhưng extractor và entity resolution có thể sai;
graph toàn bộ tài liệu từ đầu quá tốn chi phí kiểm chứng. Vector retrieval dễ
khởi đầu nhưng phải kiểm tra groundedness và quyền truy cập. Tôi loại việc lấy
tất cả câu trả lời của agent làm dữ liệu tốt để fine-tune: nó có thể khuếch đại
lỗi của chính model và làm metric đẹp giả nếu lẫn dữ liệu eval.

## Quyết định 6 — Replay, cache và vận hành khi tăng quy mô

**Câu hỏi:** chạy lại và tăng tải mười lần có còn an toàn? Tôi chọn idempotency
theo khóa ở Silver, overwrite partition cho aggregate và cache embedding theo
hash nội dung cộng model version. LLM transform thêm prompt version vào cache;
validate output và lưu cả kết quả lỗi để replay không gọi lại vô hạn. Ước tính
token và ngân sách trước batch; lỗi tạm thời cần chính sách retry riêng, khác lỗi
schema đã xác định. Chi phí lab dùng giá giả lập, không suy ra hóa đơn production.

Đánh đổi: cache giảm tiền gọi model nhưng phải có invalidation, quota và purge
khi dữ liệu bị xóa. DuckDB là prototype một máy, một writer; khi tải tăng, đo
thời gian backfill, peak memory, số file nhỏ và chi phí LLM trước khi chuyển sang
warehouse/lakehouse hoặc xử lý phân tán. Checksums, số dòng, tỷ lệ quarantine và
freshness giúp phát hiện pipeline chạy xong nhưng dữ liệu sai. Không chọn Spark
chỉ vì số lượng công cụ; chỉ chuyển khi giới hạn đã được đo.

## Sơ đồ kiến trúc

```text
ERP/tickets CDC ───────┐
Policies + manifest ──┼─> Bronze raw + lineage (restricted access)
Agent traces/events ──┘                  │
                               Schema + PII gate
                                  │          └─> Quarantine + review
                           Silver keyed state/events
                                  │
                 ┌────────────────┼──────────────────┐
             Features         Doc chunks       Trace datasets
           event time      hash/model cache    holdout + clean DPO
                 │                │                  │
           Routing model     RAG index        Versioned training
                 └────────────────┼──────────────────┘
                             Support agent
                                  │
                          Trace + user feedback

Deletion manifest ─> raw/snapshots/cache/index/backups + version revocation
Order status response ─> authorized business API (current source of truth)
```

## Kiểm chứng và giới hạn

Prototype local đã có contract, PII regex, keyed merge, delete tombstone,
lookback đo từ Bronze và checksum rerun. Extensions minh họa decontamination,
ASOF JOIN và traversal hai hop. Chúng chưa chứng minh chất lượng embedding ngữ
nghĩa, NER tiếng Việt, quyền truy cập nhiều tenant hay SLA production.
Trước triển khai cần tập eval có nhãn, kiểm tra xóa end-to-end và load test; kết
quả đo sẽ quyết định có thay công cụ hoặc điều chỉnh các giả định SLA hay không.
