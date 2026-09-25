# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: BÙI THU HẰNG

Công cụ gán nhãn đã dùng: CVAT

## 1. Dữ liệu và cách chia tập

Video là một cảnh đường cao tốc ban đêm được quay từ camera cố định; cùng một xe có thể xuất hiện trong nhiều frame liên tiếp. Vì vậy, 268 ảnh chưa gán nhãn (pool) và 20 ảnh kiểm thử (test set) được chia theo trục thời gian, đồng thời có vùng đệm giữa hai tập. Ảnh trong pool gần ảnh test nhất vẫn cách 4.4 giây.

Nếu chia ngẫu nhiên, các frame rất gần nhau về thời gian, thậm chí cùng một chiếc xe với gần như cùng nền ảnh, có thể xuất hiện ở cả tập dùng để học và tập kiểm thử. Khi đó mô hình sẽ được đánh giá trên dữ liệu quá giống dữ liệu đã thấy trước đó, khiến kết quả test có xu hướng lạc quan hơn khả năng áp dụng cho các thời điểm mới. Chia theo thời gian và có vùng đệm giúp giảm rò rỉ thông tin giữa các tập và làm phép so sánh trước/sau fine-tune đáng tin cậy hơn.

## 2. Mô hình khởi đầu lạnh

Dòng vòng 0 trong `rounds_table.md` cho thấy mô hình là YOLOv8n khởi đầu từ trọng số COCO và chưa học ảnh hoặc box nào của bài này:

- AP50 = 0.771
- P@0.25 = 0.925
- R@0.25 = 0.489
- F1 = 0.640
- Recall small = 0.182
- Recall medium = 0.547
- Recall large = 0.561

Recall của xe nhỏ chỉ đạt 0.182, thấp hơn rõ rệt so với xe vừa và xe lớn. Điều này cho thấy ở trạng thái cold start, nhóm xe ở xa hoặc có kích thước box nhỏ là nhóm khó nhất đối với mô hình.

Trong `compare_round0.jpg`, tại `frame_0150`, mô hình khớp 10 trong 20 box tham chiếu và còn 10 false negative. Nhiều box không được khớp nằm ở các xe nhỏ, xa hoặc khó quan sát trong cảnh đêm.

Tuy nhiên, không nên coi mọi khác biệt giữa dự đoán và nhãn tham chiếu là lỗi chắc chắn của mô hình. Ví dụ, với các cụm đèn xe rất xa hoặc ánh phản chiếu trên mặt đường, cần mở ảnh gốc và kiểm tra lại guideline trước khi kết luận. Nhãn test do một mô hình khác tạo và chưa được người rà thủ công từng box, vì vậy một box tham chiếu vẫn có thể vẽ lệch thân xe hoặc khoanh nhầm vùng sáng.

## 3. Chiến lược chọn mẫu

Mỗi ảnh trong pool được tính điểm theo công thức:

`score = 0.5·U + 0.3·A + 0.2·D`

Trong đó:

- `U` biểu diễn mức độ bất định của mô hình ở những box khó nhất trong ảnh.
- `A` phản ánh số lượng box có confidence nằm trong vùng mơ hồ, sau đó được chuẩn hóa so với ảnh có nhiều box mơ hồ nhất.
- `D` khuyến khích chọn các thời điểm khác biệt về mặt thời gian so với những ảnh đã được gán nhãn.

Ở vòng đầu chưa có ảnh đã gán nên `D = 1` cho tất cả ảnh. Ngoài score, chiến lược còn sử dụng `MIN_GAP_S = 2` giây để tránh chọn nhiều frame gần như trùng nhau trong cùng một lô.

Theo `SELECTION.md` và `selection_round1.csv`:

- `frame_0182.jpg`: score = 0.9591, có 18 box mơ hồ.
- `frame_0099.jpg`: score = 0.9063, có 14 box mơ hồ.
- `frame_0107.jpg`: score = 0.8876, có 15 box mơ hồ.

Cả ba frame trên đều nằm trong 12 ảnh được chọn cho vòng 1.

