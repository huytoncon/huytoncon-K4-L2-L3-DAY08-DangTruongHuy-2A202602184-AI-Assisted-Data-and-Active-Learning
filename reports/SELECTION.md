# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

Nếu chỉ đủ công rà 5 ảnh, tôi chọn: **frame_0182.jpg** (hạng 1, score 0.9591, t=72.8s), **frame_0099.jpg**
(hạng 8, score 0.9063, t=39.6s), **frame_0227.jpg** (hạng 11, score 0.8915, t=90.8s), **frame_0331.jpg**
(hạng 5, score 0.9154, t=132.4s) và **frame_0369.jpg** (hạng 2, score 0.9324, t=147.6s).

Lý do: bốn ảnh đầu và ảnh cuối trải đều trên trục thời gian (39.6s → 72.8s → 90.8s → 132.4s → 147.6s),
tránh dồn ngân sách vào một đoạn video. Đây chính là quyết định phải xét ảnh gần trùng: trong 50 dòng
đầu, quanh t=130–132s có một cụm gần như liên tiếp — `frame_0326` (130.4s), `frame_0328` (131.2s),
`frame_0329` (131.6s), `frame_0330` (132.0s), `frame_0331` (132.4s) — cách nhau 0.4–0.8 giây, gần như
cùng một khung hình xe vì camera đứng yên (xem `data/DATA.md`). Thay vì rà cả cụm, tôi chỉ chọn một đại
diện có score cao nhất trong cụm (`frame_0331`, score 0.9154) và dành công còn lại cho các đoạn thời
gian khác chưa được xem. Tương tự, quanh t=147–153s có cụm `frame_0368/0369/0372/0374/0380/0383/0384`;
tôi chỉ chọn `frame_0369` (score cao nhất cụm) thay vì `frame_0372` (score 0.9101, cũng rất cao) vì
hai ảnh này cách nhau chỉ 1.2 giây.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

1. **frame_0182.jpg** — hạng 1, score 0.9591 (U=0.9182, A=1.0). Đây là ảnh có độ bất định cao nhất
   trong toàn bộ pool, và toàn bộ 18/28 box gợi ý (A=1.0) nằm ở mức tin cậy mơ hồ 0.15–0.5. Việc rà
   ảnh này ưu tiên hàng đầu là hợp lý vì đây cũng là ảnh tôi đã dùng cho Blind Scan độc lập
   (`reports/BLIND_SCAN.md`): tôi tự đếm 25 xe trước khi xem nhãn AI, trong khi AI chỉ đề xuất 28 box
   với 18 box không chắc chắn — khớp với việc ảnh này nhiều xe khó (xe xa, xe bị che).
2. **frame_0331.jpg** — hạng 5, score 0.9154, có **47 box gợi ý**, nhiều nhất trong cả lô 12 ảnh (xem
   contact sheet `outputs/selection_round1.jpg`, ảnh đông xe nhất). Đúng như dự đoán, đây cũng là ảnh
   tôi sửa nhiều nhất theo `outputs/round1_diff.md`: edited 3, deleted 5, added 17 — nhiều xe cần thêm
   và nhiều box trùng cần xoá vì cảnh quá đông xe.
3. **frame_0392.jpg** — hạng 15 (vẫn nằm trong lô 12 vì công cụ nới `MIN_GAP_S` khi thiếu ảnh), nhưng
   có **U=0.9747 — độ bất định cao nhất trong 50 dòng đầu**, cao hơn cả frame_0182. Điểm tổng của ảnh
   này thấp hơn (A chỉ 0.6667) nên xếp hạng 15, nhưng minh hoạ rằng ảnh có độ bất định gốc cao nhất
   không nhất thiết đứng đầu bảng xếp hạng cuối, vì score còn phụ thuộc tỉ lệ box mơ hồ (A).

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

**frame_0330.jpg** (hạng 12, score 0.8899) đứng trong top-12 theo điểm thô nhưng **không được chọn**
vào lô cuối. Lý do: nó cách `frame_0326` (đã chọn, hạng 4) chỉ 1.6 giây và cách `frame_0331` (đã chọn,
hạng 5) chỉ 0.4 giây — cả hai đều dưới `MIN_GAP_S = 2.0` giây, nên công cụ chọn lô (`tools/al_select.py`)
bỏ qua nó để tránh trùng lặp, và nhường chỗ cho `frame_0270` (hạng 13), `frame_0107` (hạng 14),
`frame_0392` (hạng 15) — đúng như bài mô tả: "ảnh được chọn không nhất thiết là 12 ảnh đứng đầu theo điểm".

Điều phép chọn này chưa chứng minh về chất lượng mô hình:

Điểm số chỉ đo **mức mô hình hiện tại chưa chắc chắn** (U, A) và mức đa dạng thời gian (D) — không đo
việc sửa nhãn cho các ảnh này có thực sự giúp mô hình tốt hơn trên tập test hay không. Thực tế vòng 1
cho thấy điều đó: sau khi sửa đúng 12 ảnh do chiến lược `uncertainty` chọn, AP50 lại **giảm** từ 0.771
xuống 0.646 (xem `reports/REPORT.md` mục 4). Ngoài ra, điểm cao cũng không loại trừ trường hợp ảnh quá
nhòe hoặc xe bị che gần hết khiến việc gán nhãn không chính xác/nhất quán, hay ảnh tuy bất định nhưng
nội dung không đại diện cho các tình huống mô hình sẽ gặp trong thực tế.
