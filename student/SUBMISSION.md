# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Pham Minh Hieu
- MSSV: 2A202602630
- Email:
- Link repo (fork): https://github.com/hieulovecat/K4-L3-Day23-PhamMinhHieu-2A202602630
- Commit hash nộp (`git rev-parse HEAD`): nộp trên LMS (hash của commit cuối cùng trước deadline)

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: `compare`, `[0, 198]` (199 frame), `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`, `seed = 0`
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: precision = 0.9701, recall = 0.7004, tp/fp/fn = 519 / 16 / 222
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.1503 m, 502, 11.3436, 0, 239, 2.5226
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.1359 m, 502, 9.2668, 0, 239, 2.5226
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: xem bảng và phân tích bên dưới.

| Mode | RMSE (m) | matches | ghost | miss | precision_track | coverage |
|---|---|---|---|---|---|---|
| lidar | 0.1503 | 502 | 0 | 239 | 502/502 = 1.000 | 502/519 = 0.967 |
| fused | 0.1359 | 502 | 0 | 239 | 502/502 = 1.000 | 502/519 = 0.967 |

**Phân tích.**

- **Hai mode có cùng tập track.** `matches`, `ghost_track_frames`, `missed_gt_frames`
  và `mean_confirmed_tracks` trùng khít. Đối chiếu từng record trong `grade_run.log`,
  các trường `det_tp/det_fp/det_fn/valid_gt/confirmed/matches/ghosts/misses` giống
  nhau ở **cả 199 frame**. Chỉ `sum_sq_err` là khác. Như vậy RMSE của hai mode được
  tính trên **cùng 502 cặp ghép**, nên so sánh hai con số RMSE là công bằng: không
  có chuyện fused "đẹp" hơn vì bỏ bớt cặp khó.
- **Camera giúp giảm sai số vị trí.** `sum_sq_err` giảm từ 11.34 xuống 9.27 (khoảng −18%).
  RMSE giảm từ 0.150 m xuống 0.136 m (−0.014 m), thoả điều kiện
  `rmse_fused − rmse_lidar ≤ 0.05 m`. Xét theo từng frame có cặp ghép, fused có
  `sum_sq_err` thấp hơn ở 137 frame và cao hơn ở 58 frame. Ví dụ frame 150:
  lidar 0.0761, fused 0.0327. Log chỉ ghi tổng sai số 3D nên không tách được theo
  trục. Về lý thuyết, đo pinhole 2D chỉ ràng buộc hướng nhìn (ngang và đứng), còn
  độ sâu gần như không quan sát được, nên phần cải thiện phải đến từ hai hướng đó.
- **Ghost = 0** ở cả hai mode. Detector có 16 FP, nhưng một track mới cần khoảng
  4 hit LiDAR liên tiếp mới được xác nhận (score 1/6 → 5/6 > 0.8). Vì vậy FP đơn lẻ
  không bao giờ trở thành track confirmed.
- **Miss = 239 không phải do tracker.** Detector bỏ sót 222 xe (det_fn). Phần còn lại
  là các frame khởi động: frame 0–3 có `confirmed = 0` vì track chưa đủ score, đến
  frame 4 mới confirm 2 track. Thêm vài frame khi một xe mới xuất hiện. Vì vậy chỉ số
  công bằng cho tracker là `coverage = matches / det_tp = 0.967`, không phải
  `matches / valid_gt`.
