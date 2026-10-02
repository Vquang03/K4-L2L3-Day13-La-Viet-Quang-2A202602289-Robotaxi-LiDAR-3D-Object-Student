# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: 
- Thành viên: Lã Việt Quang.
- Trạng thái: `executed-on-room-LC-machine`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: dữ liệu thực tế trong `smoke.json` cho thấy chạy đúng trong container Linux `amd64`, không có thông tin người và thời gian cá nhân ở đây.
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`
  - Working tree dirty: `true` (có thay đổi trong repo khi chạy)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - `frame_id`: `demo`
  - dataset: `KITTI`
  - input SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp:
  - Path: `/opt/PointPillars/pretrained/epoch_160.pth`
  - SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: không có score threshold riêng trong output; chạy với mặc định của model/image.
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - `z_ground = 0.075 m`
  - `height_offset_m = 1.805 m`

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ có 1 hộp xuất hiện; `mean_z` thấp nhất trong ba lượt. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | Sau khi đổi `delta`, box count tăng lên 13 và xuất hiện nhiều đối tượng hơn rõ rệt so với A. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Khi đổi pillar size sang 0.32, số hộp giảm xuống 6; đây là biến đổi phân bố không gian rõ rệt. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. `run-A/summary.csv` có `n_boxes=1`, `mean_z=0.330`; `run-B/summary.csv` có `n_boxes=13`, `mean_z=1.034`. Dữ liệu cho thấy B không chỉ là “dịch hộp cũ” mà là một prediction mới với nhiều đối tượng và phân bố khác; trong side-view, vùng phía trước và phạm vi x của đối tượng rộng hơn đáng kể.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. `summary.csv` cho thấy `n_boxes=13` ở B và `n_boxes=6` ở C, với `mean_z` lần lượt 1.034 và 1.091. Số lượng, lớp và vị trí thay đổi rõ; điều này cho thấy cỡ voxel ảnh hưởng đáng kể đến kết quả. Có đủ bằng chứng để kết luận “tốt hơn” không? Chưa đủ, vì thiếu ground truth; ta chỉ có thể nói model đã thay đổi phân bố và số hộp khi thay đổi `voxel_size`.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Side image chỉ quan sát một mặt phẳng, nên các đối tượng ở xa hoặc che khuất có thể bị bỏ sót hoặc trông “khác” do góc nhìn và giới hạn front-window.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Chỉ các file `boxes-*.json` đã chạy từ prediction của model; chưa có ground truth để khẳng định “đúng”. Về mặt QC, cần kiểm `x/y/z`, `label`, `score`, và `yaw` để phân biệt lỗi hệ thống với bản chất của input.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | 0 m | Không | Không cần dừng batch; đây là bản copy gốc chưa chỉnh sửa | `qc-cases/manifest.json` mô tả `correct` là “Unmodified source-frame prediction copy” và `source_prediction_sha256` khớp với `run-B` (`c2a8db247353f0ef00299ff50acff997b7b4a87bdcfcefe792815acb5652cc80`). |
| case-batch-z | 13/13 | -1.805 m trên mọi hộp | Không; class/x/y/yaw không đổi, chỉ `z` bị shift | Rõ ràng là lỗi batch, không phải sai ngẫu nhiên | `qc-cases/manifest.json` ghi: “Every box z is shifted down by delta + z_ground”; `delta + z_ground = 1.73 + 0.075 = 1.805 m`, và các giá trị `z` trong `case-batch-z.json` đều thấp hơn `case-correct.json` khoảng 1.805 m. |
| case-one-box-z | 1/13 | -1.805 m ở hộp đầu tiên | Không; các hộp còn lại giữ nguyên; class/x/y/yaw không đổi | Rõ là lỗi cục bộ/điểm đối tượng, cần kiểm từng hộp | `qc-cases/manifest.json` mô tả: “Only the first box z is shifted down by delta + z_ground”. Trong `case-one-box-z.json`, chỉ box đầu tiên có `z` lệch ~1.805 m; 12 box còn lại không đổi. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

- Học viên: Tôi chỉ phân tích kết quả đã có trong thư mục `ket-qua-ca-nhan`. Quan sát chính: `run-A/summary.csv` có `n_boxes=1`, `run-B` có `n_boxes=13`, `run-C` có `n_boxes=6`; `mean_z` tương ứng là `0.330`, `1.034`, `1.091`. Điều này cho thấy khi thay `delta` hay `voxel_size`, output không đổi đơn thuần mà thay đổi số lượng đối tượng và phân bố z. Tôi ghi rõ rằng `case-batch-z` là lỗi batch vì mỗi hộp bị dịch xuống `1.805 m`, còn `case-one-box-z` là lỗi cục bộ vì chỉ hộp đầu tiên lệch, nên không được dùng làm nhãn/ground-truth.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: theo output `smoke.json`, chạy đúng trên frame `demo` trong vùng front-window, không có dữ liệu LC riêng được chèn vào report này.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: đã có output thực tế (`run-A/B/C` và `qc-cases`), không cần chạy thêm nửa nếu LC đã xác nhận `smoke.json` passed.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: `smoke.json` trạng thái `passed`; các ca lỗi được đặt trong `qc-cases` vì mục đích đào tạo, không dùng để import CVAT.
- Nhận xét từng thành viên và quyết định dừng pipeline: cần thêm nhận xét cá nhân tùy từng học viên; pipeline đã được dừng sau khi QC cases hoàn tất và `smoke.json` chứng nhận passed.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: nếu thực hiện huấn luyện/điều chỉnh tiếp, nên bắt đầu từ `run-B` như prediction gốc và kiểm định bằng các ca QC này. Chưa có ground truth để kết luận chính xác hơn, nên chỉ có thể so sánh và xác định sai lệch z có chủ đích. 
