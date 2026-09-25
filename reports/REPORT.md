# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Đặng Trường Huy

Công cụ gán nhãn đã dùng: CVAT (Docker, chạy cục bộ tại máy cá nhân, phiên bản v2.76.0)

## 1. Dữ liệu và cách chia tập

Camera trong video đứng yên và mỗi xe ở lại trong khung hình vài giây, trong khi hai ảnh liên tiếp chỉ
cách nhau 0.4 giây (2.5 khung hình/giây) là gần như giống hệt nhau. Nếu chia ngẫu nhiên, rất có thể
cùng một chiếc xe xuất hiện ở cả tập huấn luyện (pool) lẫn tập kiểm thử (test) — chỉ khác vài khung
hình. Khi đó mô hình được chấm điểm trên chính chiếc xe nó đã "nhìn thấy" lúc huấn luyện, số đo AP50 sẽ
bị đẩy cao hơn khả năng thật của mô hình trên xe hoàn toàn mới. Đây là rò rỉ dữ liệu (data leakage), và
số đo bị lệch theo hướng **lạc quan giả** — trông tốt hơn thực tế.

Vì vậy bài chia theo trục thời gian: 20 ảnh test lấy từ 4 đoạn quanh giây 20/60/100/140, có vùng đệm 4
giây trước/sau mỗi đoạn bị loại bỏ hoàn toàn (không dùng cho pool lẫn test). Ảnh pool gần ảnh test nhất
vẫn cách 4.4 giây — đủ xa để một chiếc xe di chuyển qua khỏi khung hình, giảm khả năng cùng một xe xuất
hiện ở cả hai tập.

## 2. Mô hình khởi đầu lạnh (cold start)

Từ `reports/rounds_table.md`, dòng vòng 0:

| vòng | model | ảnh train | box train | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Độ phủ (recall) theo kích thước cho thấy mô hình cold start bỏ sót rất nhiều **xe nhỏ** (recall chỉ
0.182 — bỏ sót hơn 4/5 số xe nhỏ), khá hơn ở xe vừa (0.547) và xe lớn (0.561) nhưng vẫn bỏ sót gần một
nửa. Nhìn `outputs/compare_round0.jpg`: ở cả 4 ảnh test mẫu (frame_0050, frame_0150, frame_0250,
frame_0350), các box màu vàng (FN — bỏ sót) tập trung ở dải phía trên ảnh, nơi xe ở xa và nhỏ, đúng
khớp với số liệu recall theo kích thước. Các box đỏ (FP — nhận nhầm) thường nằm ở vùng có vệt sáng đèn
pha hoặc ánh đèn phản chiếu trên mặt đường, ví dụ ở frame_0050 và frame_0350 có 2 box đỏ mỗi ảnh nằm
sát các cụm đèn sáng — giống lỗi mà `GUIDELINE_LABEL.md` cảnh báo (không tính vệt sáng đèn pha là xe).

Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai: ở góc trên-trái của
`frame_0350` (reference: 23 box), có một cụm nhiều box nhỏ sát nhau ứng với các xe ở xa chỉ còn thấy
chấm đèn — ranh giới giữa "một xe" và "hai xe đứng cạnh nhau" ở khoảng cách này rất khó phân biệt bằng
mắt thường ngay cả với người, nên số lượng box tham chiếu ở cụm này nên được một người khác rà lại
trước khi khẳng định các box đỏ/vàng quanh đó là lỗi của mô hình chứ không phải nhãn tham chiếu đếm dư.

## 3. Chiến lược chọn mẫu

Công thức `score = W_U·U + W_A·A + W_D·D` (W_U=0.5, W_A=0.3, W_D=0.2) kết hợp ba tín hiệu: **U** — mức
mô hình chưa chắc chắn, lấy trung bình từ tối đa 5 box có độ bất định cao nhất trong ảnh; **A** — tỉ lệ
box có độ tin cậy mơ hồ (0.15–0.5) so với ảnh có nhiều box mơ hồ nhất trong tập; **D** — mức đa dạng
thời gian, bằng khoảng cách tới ảnh đã chọn gần nhất (tối đa 10 giây, chia 10 — ở vòng đầu D=1 cho mọi
ảnh vì chưa có ảnh nào được chọn). Ảnh mô hình không đoán được box nào còn được cộng `EMPTY_BONUS` để
không bị bỏ qua hoàn toàn. `MIN_GAP_S = 2.0` giây là bộ lọc chống trùng lặp: khi chọn lô theo thứ tự
score giảm dần, công cụ bỏ qua mọi ảnh cách một ảnh đã chọn dưới 2.0 giây (nếu không đủ 12 ảnh mới nới
khoảng cách này) — vì camera đứng yên nên hai ảnh quá gần nhau gần như là cùng một cảnh.