- **Giới hạn.** Đo camera là tâm hộp 2D **ground-truth** của camera FRONT cộng nhiễu
  N(0, 0.5 px) theo seed ([run_lab.py](../platform/fusion_lab/scripts/run_lab.py) `_front_observations`).
  Không có camera detector nào chạy cả. Vì vậy mức cải thiện RMSE ở đây là cận trên
  lạc quan, không chứng minh chất lượng một hệ perception camera thật. Ngoài ra,
  EKF dùng `R` với σ = 5 px, lớn hơn nhiều so với nhiễu thật 0.5 px. Camera vì thế
  được tin ít hơn mức nó thực sự xứng đáng, nên kết quả an toàn (không làm xấu đi)
  nhưng chưa tận dụng hết thông tin camera.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. **Khác biệt đo lidar 3D và camera 2D trong EKF (`z`, `R`)?**
   - **LiDAR:** `z = [x, y, z]ᵀ` tính bằng mét, `R = diag(0.1², 0.1², 0.1²)`.
     `h(x)` là phép biến đổi cứng (xoay + tịnh tiến), tuyến tính theo vị trí, nên
     `H` là hằng số: ma trận xoay 3×3 cộng thêm 3 cột 0 cho vận tốc.
   - **Camera:** `z = [u, v]ᵀ` tính bằng pixel, `R = diag(5², 5²)`
     ([camera_fusion.py:84](workspace/camera_fusion.py) `build_camera_measurement`).
     `h(x)` là phép chiếu pinhole `u = cᵢ − fᵢ·y_s/x_s`, `v = cⱼ − fⱼ·z_s/x_s`
     ([camera_fusion.py:55](workspace/camera_fusion.py)). Hàm này **phi tuyến** vì chia
     cho độ sâu x_s, nên `H` là Jacobian phải tính lại tại từng `x`. Đây là lý do cần
     **Extended** KF. Camera chỉ có 2 chiều đo cho 3 chiều vị trí, nên không quan sát
     được độ sâu. Nó chủ yếu tinh chỉnh vị trí ngang và đứng, như phần phân tích ở trên.
   - Hai loại đo dùng **chung** `ekf_update` ([kalman.py:116](workspace/kalman.py)).
     Chỉ `z`, `R`, `h(x)`, `H` khác nhau, do `meas.sensor` quyết định.

2. **Vì sao cần gating Mahalanobis trước khi gán?**
   Khoảng cách Euclid bỏ qua độ bất định. Mahalanobis `d² = γᵀS⁻¹γ`, với
   `S = HPHᵀ + R` ([association.py:31](workspace/association.py)), chuẩn hoá innovation
   theo độ bất định của cả track lẫn sensor. Track mới (P lớn) được phép ghép xa hơn,
   track đã ổn định thì chặt hơn. Hai đơn vị khác nhau (mét và pixel) cũng đưa được về
   cùng một thang không thứ nguyên. Nếu đo khớp với mô hình, `d²` tuân theo phân phối
   χ² với số bậc tự do bằng `dim_meas`. Cổng `d² < χ²₀.₉₉₅(dim_meas)` (12.84 cho lidar,
   10.60 cho camera, [association.py:61](workspace/association.py)) loại các cặp gần như
   chắc chắn sai trước khi gán greedy. Không có cổng này, một FP hoặc một đo của xe khác
   có thể bị gán vào track và kéo track lệch, sinh ghost hoặc đổi ID. Cost matrix kiểm
   tra **FOV trước** khi tính Mahalanobis ([association.py:82](workspace/association.py)),
   để không chiếu một điểm sau lưng camera (độ sâu ≤ 0).

3. **Pipeline là track-then-fuse hay fuse-then-track? Chỉ ra trên log.**
   **Track-then-fuse.** Chỉ có một danh sách track. Mỗi frame, `KF.predict` được gọi một
   lần ([run_lab.py:200](../platform/fusion_lab/scripts/run_lab.py)), sau đó
   `associate_and_update(..., lidar_sensor)` (dòng 202), rồi mới đến
   `associate_and_update(..., camera_sensor)` (dòng 209). Camera **không** được gộp
   với point cloud trước bước detection; nó chỉ là một lượt EKF update thêm trên track
   đã có. Trên log: record `lidar` và `fused` của cùng một frame có `confirmed`,
   `matches`, `ghosts`, `misses` **giống hệt nhau ở cả 199 frame**, chỉ `sum_sq_err`
   khác. Ví dụ frame 100: cả hai có `confirmed = 3, matches = 3`, nhưng `sum_sq_err` là
   0.0601 (lidar) so với 0.0488 (fused). Track được tạo và quản lý hoàn toàn từ LiDAR;
   camera chỉ tinh chỉnh vị trí của track sẵn có.

4. **Nếu camera lệch calibration, triệu chứng gì trên innovation/residual?**
   Lệch extrinsic làm `h(x)` chiếu sai một cách **có hệ thống**. Innovation camera
   `γ = z − h(x)` không còn có trung bình 0 mà bị lệch một bias ổn định (lệch góc
   yaw thì chủ yếu lệch `u`, lệch pitch thì lệch `v`). Bias này tăng hoặc giảm theo độ
   sâu, vì một góc lệch cố định ứng với số pixel cố định nhưng số mét thì tăng theo
   khoảng cách. Hệ quả là NIS `d²` trung bình tăng vượt quá `dim_meas = 2`. Nếu lệch
   nhỏ, đo vẫn lọt cổng χ² và EKF bị **kéo** về phía sai, nên RMSE fused tăng và có thể
   vượt RMSE lidar. Nếu lệch lớn, phần lớn đo camera bị cổng loại (cost = inf), fused
   lùi về gần như lidar-only. Vì thế nhìn RMSE thôi có thể không thấy lỗi, cần theo dõi
   thêm phân phối innovation và tỉ lệ đo camera bị gate.

