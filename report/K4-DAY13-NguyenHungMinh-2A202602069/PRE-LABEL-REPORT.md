# Báo cáo thực hành PointPillars — Day 13

Bản nộp theo quy định Codelab VLearn Day 13.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân (Thực hiện một mình)
- Thành viên: xem `TEAMMATES.md` (Nguyễn Hùng Minh - MSSV: 2A202602069, đảm nhiệm tất cả các vai trò).
- Trạng thái: `executed-by-group` (tự chạy native trên máy Mac cá nhân).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Hùng Minh; 2026-10-02 09:52:04 (UTC 02:52:04); macOS Apple Silicon `arm64`.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-arm64` ; `sha256:dd6999ad5dd67962fdca18eb526132c08ce105980c9d89475d66cb1a5193f8a1` ; commit `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (frame `demo`); SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI có sẵn trong image: `/opt/PointPillars/pretrained/epoch_160.pth` ; SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window; score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh reflectance gốc bị lược bỏ, đặt RGB=0; `z_ground` được ước lượng từ đám mây điểm là `0.075 m`.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ phát hiện duy nhất 1 hộp `vehicles` ở x=13.15m với score thấp (0.32). Do không bù `delta=1.73m` (chiều cao sensor), phân phối z của PCD bị lệch so với dữ liệu huấn luyện của KITTI checkpoint, dẫn tới mô hình bỏ sót hầu hết các đối tượng. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | Phát hiện 13 hộp (10 vehicles, 1 two-wheels, 2 pedestrian) trải đều từ x=3.7m đến x=55.58m. Độ tin cậy rất cao (nhiều xe có score từ 0.81 đến 0.93). Cao độ sensor được bù đúng theo phân bố của KITTI. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Phát hiện 6 hộp nhưng toàn bộ là `pedestrian` (0 vehicles). Tăng kích thước pillar XY lên gấp đôi (0.32m) làm giảm độ phân giải không gian trên feature map 2D, các đặc trưng hình học của ô tô bị biến dạng/mất chi tiết, dẫn đến detector bỏ sót toàn bộ ô tô và dự đoán nhầm sang pedestrian. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh Side của A gần như trống trơn không bao được cụm điểm nào, trong khi ảnh Side của B bao phủ rõ ràng các cụm điểm xe ở nhiều cự ly. Đây là chạy lại model trên input khác (được chuẩn hóa lại cao độ z), không chỉ dịch hộp cũ; điều em còn chưa chắc là: ước lượng mặt phẳng đất `z_ground = 0.075m` là giá trị xấp xỉ toàn cảnh, địa hình xa hoặc gồ ghề có thể có sai lệch cục bộ.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Khi tăng kích thước pillar từ 0.16m lên 0.32m, toàn bộ 10 xe và 1 xe hai bánh biến mất, thay vào đó là 6 hộp pedestrian ở các vị trí khác nhau. Không đủ bằng chứng để kết luận C tốt hơn; thực tế C cho kết quả suy giảm nghiêm trọng do checkpoint được huấn luyện trên kích thước pillar 0.16m, việc thay đổi biểu diễn đầu vào làm sai lệch receptive field của mạng backbone.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Cửa sổ quét front-window chỉ lấy vùng phía trước xe, các đối tượng ngoài góc quan sát này bị cắt bỏ nên không tính là bỏ sót (miss). Góc nhìn Side là hình chiếu 2D x-z nên các xe có cùng khoảng cách x/z nhưng khác hoành độ y sẽ bị chồng chập lên nhau; không thể đánh giá chính xác góc xoay yaw hay phân biệt các xe chạy song song nếu chỉ nhìn riêng góc Side mà thiếu Top view, Front view và camera đối chiếu.
- JSON nào còn chưa đủ cơ sở để import? Cả ba file JSON A, B, C và các ca QC đều là dữ liệu KITTI minh họa, không có cơ sở và tuyệt đối không được import vào CVAT Robotaxi (vốn sử dụng hệ tọa độ, schema và khung hình Robotaxi riêng).

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 m | Không đổi | Giữ nguyên pipeline, kiểm tra từng đối tượng nếu cần | Bản sao nguyên vẹn của kết quả lượt B, các hộp bám khít cụm điểm LiDAR tương ứng. |
| case-batch-z | 13 / 13 | 1.805 m (bị trừ delta + z_ground) | Không đổi | **Dừng batch**, báo LC kiểm tra pipeline/phép biến đổi tọa độ | Toàn bộ 13/13 hộp bị chìm đồng loạt xuống dưới mặt đất đúng 1.805m, trong khi x, y, yaw và class không đổi. Đây là dấu hiệu điển hình của việc pipeline quên cộng ngược z sau inference. |
| case-one-box-z | 1 / 13 | 1.805 m | Không đổi | **Kiểm tra từng hộp**, không dừng batch | Chỉ duy nhất hộp đầu tiên bị lệch chìm z, 12 hộp còn lại vẫn đúng vị trí chuẩn. Đây là lỗi đối tượng cục bộ (có thể do nhiễu điểm hoặc lỗi gán nhãn đơn lẻ), không phải lỗi pipeline hệ thống. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Người thực hiện:** Nguyễn Hùng Minh (MSSV: 2A202602069).
- **Vai trò đã làm:** Do thực hiện cá nhân, em đảm nhiệm toàn bộ quy trình: vận hành Docker trên macOS ARM64 chạy runner gói student; kiểm tra tính toàn vẹn cấu hình trong `smoke.json` và các file `boxes-*.json`; đối chiếu hình chiếu x-z trong ảnh `side-*.png` và tổng hợp phân tích báo cáo.
- **Quan sát A/B/C:** Khi chạy lượt A (delta=0m), mô hình chỉ nhận diện được 1 hộp xe ở `run-A/boxes-demo-delta-0-voxel-0.16.json` (score=0.32). Khi chuyển sang lượt B (bù delta=1.73m theo chiều cao sensor KITTI), mô hình nhận diện chuẩn xác 13 hộp với độ tin cậy vượt trội. Khi chuyển sang lượt C (tăng pillar lên 0.32m), toàn bộ xe biến mất và bị dự đoán nhầm thành pedestrian do độ phân giải lưới đặc trưng bị co cụm quá mức.
- **Diễn giải phép đổi z thuận/ngược:**
  - Thuận: $z_{model} = z_{source} - z_{ground} - delta$ (đưa đám mây điểm về mặt phẳng tham chiếu mà mô hình được huấn luyện).
  - Ngược: $z_{source} = z_{model} + z_{ground} + delta$ (chuyển các bounding box dự đoán trở lại hệ tọa độ cảm biến ban đầu của xe).
  - Nếu pipeline bị lỗi quên phép ngược, toàn bộ bounding box sẽ bị dịch xuống dưới một khoảng đúng bằng $delta + z_{ground} = 1.73 + 0.075 = 1.805m$, đúng như trong `case-batch-z`.
- **Quyết định lỗi batch và hành động:** Khi quan sát thấy tất cả các hộp trong frame bị lệch cùng một khoảng cao độ z đồng nhất, hành động đúng đắn là **dừng ngay việc sửa tay trên CVAT**, báo cáo ngay cho LC/kỹ sư pipeline để kiểm tra lại code biến đổi tọa độ. Việc sửa tay từng hộp trong trường hợp lỗi hệ thống sẽ gây lãng phí thời gian và làm sai lệch dữ liệu.
- **Điều chưa chắc:** Việc xác định `z_ground = 0.075m` dựa trên thuật toán ước lượng mặt phẳng đất toàn cảnh của PCD mẫu; nếu địa hình thực tế mấp mô hoặc có độ dốc lớn, cao độ mặt đất cục bộ tại từng vị trí vật thể có thể sai khác đôi chút.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