Ba ảnh trong lô (`reports/SELECTION.md`) minh hoạ cách cân nhắc: **frame_0182** (hạng 1, U=0.9182 cao
nhất) được ưu tiên vì mô hình bất định nhất ở đây; **frame_0331** (hạng 5, 47 box gợi ý — nhiều nhất
lô) cho thấy điểm cao còn phản ánh cảnh phức tạp, tốn công sửa hơn; **frame_0392** (hạng 15, U=0.9747
— cao hơn cả frame_0182) cho thấy bất định gốc cao không đảm bảo thứ hạng cao nếu tỉ lệ box mơ hồ (A)
thấp. Một ảnh khác, **frame_0330** (hạng 12, score 0.8899, đứng trong top-12 theo điểm thô) bị loại
khỏi lô cuối vì cách `frame_0326` chỉ 1.6s và `frame_0331` chỉ 0.4s (dưới MIN_GAP_S) — minh hoạ rằng
điểm bất định cao có thể bị "phạt" bởi ràng buộc chống trùng lặp.

Điểm bất định **không** chứng minh sửa ảnh đó sẽ cải thiện mô hình — nó chỉ cho biết mô hình hiện tại
đang mơ hồ ở đâu, tức là nơi *cần con người xem xét*, không phải nơi *chắc chắn giúp mô hình tốt lên*.
Kết quả vòng 1 minh chứng rõ điều này: đúng lô 12 ảnh có điểm bất định cao nhất được sửa kỹ, nhưng AP50
lại giảm (mục 4) — cho thấy độ bất định của mô hình và mức cải thiện sau khi học thêm là hai thứ khác
nhau, còn phụ thuộc vào cách huấn luyện, số lượng ảnh và tính nhất quán của nhãn.

## 4. Các vòng học chủ động (active learning)

Từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 332 | 0.646 | -0.125 | 1.000 | 0.166 | 0.285 | 0.000 | 0.142 | 0.610 |

Mức độ sửa nhãn gợi ý (`outputs/round1_diff.md`, 12 ảnh, model đề xuất 169 box, sau khi sửa còn 332
box): **accepted 143**, **edited 13**, **deleted 13** (FP của model), **added 176** (FN của model),
accept rate 85%. Số box `added` (176) lớn hơn cả tổng box AI đề xuất ban đầu (169) — model bỏ sót rất
nhiều xe, khớp với recall thấp đã thấy ở vòng 0.

AP50 vòng 1 **giảm** 0.125 so với cold start (0.771 → 0.646), không tăng. Precision lên tuyệt đối 1.000
(không còn FP) nhưng recall sập từ 0.489 xuống 0.166. Theo kích thước: recall xe nhỏ giảm từ 0.182 về
**0.0** (không còn nhận được xe nhỏ nào), xe vừa giảm từ 0.547 xuống 0.142, chỉ xe lớn nhích nhẹ từ
0.561 lên 0.610. Nhìn `outputs/compare_round1.jpg`, ví dụ `frame_0050`: cold start có TP 11/FP 2/FN 7,
vòng 1 chỉ còn TP 4/FP 0/FN 14 — mô hình sau fine-tune bỏ sót nhiều xe hơn hẳn, dù không còn box sai nào
(có thể kiểm bằng cách đếm box xanh/đỏ/vàng trực tiếp trên ảnh này).