Ngược lại, `frame_0372.jpg` có score = 0.9101 nhưng không được chọn vì chỉ cách `frame_0369.jpg` khoảng 1.2 giây, nhỏ hơn `MIN_GAP_S`. Việc loại bớt frame gần trùng giúp tránh dùng ngân sách gán nhãn cho nhiều cảnh chứa gần như cùng thông tin.

Điểm bất định cao không chứng minh rằng việc gán nhãn ảnh đó chắc chắn sẽ cải thiện mô hình. Nó chỉ cho biết mô hình hiện tại chưa chắc chắn ở ảnh đó. Hiệu quả sau fine-tune còn phụ thuộc vào chất lượng nhãn, mức độ đa dạng của lô ảnh, độ trùng lặp giữa các frame và việc các trường hợp được chọn có đại diện cho lỗi của mô hình trên dữ liệu rộng hơn hay không.

## 4. Các vòng học chủ động

Bảng dưới đây được lấy từ `rounds_table.md`. Precision, Recall và F1 được tính ở ngưỡng confidence 0.25 trên cùng 20 ảnh test.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vòng 1 | 12 | 323 | 0.625 | -0.146 | 1.000 | 0.122 | 0.217 | 0.000 | 0.085 | 0.585 |

Ở vòng 0 chưa có nhãn AI nào được con người rà sửa.

Ở vòng 1, `round1_diff.md` ghi nhận AI ban đầu gợi ý 169 box trên 12 ảnh. Sau khi rà trên CVAT, tập nhãn cuối có 323 box, gồm:

- 153 box giữ nguyên
- 6 box chỉnh sửa
- 10 box xoá
- 164 box thêm mới

Như vậy, phần nhãn cuối khác khá nhiều so với pre-label ban đầu, đặc biệt do số lượng box được thêm mới lớn. Điều này cho thấy mô hình gợi ý ban đầu bỏ sót nhiều đối tượng mà người rà cho rằng cần được gán nhãn theo guideline.

Sau fine-tune, AP50 giảm từ 0.771 xuống 0.625, tức giảm 0.146 so với cold start. Vì đây mới là vòng active learning đầu tiên nên mức thay đổi này cũng chính là mức thay đổi so với vòng trước.

Precision tăng từ 0.925 lên 1.000, nhưng recall chung giảm mạnh từ 0.489 xuống 0.122 và F1 giảm từ 0.640 xuống 0.217. Vì vậy, precision cao hơn không đồng nghĩa với mô hình tổng thể tốt hơn: mô hình sau fine-tune trở nên thận trọng hơn, ít tạo false positive ở ngưỡng chấm nhưng đồng thời bỏ sót nhiều xe hơn.

Theo kích thước xe:

- Recall small giảm từ 0.182 xuống 0.000.
- Recall medium giảm từ 0.547 xuống 0.085.
- Recall large tăng nhẹ từ 0.561 lên 0.585.

Như vậy, nhóm xe lớn không suy giảm và còn tăng nhẹ, trong khi xe nhỏ và xe vừa bị ảnh hưởng rõ rệt nhất.

Trên `compare_round1.jpg`, `frame_0150` là một ví dụ rõ. Ở cold start, frame này có 10 TP và 10 FN; sau fine-tune chỉ còn 1 TP và 19 FN. Nhiều xe từng được mô hình cold start phát hiện không còn dự đoán khớp ở ngưỡng đánh giá sau vòng 1. Kết quả này cho thấy cần kiểm tra confidence của dự đoán sau fine-tune và chất lượng/phân bố của 323 box huấn luyện trước khi tiếp tục train, thay vì mặc định rằng active learning luôn giúp mô hình tốt hơn.

`BLIND_SCAN.md` ghi lại quan sát độc lập trước khi tôi xem pre-label. Với `frame_0099.jpg`, tôi đếm 22 xe và ghi nhận hai trường hợp dễ bị AI bỏ sót hoặc vẽ sai: xe ở xa, hình ảnh mờ và thân xe khó quan sát; và xe bị che khuất một phần phía sau xe khác.

`REVIEW_LOG.csv` ghi lại các thao tác tôi thực hiện khi rà pre-label, gồm các quyết định giữ, thêm, xoá hoặc chỉnh box. `round1_diff.md` là nguồn dùng để thống kê định lượng số box retained/edited/deleted/added giữa pre-label và nhãn cuối, còn `compare_round1.jpg` và các metric là bằng chứng về hành vi của mô hình sau khi fine-tune.

