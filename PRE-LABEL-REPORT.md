# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Nhóm KPM / Phòng C305
- Thành viên: xem `TEAMMATES.md` — Lý Hồng Phúc (2A202602221), Nguyễn Công Khải (2A202602243), Tạ Văn Mạnh Đức (2A202602235), vai trò từng lượt A/B/C đã điền trong bảng.
- Trạng thái: `executed-by-group` — chạy thật trên máy nhóm (không phải `provided-results`), nhưng **cần nhóm xác nhận lại**: thao tác Docker/CLI được thực hiện qua Claude Code (AI assistant) theo yêu cầu của Lý Hồng Phúc, không phải tự tay gõ từng lệnh. Nếu LC yêu cầu nghiêm ngặt "tự tay vận hành", hãy nói rõ điều này với LC thay vì giữ nguyên nhãn `executed-by-group` mà không giải thích.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Lý Hồng Phúc (vai trò vận hành lượt A, có mặt khi chạy toàn bộ A/B/C) — hỗ trợ bởi Claude Code; 2026-10-01 ~07:49–07:50 UTC; Windows 11 + Docker Desktop (Linux containers), linux/amd64
- Image tag và image ID; phiên bản repo: `day13-pointpillars:student`; `sha256:c8279af3d2f8b064d811124d7f06b2f83b745f855623680d0e067c7a5e075650`; repo_revision `f5f1de01c98d34240847e3634b4c69fe7376d9db` (working tree sạch, không dirty)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `data/demo.pcd`, frame_id=`demo`; chạy trên máy nhóm; input_sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: `/opt/PointPillars/pretrained/epoch_160.pth`, sha256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window; score threshold: front ROI, score threshold 0.3 (giữ cố định cả A/B/C theo cấu hình gói)
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance thật bị bỏ khỏi PCD (kênh thứ tư = 0, placeholder); z_ground=0.075 m ước lượng từ dữ liệu, không phải đo mặt đường thật

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ 1 hộp `vehicles`; ảnh Side cho thấy model gần như không bắt được các object khác khi z chưa dịch |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 13 hộp: 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; hộp bám sát điểm trên mặt đất (z dương, quanh 0.7–1.4 m) |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | Còn 6 hộp, toàn bộ là `pedestrian`; mất hết `vehicles`/`two-wheels` so với B |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file khác ở `run-A/side-...png` vs `run-B/side-...png`: B xuất hiện thêm 2 class mới (`pedestrian`, `two-wheels`) không có ở A. Đây là chạy lại model trên input khác (dịch z trước inference), không chỉ dịch hộp cũ — số hộp nhảy 1→13 chứng tỏ dịch z ở input thay đổi cách model nhìn thấy đối tượng (ảnh hưởng voxel hoá/pillar), không phải tịnh tiến kết quả có sẵn; điều còn chưa chắc là `z_ground=0.075 m` chỉ ước lượng từ dữ liệu, chưa đo mặt đường thật.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file khác ở `run-B/side-...png` vs `run-C/side-...png`: C mất hoàn toàn `vehicles`/`two-wheels`, chỉ còn `pedestrian`. Đối chiếu toạ độ JSON (không chỉ nhìn ảnh): 5/6 hộp của C nằm cách một hộp `vehicles`/`two-wheels` cũ của B chưa tới 1.5 m (gần nhất 0.35 m) nhưng bị gán lại thành `pedestrian`; 1/6 hộp (x≈33.5, y≈-15.6) cách hộp gần nhất của B tới 8.4 m nên là phát hiện mới. Chưa đủ bằng chứng để kết luận cấu hình nào tốt hơn — số hộp ít hơn có thể là mất đối tượng thật (recall thấp) hoặc gán sai class, không phải lọc đúng nhiễu.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Nhìn `run-B/side-...png`, hộp gần (x<20 m) nằm trong vùng điểm dày nên khung hộp khớp sát biên điểm, dễ tin yaw/kích thước; hộp xa (x≈55–57 m, gần rìa ROI) chỉ phủ vài chục điểm thưa, khung hộp rộng hơn nhiều so với cụm điểm thật — yaw/kích thước vùng này nên coi là chưa chắc, cần ảnh Trên/Trước đối chiếu thêm.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Không JSON nào trong 3 lượt này được import — đây là prediction thô trên PCD demo KITTI, không phải job Robotaxi. Nếu áp dụng cách đọc này cho job Robotaxi thật, cần kiểm thêm ảnh camera cùng frame trước khi tin class, vì ở đây không có ảnh camera để đối chiếu.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | — | Không đổi (baseline = prediction thật lượt B) | Không cần (không có lỗi) | `qc-cases/case-correct.json`, `qc-cases/side-correct.png` |
| case-batch-z | 13/13 | −1.805 m cho **toàn bộ** 13 hộp | Class/x/y/yaw giữ nguyên, chỉ z đổi | Dừng batch, kiểm transform/pipeline trước (lệch đồng loạt, cùng lượng, cùng chiều → nghi lỗi hệ quy chiếu/pipeline, không phải lỗi từng vật thể) | `qc-cases/case-batch-z.json`, `qc-cases/side-batch-z.png` |
| case-one-box-z | 1/13 (hộp `vehicles` đầu tiên) | −1.805 m chỉ hộp đó | Class/x/y/yaw giữ nguyên ở mọi hộp; 12 hộp còn lại không đổi | Kiểm riêng hộp đó qua nhiều góc nhìn/đối tượng, không dừng cả batch (chỉ 1/13 lệch, các hộp khác bình thường) | `qc-cases/case-one-box-z.json`, `qc-cases/side-one-box-z.png` |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction thật của lượt B (dịch z cố định −1.805 m), không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