Phân biệt ba nguồn thông tin: **Blind Scan** (`BLIND_SCAN.md`) là quan sát độc lập của tôi trên
`frame_0182` *trước khi* xem nhãn AI — tôi đếm 25 xe và dự đoán trước hai điểm khó (xe ra khỏi mép ảnh,
xe xa chỉ còn chấm sáng); đây là góc nhìn con người chưa bị ảnh hưởng bởi AI. **REVIEW_LOG.csv** ghi lại
lỗi thật đã sửa sau khi xem nhãn AI, ví dụ tại `frame_0227`: thêm box cho "xe nhỏ/tối gần giữa ảnh, chỉ
thấy cụm đèn pha mờ" (added — lỗi bỏ sót của AI), xoá "box AI trùng lên xe tải/van lớn" (deleted — lỗi
nhân đôi), sửa "box lệch không ôm sát nóc và đuôi xe tải/van" (edited — lỗi khung lệch). **round1_diff**
là kết quả định lượng do công cụ so sánh hai bộ nhãn tạo ra (143/13/13/176), còn **metrics_round1.json**
là kết quả của mô hình sau khi học từ các nhãn đó trên tập test — ba loại thông tin này độc lập với
nhau: quan sát người, nhật ký sửa, và số đo mô hình.

Một ca khó xử lý theo `GUIDELINE_LABEL.md`: tại `frame_0227`, có xe tải/van bị xe khác che một phần —
theo quy tắc "xe bị xe khác che một phần → chỉ vẽ box cho phần nhìn thấy", tôi chỉ vẽ box ôm phần thân
xe còn quan sát được, không ước lượng phần bị che, để giữ nhất quán với cách xử lý ở các ảnh khác trong
lô.

## 5. Kết luận và giới hạn

So với cold start, vòng 1 có kết quả **kém hơn** trên AP50 (0.771 → 0.646) dù precision hoàn hảo. Tôi
đề xuất **dừng lại kiểm tra thay vì lập tức chạy vòng 2**: sự sụt giảm 0.125 lớn hơn nhiều so với mức
nhiễu bài lưu ý (dưới ~0.01 với 20 ảnh test), nên nhiều khả năng đây là hiện tượng thật (có thể do
fine-tune 50 epoch trên chỉ 12 ảnh/332 box khiến mô hình học lệch, quên bớt kiến thức pretrained cho xe
nhỏ) chứ không phải nhiễu đo lường — cần xem lại cấu hình huấn luyện trước khi gán nhãn thêm cho vòng 2.

Nếu làm vòng 2, hai trường hợp còn khó nên ưu tiên: (1) **xe nhỏ/xa chỉ còn chấm đèn** — recall nhóm
này đã về 0, cần thêm nhiều ảnh có xe nhỏ đa dạng vị trí để mô hình học lại, nhưng công sửa cao vì mỗi
ảnh có hàng chục box nhỏ khó phân định (như `frame_0331` với 47 box); (2) **xe bị che một phần** — cần
nhất quán theo guideline, nhưng dễ bị đánh giá sai nếu người gán nhãn khác nhau xử lý khác nhau. Cả hai
đều có nguy cơ ảnh gần trùng cao vì pool lấy mẫu 2.5 khung hình/giây — cần áp dụng `MIN_GAP_S` khi chọn
ảnh vòng 2 để không lặp lại việc rà nhiều ảnh gần như giống hệt nhau (đã thấy ở mục 3 với cụm quanh
t=130–132s và t=147–153s).

Tập kiểm thử chỉ có 20 ảnh (403 box sau khi bỏ 14 box cao dưới 16px) là một mẫu rất nhỏ, nên số đo dễ
dao động mạnh khi chỉ vài box đổi trạng thái TP/FP/FN — kết luận rút ra khó tổng quát hoá chắc chắn cho
toàn bộ video hay điều kiện khác. Quy tắc bỏ qua xe cao dưới 16px giúp tránh tranh cãi ở nhóm khó gán
nhãn nhất quán, nhưng cũng nghĩa là mô hình không được thưởng/phạt gì cho việc xử lý nhóm này, dù đây
lại là nhóm chiếm phần lớn lỗi bỏ sót ở kích thước "nhỏ" đã ghi nhận. Nhãn tham chiếu do một mô hình
khác tạo ra và **chưa được người rà từng box** (mục 2), nên AP50 đo mức khớp với bộ tham chiếu này, chưa
chắc là mức chính xác tuyệt đối so với người. Trước khi cho mô hình học thêm, tôi sẽ kiểm tra: (a) cấu
hình huấn luyện trên Colab (số epoch, learning rate — có thể 50 epoch trên 12 ảnh là quá nhiều), (b)
tính nhất quán giữa các box tôi vừa sửa (có ảnh nào xử lý khác cách với ảnh khác không), và (c) một vài
box tham chiếu nghi ngờ đã nêu ở mục 2 trước khi kết luận chắc chắn mô hình "học tệ hơn".