Trong quá trình kiểm tra, một số thao tác được ghi là `edited` trong `REVIEW_LOG.csv` ở `frame_0187` và `frame_0312`, trong khi bảng diff theo từng frame không ghi hai trường hợp đó là `edited`. Vì vậy, tôi không sử dụng hai dòng này để suy ra số lượng chỉnh sửa định lượng; các con số tổng hợp retained/edited/deleted/added trong báo cáo được lấy trực tiếp từ `round1_diff.md`. Sự khác biệt này cũng cho thấy cần phân biệt giữa log thao tác thủ công của người rà và kết quả đối chiếu box tự động của script.

Một trường hợp khó theo guideline là xe bị che bởi xe khác. Khi đó chỉ vẽ box cho phần thân xe thực sự nhìn thấy, không mở rộng box sang phần bị khuất. Một trường hợp khác là xe ở rất xa chỉ còn hai chấm đèn; nếu box cao dưới khoảng 16 pixel thì việc gán hay không gán đều được chấp nhận và trường hợp này được bỏ qua khi chấm. Điều quan trọng là áp dụng cùng một cách xử lý nhất quán trong toàn bộ lô.

## 5. Kết luận và giới hạn

Tôi dừng ở vòng 1 để kiểm tra lại dữ liệu và dự đoán trước khi cho mô hình học thêm. Sau fine-tune, AP50 giảm từ 0.771 xuống 0.625; recall chung giảm từ 0.489 xuống 0.122, trong đó recall của xe nhỏ giảm xuống 0.000 và xe vừa giảm xuống 0.085. Với kết quả này, tiếp tục train ngay mà chưa QC lại dữ liệu có thể làm khó xác định nguyên nhân của mức suy giảm.

Nếu thực hiện vòng tiếp theo, tôi ưu tiên hai nhóm trường hợp:

1. Xe ở xa, hình ảnh mờ hoặc chỉ còn cụm đèn. Đây là nhóm dễ bị bỏ sót nhất nhưng cũng tốn công rà vì biên thân xe khó xác định.
2. Xe bị che khuất, đứng sát xe khác hoặc nằm sát mép ảnh. Đây là các trường hợp dễ bị gộp box, cắt box hoặc xác định sai phần thân nhìn thấy.

Tuy nhiên, việc chọn ảnh tiếp theo cũng phải tính đến chi phí rà nhãn. Với video từ camera cố định, nhiều frame gần nhau có thể chứa cùng xe và gần như cùng bối cảnh. Chọn quá nhiều ảnh gần trùng sẽ làm tăng công gán nhãn nhưng không nhất thiết cung cấp thêm nhiều thông tin mới cho mô hình.

Kết quả hiện tại còn một số giới hạn. Tập test chỉ có 20 ảnh nên một số ít thay đổi trong dự đoán có thể ảnh hưởng khá rõ đến metric. Ngoài ra, 14 box tham chiếu có chiều cao dưới 16 pixel được bỏ qua khi chấm, nên metric không phản ánh đầy đủ khả năng phát hiện tất cả xe cực nhỏ ở xa. Nhãn tham chiếu của test cũng được tạo bởi mô hình khác và chưa được người rà thủ công, vì vậy mức khớp với reference không thể được coi là chân lý tuyệt đối về chất lượng ngoài thực tế.

Nếu AP50 giảm như ở vòng 1, trước khi train thêm tôi sẽ:

- kiểm tra lại các box đã thêm, xoá hoặc chỉnh trong Round 1 theo guideline;
- kiểm tra tính nhất quán của cách vẽ box giữa 12 ảnh;
- xem lại các trường hợp tụt mạnh như `frame_0150`;
- kiểm tra confidence của các dự đoán sau fine-tune;
- so sánh precision, recall và recall theo kích thước xe thay vì chỉ nhìn AP50;
- kiểm tra xem 12 ảnh active learning có quá tập trung vào một kiểu cảnh hoặc chứa nhiều frame gần trùng hay không.

Chỉ sau khi hoàn thành các bước QC này mới nên quyết định tiếp tục thêm một vòng active learning.