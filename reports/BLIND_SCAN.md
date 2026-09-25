# Quét độc lập trước khi xem pre-label

Frame: frame_0099.jpg

Số xe nhìn thấy bằng mắt: 22

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:

1. Xe ở xa, hình ảnh mờ và thân xe khó quan sát nên AI dễ bỏ sót hoặc xác định box không chính xác.

2. Xe bị che khuất một phần phía sau xe khác nên AI dễ bỏ sót hoặc vẽ box lệch so với phần xe thực sự nhìn thấy.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.