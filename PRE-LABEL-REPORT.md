# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: `[[ĐIỀN: mã nhóm/phòng]]`
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt). `[[ĐIỀN]]`
- Trạng thái: `[[ĐIỀN: executed-by-group / executed-on-room-LC-machine / provided-results]]` — *lưu ý: lượt chạy dưới đây được thực hiện qua Claude Code (AI assistant) trên máy của thành viên theo yêu cầu nhóm, không phải một người tự tay gõ lệnh; nhóm cần tự quyết định ghi trạng thái nào cho đúng và trao đổi với LC nếu không chắc.*
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: `[[ĐIỀN tên người chịu trách nhiệm lượt chạy]]`; 2026-10-01 ~07:49–07:50 UTC; Windows 11 + Docker Desktop (Linux containers), linux/amd64
- Image tag và image ID; phiên bản repo: `day13-pointpillars:student`; `sha256:c8279af3d2f8b064d811124d7f06b2f83b745f855623680d0e067c7a5e075650`; repo_revision `f5f1de01c98d34240847e3634b4c69fe7376d9db` (working tree sạch, không dirty)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `data/demo.pcd`, frame_id=`demo`; chạy trên máy nhóm; input_sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: front ROI, score threshold 0.3 (giữ cố định cả A/B/C theo cấu hình gói)
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance thật bị bỏ khỏi PCD (kênh thứ tư = 0, placeholder); z_ground=0.075 m ước lượng từ dữ liệu, không phải đo mặt đường thật

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ 1 hộp `vehicles`; ảnh Side cho thấy model gần như không bắt được các object khác khi z chưa dịch |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 13 hộp: 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; hộp bám sát điểm trên mặt đất (z dương, quanh 0.7–1.4 m) |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Còn 6 hộp, toàn bộ là `pedestrian`; mất hết `vehicles`/`two-wheels` so với B |

- A/B: thay input trước model **không** giống dịch cùng một hằng số cho output. Số hộp nhảy 1→13 và xuất hiện thêm 2 class mới (`pedestrian`, `two-wheels`) — chứng tỏ dịch z ở input làm thay đổi cách model nhìn thấy đối tượng (ảnh hưởng đến voxel hoá/pillar), chứ không đơn thuần tịnh tiến kết quả có sẵn.
- B/C: pillar thô hơn (0.16→0.32) làm số hộp giảm 13→6 và **mất hoàn toàn** 2 class `vehicles`, `two-wheels`, chỉ còn `pedestrian`. Chưa đủ bằng chứng để nói cấu hình nào "tốt hơn" — số hộp ít hơn có thể là mất đối tượng thật (recall thấp) chứ không phải lọc đúng nhiễu; `[[ĐIỀN thêm nếu nhóm đối chiếu ảnh Side kỹ hơn]]`.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? `[[ĐIỀN: quan sát của nhóm từ ảnh Side, ví dụ hộp ở rìa ROI có bị cắt/yaw khó đọc không]]`
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Không JSON nào trong 3 lượt này được import — đây là prediction thô trên PCD demo KITTI, không phải job Robotaxi; `[[ĐIỀN nếu nhóm có thêm nhận định]]`

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | — | Không đổi (baseline = prediction thật lượt B) | Không cần (không có lỗi) | `qc-cases/case-correct.json`, `qc-cases/side-correct.png` |
| case-batch-z | 13/13 | −1.805 m cho **toàn bộ** 13 hộp | Class/x/y/yaw giữ nguyên, chỉ z đổi | Dừng batch, kiểm transform/pipeline trước (lệch đồng loạt, cùng lượng, cùng chiều → nghi lỗi hệ quy chiếu/pipeline, không phải lỗi từng vật thể) | `qc-cases/case-batch-z.json`, `qc-cases/side-batch-z.png` |
| case-one-box-z | 1/13 (hộp `vehicles` đầu tiên) | −1.805 m chỉ hộp đó | Class/x/y/yaw giữ nguyên ở mọi hộp; 12 hộp còn lại không đổi | Kiểm riêng hộp đó qua nhiều góc nhìn/đối tượng, không dừng cả batch (chỉ 1/13 lệch, các hộp khác bình thường) | `qc-cases/case-one-box-z.json`, `qc-cases/side-one-box-z.png` |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction thật của lượt B (dịch z cố định −1.805 m), không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