5. **Vì sao `associate_and_update(..., sensor)` cần sensor tường minh ở frame rỗng?
   Vì sao lidar quyết định score/init/delete còn camera chỉ EKF update?**
   Khi `meas_list` rỗng, không có `meas.sensor` nào để suy ra đây là lượt lidar hay
   camera. Nhưng một frame lidar rỗng vẫn **phải** trừ score các track trong FOV và
   xoá track hết score. Hàm luôn gọi `manager.manage_tracks(unassigned_tracks,
   unassigned_meas, sensor)` ([association.py:153](workspace/association.py)), và
   manager dựa vào `sensor.name` để biết có áp dụng vòng đời track hay không. Nếu bỏ
   qua frame rỗng, track của một FP sẽ sống mãi và thành ghost.
   LiDAR quyết định sự tồn tại vì nó cho vị trí 3D đầy đủ, phủ 360°, và chính detection
   LiDAR là nguồn tạo track. Camera chỉ có 2D, FOV hẹp (FRONT), và trong lab là đo mô
   phỏng từ nhãn. Nếu camera được cộng/trừ score, một xe ngoài FOV camera sẽ bị trừ
   oan, và cùng một xe sẽ bị tính hai lần mỗi frame. Vì vậy camera chỉ chạy
   `filter_obj.update`; `handle_updated_track` và `manage_tracks` của manager bỏ qua
   lượt camera.

6. **Điều kiện xác nhận, giữ confirmed sau miss, và điều kiện xoá track.**
   ([track_management.py](workspace/track_management.py))
   - **Khởi tạo:** từ một đo LiDAR chưa được gán, `score = 1/6`, `state = initialized`.
     P vị trí là R xoay sang hệ toạ độ xe; P vận tốc là σ = 50, 50, 5.
   - **Mỗi frame lidar:** hit thì `score += 1/6`, tối đa 1. Miss **trong FOV LiDAR** thì
     `score −= 1/6`. Track chưa confirmed mà có hit chuyển sang `tentative`.
   - **Xác nhận:** khi `score > 0.8`. Cần 4 hit sau lần khởi tạo (5/6 ≈ 0.833). Trên log,
     frame 0–3 có `confirmed = 0`, frame 4 có `confirmed = 2`.
   - **Giữ confirmed:** đã confirmed thì không bao giờ hạ trạng thái. Một miss từ
     score 1.0 còn 5/6 ≥ 0.6, nên track không bị xoá.
   - **Xoá:** khi `P[0,0]` hoặc `P[1,1]` lớn hơn `max_P = 9` (độ bất định vị trí > 3 m),
     **hoặc** track confirmed có `score < 0.6`, **hoặc** track chưa confirmed có
     `score ≤ 0`. Track confirmed với score 1.0 bị xoá sau 3 miss liên tiếp
     (1 → 0.83 → 0.67 → 0.5).

## Bonus (không bắt buộc)

- Không

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): Claude Code (model Claude Opus 5.5)
- Dùng cho phần nào (hàm, câu hỏi, debug): Claude viết code toàn bộ Part E–H (`kalman.py`, `camera_fusion.py`, `association.py`, `track_management.py`), cài môi trường, chạy `fusion-run-lab`, phân tích log và soạn nháp báo cáo này (phần tóm tắt và 6 câu trả lời). Các commit có dòng `Co-Authored-By: Claude`.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): `pytest student/tests -q` pass 128/128 (không còn failed/xfailed); chạy Waymo `--fusion compare --seed 0`, frame 0–198; đối chiếu số trong báo cáo với `metrics.json` và `grade_run.log`; `python tools/check_submission.py`; đối chiếu công thức EKF, χ² và pinhole với `docs/HUONG_DAN_KY_THUAT.md`.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [x] Đã điền đủ file này, gồm khai báo AI
- [x] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [x] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
