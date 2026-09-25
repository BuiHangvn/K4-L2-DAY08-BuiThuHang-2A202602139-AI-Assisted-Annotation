# Vì sao chọn lô này?

Nếu chỉ có thời gian rà năm ảnh trong 50 dòng đầu của `outputs/selection_round1.csv`, tôi ưu tiên:

1. `frame_0182.jpg` — hạng 1, score 0.9591, thời điểm 72.8 giây.
2. `frame_0369.jpg` — hạng 2, score 0.9324, thời điểm 147.6 giây.
3. `frame_0380.jpg` — hạng 3, score 0.9170, thời điểm 152.0 giây.
4. `frame_0326.jpg` — hạng 4, score 0.9155, thời điểm 130.4 giây.
5. `frame_0099.jpg` — hạng 8, score 0.9063, thời điểm 39.6 giây.

Bốn ảnh đầu có score rất cao, còn `frame_0099.jpg` giúp mở rộng lô rà sang một thời điểm khác của video thay vì chỉ tập trung vào các frame có hạng liền nhau.

Tôi không ưu tiên `frame_0331.jpg` dù ảnh này đứng hạng 5 với score 0.9154, vì nó ở rất gần `frame_0326.jpg` về thời gian. Hai frame cách nhau đúng 2.0 giây nên vẫn đạt ngưỡng `MIN_GAP_S = 2`, nhưng với ngân sách chỉ năm ảnh, tôi ưu tiên dùng một lượt rà cho một thời điểm khác để giảm nguy cơ lặp lại cùng kiểu cảnh.

Trong lô 12 ảnh do chiến lược active learning chọn, tôi đối chiếu ba trường hợp:

- `frame_0182.jpg`: hạng 1, score 0.9591, thời điểm 72.8 giây, 28 box dự đoán và 18 box mơ hồ.
- `frame_0099.jpg`: hạng 8, score 0.9063, thời điểm 39.6 giây, 29 box dự đoán và 14 box mơ hồ.
- `frame_0107.jpg`: hạng 14, score 0.8876, thời điểm 42.8 giây, 33 box dự đoán và 15 box mơ hồ.

Cả ba đều có `selected=True` trong CSV. Score kết hợp mức độ bất định của mô hình, số lượng box mơ hồ và thành phần đa dạng theo thời gian. Khi tạo lô 12 ảnh, chiến lược còn áp dụng `MIN_GAP_S = 2` giây để hạn chế chọn nhiều frame quá gần nhau.

Một trường hợp đáng chú ý khác là `frame_0372.jpg`. Ảnh này có score 0.9101 và đứng hạng 6 nhưng có `selected=False`. Nó nằm ở thời điểm 148.8 giây, chỉ cách `frame_0369.jpg` ở 147.6 giây khoảng 1.2 giây, nhỏ hơn `MIN_GAP_S = 2`. Vì vậy, sau khi `frame_0369.jpg` được giữ lại, `frame_0372.jpg` bị loại dù score vẫn cao. Trường hợp này cho thấy chiến lược không chỉ lấy các ảnh có score cao nhất mà còn kiểm soát sự trùng lặp theo thời gian.

Chi phí rà nhãn cũng ảnh hưởng đến quyết định chọn mẫu. Các frame có nhiều xe nhỏ, xe mờ hoặc nhiều box mơ hồ có thể mang lại thông tin hữu ích nhưng đồng thời tốn nhiều thời gian kiểm tra và chỉnh box. Với ngân sách giới hạn, tôi ưu tiên các ảnh vừa có mức bất định cao vừa bổ sung bối cảnh khác biệt thay vì dùng nhiều lượt rà cho các frame gần giống nhau.

Cuối cùng, score cao chỉ cho biết mô hình hiện tại đang không chắc chắn hoặc có nhiều dự đoán cần xem lại. Nó không chứng minh rằng pre-label là sai, cũng không bảo đảm rằng việc sửa nhãn của ảnh đó sẽ làm mô hình tốt hơn sau fine-tune. Hiệu quả thực tế vẫn cần được đánh giá bằng kết quả vòng tiếp theo trên cùng tập test.