**Lý Hồng Phúc (2A202602221)** — vai trò: vận hành lượt A, xem hình học lượt B, kiểm JSON/cấu hình lượt C.
- Quan sát: `run-B/side-demo-delta-1.73-voxel-0.16.png` có 13 hộp bám các cụm điểm gần mặt đất (x≈0–30 m), trong khi `run-A` cùng PCD chỉ có 1 hộp.
- Diễn giải phép z: dịch +1.73 m vào input *trước* khi đưa vào model làm model nhận thêm 12 đối tượng mới, không phải cộng dồn 1.73 m vào từng hộp sẵn có của A — chứng tỏ đây là biến đổi input ảnh hưởng cả bước voxel hoá, không phải hậu xử lý tuyến tính trên output.
- Quyết định lỗi batch: `case-batch-z` có 13/13 hộp lệch đúng −1.805 m, cùng chiều, x/y/yaw/class giữ nguyên → dừng kiểm pipeline/transform trước, chưa sửa từng hộp.
- Chưa chắc: `z_ground=0.075 m` chỉ là ước lượng từ dữ liệu, không phải đo mặt đường thật, nên ranh giới "bình thường" so với "lệch" của z cần hỏi lại coach.

**Nguyễn Công Khải (2A202602243)** — vai trò: kiểm JSON/cấu hình lượt A, vận hành lượt B, xem hình học lượt C.
- Quan sát: `run-C/boxes-demo-delta-1.73-voxel-0.32.json` chỉ còn 6 hộp, toàn bộ là `pedestrian`; so với `run-B` (13 hộp, có cả `vehicles`/`two-wheels`/`pedestrian`), pillar thô hơn (0.32 so với 0.16) làm mất hẳn 2 class.
- Diễn giải phép z: so `run-A` (delta=0 → 1 hộp) với `run-B` (delta=1.73 → 13 hộp) cho thấy hiệu ứng không tuyến tính — không thể suy A từ B bằng cách trừ 1.73 m khỏi từng hộp, vì số lượng/đối tượng model phát hiện ra khác hẳn nhau.
- Quyết định lỗi batch: `case-one-box-z` chỉ 1/13 hộp (`vehicles` đầu tiên) lệch −1.805 m, 12 hộp còn lại giống hệt baseline `case-correct` → không dừng cả batch, chỉ kiểm riêng hộp đó qua nhiều góc nhìn.
- Chưa chắc: B→C mất `vehicles`/`two-wheels` có thể do pillar thô làm giảm recall thật, hoặc do điểm của các object đó vốn thưa; chưa đủ căn cứ để kết luận nguyên nhân chính.

**Tạ Văn Mạnh Đức (2A202602235)** — vai trò: xem hình học lượt A, kiểm JSON/cấu hình lượt B, vận hành lượt C.
- Quan sát: `run-A/side-demo-delta-0-voxel-0.16.png` chỉ có 1 hộp `vehicles` quanh x≈10–15 m dù point cloud có nhiều cụm điểm khác (có thể là xe, người đi bộ ở xa) — model gần như không bắt được gì khi z chưa dịch.
- Diễn giải phép z: checkpoint pretrained KITTI quen với hệ toạ độ gốc khác hệ nguồn của `demo.pcd`, nên cần dịch +1.73 m vào input trước inference (phép z "thuận" = sửa input cho khớp hệ toạ độ model quen, không phải sửa ngược trên output sau khi đã có kết quả).
- Quyết định lỗi batch: `case-correct` (baseline của B) không có hộp nào lệch (0/13) nên dùng làm mốc so sánh; khi cả batch lệch cùng lượng −1.805 m → xác định lỗi hệ thống (transform); khi chỉ 1 hộp lệch riêng lẻ → xác định lỗi cục bộ của hộp đó.
- Chưa chắc: gói Student không có ảnh camera đối chiếu (không có Robotaxi), nên không thể xác nhận class 100% đúng chỉ bằng hình học LiDAR.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
