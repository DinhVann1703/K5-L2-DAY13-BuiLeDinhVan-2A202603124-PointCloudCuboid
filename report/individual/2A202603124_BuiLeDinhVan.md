# Báo cáo Day 13 — Bùi Lê Đình Văn, 2A202603124

## 1. Kết luận trước
- **Thông tin cá nhân & môi trường:** Mã số học viên: `2A202603124`; Task CVAT: `D13-2A202603124`; 10 frame được giao: `s01f04`, `s01f51`, `s02f01`, `s02f39`, `s03f14`, `s03f55`, `s04f29`, `s04f44`, `s05f02`, `s05f20`. Hệ điều hành: Windows 11; Trình duyệt: Google Chrome; Phiên bản CVAT Server: `2.74.1` (CVAT SDK `2.83.0`); MSSV bạn ngồi cạnh / đổi kiểm: thực hiện độc lập trên máy local.
- **Tổng quan kết quả:** Đã gán nhãn tổng cộng **85 cuboid** trên 10 frame. Hai frame ít cuboid nhất là `s02f01` (5 cuboid) và `s03f55` (5 cuboid) do bối cảnh tuyến đường thoáng, mật độ phương tiện thấp và các xe ở cự ly quá xa (> 50 m) không đủ mật độ điểm LiDAR để xác định hình khối. Lệnh `selfcheck` ban đầu gắn cờ 10 cuboid; sau khi kiểm tra qua ảnh chiếu ngược `preview` và chỉnh sửa trên CVAT, số cảnh báo giảm xuống còn **4/85 cuboid** (các cảnh báo còn lại đều đã được đối chiếu và xác nhận hợp lệ theo địa hình/vật thể lân cận). Phần tự tin nhất là xác định đúng phân lớp đối tượng (`label`), góc hướng di chuyển (`yaw`) và kích thước chiều dài/rộng ở cự ly gần (< 25 m); phần còn yếu nhất là căn cao độ đáy hộp chạm mặt đường trên các đoạn đường có độ dốc dọc và ước lượng kích thước amodal đối với xe bị che khuất một nửa ở cự ly xa.
- **Thao tác sử dụng CVAT 3D:**
  - *Đổi góc nhìn:* Giữ chuột trái và kéo để xoay không gian 3D; giữ chuột phải (hoặc `Shift` + chuột trái) để dời (pan) khung nhìn; lăn con lăn chuột để phóng to/thu nhỏ; dùng các nút điều hướng hoặc khung chiếu `Top` để quan sát trực diện từ trên xuống.
  - *Vẽ và chỉnh cuboid:* Chọn công cụ vẽ cuboid (chế độ `Shape`, phím tắt `N`), chọn đúng lớp nhãn (`Car`, `Truck`, `Bus`, `Pedestrian`) và click vào tâm cụm điểm; nháy đúp hoặc chọn đối tượng để mở 3 khung chiếu `Top`, `Side`, `Front` phía dưới nhằm kéo sát kích thước (`l`, `w`, `h`) và xoay góc hướng.
  - *Mũi tên hướng và đối chiếu camera:* Quan sát trục màu đỏ (`+X`) của cuboid trong khung `Top` và khung 3D sao cho mũi tên chỉ đúng về phía đầu xe đang di chuyển; kiểm tra đồng thời với các khung ảnh `cam01`–`cam07` bên phải và xuất ảnh chiếu ngược bằng lệnh `python scripts/lab.py preview` vào thư mục `work/xem/` để rà soát lỗi.

## 2. Đọc frame (Bước 2)
Hai frame đọc kỹ: `s01f04` (ảnh `t1-s01f04-3d.png`, `t1-s01f04-top.png`) và `s01f51` (ảnh `t1-s01f51-3d.png`, `t1-s01f51-top.png`).

| Frame | Phía trước xe ở cạnh nào của ảnh nhìn từ trên xuống; +x, +y chỉ về đâu | Mặt đường (phẳng/dốc; thấp hơn LiDAR ~ m) | Vật xa nhất đọc được (~ m) | Loại vật thấy | Vật phía sau / hai bên | Cụm chưa đọc được |
|---|---|---|---|---|---|---|
| `s01f04` | Phía trước xe ở cạnh trên; trục `+x` hướng thẳng về phía đầu xe ego, trục `+y` hướng vuông góc sang bên trái làn đường. | Mặt đường đại lộ tương đối phẳng, thấp hơn cảm biến LiDAR khoảng `1,7 m` ($z \approx -1,7\text{ m}$). | Khoảng `45–50 m` (các xe chạy cùng chiều phía trước). | `Car`, `Truck`, `Pedestrian` | Bên trái có xe tải thùng trắng (`#2 Truck`) và xe con xanh; phía sau có xe con bạc chạy cùng chiều; vỉa hè bên phải có hai người đi bộ (`#8`, `#9 Pedestrian`). | Một số cụm điểm nhỏ sát hàng rào cây xanh vỉa hè và các xe ở làn xa > 55 m chỉ còn 2–3 điểm rời rạc. |
| `s01f51` | Phía trước xe ở cạnh trên; trục `+x` chỉ dọc theo hướng xe tiến phía trước, trục `+y` chỉ sang trái. | Mặt đường tương đối phẳng, hơi nghiêng nhẹ ở làn rìa, thấp hơn LiDAR khoảng `1,7 m`. | Khoảng `45 m`. | `Car`, `Truck` | Có xe con chạy song song ở làn bên cạnh và xe bám phía sau trong phạm vi quét 360°. | Các cụm điểm phản xạ thấp trên bề mặt kính/sơn tối màu của xe ở rìa xa và dải cây phân cách. |

**So sánh thông tin giữa Camera và LiDAR (từ 2 frame `s01f04` và `s01f51`):**
- *3 điều **chỉ camera** cho biết:* (1) Màu sắc, kiểu dáng chi tiết và chất liệu phương tiện (ví dụ phân biệt xe tải thùng kín trắng với xe con xanh/bạc); (2) Phân biệt rõ ranh giới giữa người đi bộ đứng gần cột/cây xanh trên vỉa hè hoặc nhận diện các xe ở rất xa (> 60 m) khi điểm LiDAR đã mất hẳn; (3) Trạng thái đèn tín hiệu giao thông, vạch kẻ liền/đứt trên mặt đường và hướng mặt/tư thế di chuyển của người đi bộ.
- *3 điều **chỉ LiDAR** cho biết:* (1) Tọa độ không gian 3D chính xác $(x, y, z)$ và khoảng cách trực tiếp tính bằng mét từ xe ego tới từng vật thể; (2) Kích thước hình học thực tế (chiều dài, chiều rộng, chiều cao tính bằng mét) không bị biến dạng phối cảnh như ảnh 2D; (3) Hình dạng khối chữ L (mặt hông và mặt sau/trước) giúp xác định chính xác góc lệch phương vị (`yaw`) của xe ngay cả trong vùng bóng râm hoặc ngược sáng.

Tám frame lướt:

| Frame | Phía trước ở cạnh nào của ảnh nhìn từ trên xuống | Điều lạ (nếu có) |
|---|---|---|
| `s02f01` | Cạnh trên (trục `+x` hướng thẳng lên theo chiều tiến của xe ego) | Mật độ phương tiện thưa (5 cuboid), xuất hiện xe buýt kích thước lớn (`Bus`). |
| `s02f39` | Cạnh trên (trục `+x` hướng lên phía trước) | Các xe chạy sát làn nhau dễ gây bóng che một phần hông xe phía sau. |
| `s03f14` | Cạnh trên (trục `+x` hướng lên phía trước) | Mật độ giao thông đông (11 cuboid), nhiều xe nối đuôi nhau ở cự ly trung bình. |
| `s03f55` | Cạnh trên (trục `+x` hướng lên phía trước) | Đường thoáng (5 cuboid), nhiều phương tiện ở xa > 50 m không đủ điểm LiDAR. |
| `s04f29` | Cạnh trên (trục `+x` hướng lên phía trước) | Có phương tiện thân dài trên 6 m dễ nhầm giữa `Car` cỡ lớn và `Truck`. |
| `s04f44` | Cạnh trên (trục `+x` hướng lên phía trước) | Có xe thân dài ~7 m chạy cùng làn cần đối chiếu kỹ camera để phân loại đúng. |
| `s05f02` | Cạnh trên (trục `+x` hướng lên phía trước) | Mặt đường có độ dốc tự nhiên khiến một số xe có tọa độ $z$ đáy thấp hơn ngưỡng chuẩn `-1,7 m`. |
| `s05f20` | Cạnh trên (trục `+x` hướng lên phía trước) | Số lượng vật thể ít (6 cuboid), tia LiDAR quét rõ mặt đường vòng cung đồng tâm. |

## 3. Ba cuboid có hướng dẫn (Bước 3)
Ảnh: `t2-a-car.png`, `t2-b-bus-truck.png`, `t2-c-ped-cyclist.png`.

| Vật | Frame, idx | Class | Mũi tên yaw đúng hướng? | Đáy chạm đường? | Đã kiểm Top/Side/Front + ảnh? | Kích thước trong bảng 4.3? |
|---|---|---|---|---|---|---|
| (a) xe con gần | `s01f04`, `#1` | `Car` | Có (trục đỏ hướng về đầu xe phía trước) | Có ($z_{\text{đáy}} \approx -1,70\text{ m}$) | Đã kiểm đủ 3 góc chiếu + `cam01`/`cam03` | Có (trong khoảng $l \in [3,0; 6,0]\text{ m}$, $w \in [1,4; 2,3]\text{ m}$) |
| (b) xe buýt hoặc xe tải | `s01f04`, `#2` | `Truck` | Có (trục đỏ hướng về cabin đầu xe, `yaw = 88,3°`) | Có ($z_{\text{đáy}} = -1,94\text{ m}$ chạm mặt đường) | Đã kiểm đủ 3 góc chiếu + `cam05` | Có (`l = 5,895 m`, `w = 2,275 m`, `h = 2,835 m`) |
| (c) người đi bộ hoặc xe hai bánh | `s01f04`, `#8` | `Pedestrian` | Có (trục đỏ hướng theo chiều bước đi, `yaw = 89,9°`) | Có ($z_{\text{đáy}} = -1,57\text{ m}$ chạm vỉa hè) | Đã kiểm đủ 3 góc chiếu + camera phải | Có (`l = 0,652 m`, `w = 0,660 m`, `h = 1,950 m`) |

- **Vòng hiệu chuẩn & đối tượng bỏ qua:** Thực hiện bài lab cá nhân trên môi trường CVAT local nên không tham gia task chung hiệu chuẩn; toàn bộ nhãn tuân thủ chặt chẽ ngưỡng kích thước và quy tắc amodal của guideline. Các vật **bỏ qua vì không đủ cơ sở**: các xe con/xe máy xuất hiện ở rìa xa trên `cam01` và `cam03` trong các frame `s02f01`, `s03f55`, `s05f20` (khoảng cách > 50–60 m) nhưng trên point cloud chỉ có dưới 3 điểm rời rạc hoặc không có điểm nào (`LOW_VISIBILITY` / `UNCERTAIN_CLASS`), không đủ căn cứ hình học để dựng hộp 3D chính xác.

## 4. Bốn hiện tượng chất lượng (Bước 4)

| Hiện tượng | Frame / tên ảnh / nguồn | Nguyên nhân (giả thuyết) | Thấy gì trên point cloud | Hệ quả khi gán (theo guideline) |
|---|---|---|---|---|
| 1. Bóng che | `s01f04` / `t4-1-occlusion.png` / Task `D13-2A202603124` | Tia laser LiDAR truyền thẳng và bị thân xe tải lớn (`#2 Truck`) ở làn gần chắn hoàn toàn, không thể xuyên tới xe chạy phía sau/bên cạnh. | Chỉ thấy một phần đầu hoặc đuôi xe phía sau (cụm điểm bị cắt cụt), phía sau xe lớn tạo thành một vùng đen hình nón không có tia quét. | Áp dụng quy tắc **amodal box**: không bóp hộp theo phần điểm ít ỏi nhìn thấy mà kéo dài hộp đủ kích thước thật của xe dựa trên phần cạnh đã lộ diện và ảnh camera. |
| 2. Thưa ở xa | `s03f55` / `t4-2-sparse.png` / Task `D13-2A202603124` | Chùm tia LiDAR phân kỳ theo góc đứng và góc ngang nên khoảng cách giữa các vòng quét giãn rộng theo cự ly (> 35–40 m). | Xe ở xa chỉ còn 1–2 đường quét ngang mảnh trên thân hoặc vài chấm rời rạc, mất hẳn biên dạng nóc và bánh xe. | Chỉ gán nhãn khi còn đủ cụm điểm xác định được vị trí và hướng xe; với vật quá xa không nhận diện được hình khối thì bỏ qua để tránh sinh nhiễu tọa độ. |
| 3. Phản xạ thấp / ít điểm bất thường | `s01f51` / `t4-3-low-reflect.png` / Task `D13-2A202603124` | Bề mặt xe sơn đen bóng hấp thụ năng lượng laser hoặc kính chắn gió/cửa sổ làm tia laser khúc xạ xuyên qua thay vì phản xạ về cảm biến. | Phần thân trên và kính xe bị khuyết điểm (rỗng ruột), chỉ còn các điểm phản xạ từ biển số, đèn phản quang và lốp xe. | Dùng các điểm phản xạ mạnh ở gầm/biển số làm mốc định vị, sau đó kéo chiều cao hộp lên đủ tầm nóc xe tiêu chuẩn theo đối chiếu trên camera. |
| 4. Điểm lạ | `s01f04` / `t4-4-noise.png` / Task `D13-2A202603124` | Tia laser phản xạ đa đường (multipath) dưới gầm xe/mặt đường ướt hoặc quét vào tán cây, dải phân cách sát lề đường. | Xuất hiện các cụm điểm lơ lửng bất thường gần mép hộp người đi bộ (`s01f04` #8, 15 điểm sát mép) hoặc bám sát hông xe (`s01f04` #10, 130 điểm sát mép). | Không kéo rộng hộp để ôm các điểm nhiễu/cây cối bên ngoài; giữ biên hộp ôm đúng kích thước vật lý của đối tượng chính. |

## 5. Số từ cuboid của tôi (Bước 5)

| Cuboid (frame, idx, class) | z (m) | h (m) | z đáy = z − h/2 (m) | dist_m (m) | r = √(x²+y²+z²) (m) | yaw_deg (°) | Nhận xét (lơ lửng / chìm / chạm đường) |
|---|---|---|---|---|---|---|---|
| `s01f04`, `#2`, `Truck` | `-0,519` | `2,835` | `-1,94` | `12,40` | `12,39` | `88,3°` | Chạm đường (`z đáy` sát mặt đường tại làn trái) |
| `s01f04`, `#8`, `Pedestrian` | `-0,598` | `1,950` | `-1,57` | `29,00` | `28,96` | `89,9°` | Chạm đường (đứng trên vỉa hè cao hơn lòng đường ~0,15 m) |
| `s05f02`, `#7`, `Car` | `-0,961` | `1,595` | `-1,76` | `15,10` | `15,10` | `91,9°` | Chạm đường (mặt đường đoạn này hơi dốc nên `ground_gap_m = -1,04 m`) |

- **(a) Cuboid gần nhất và xa nhất:** Cuboid gần nhất nằm ở cự ly `dist_m = 12,40 m` (`r = 12,39 m`, tiêu biểu là `s01f04 #2 Truck`); cuboid xa nhất nằm ở cự ly `dist_m = 37,10 m` (`s05f20 #6 Car` với `102` điểm LiDAR).
- **(b) So sánh kích thước với bảng chuẩn:** Người đi bộ `s01f04 #8 Pedestrian` có kích thước `l = 0,652 m`, `w = 0,660 m`, `h = 1,950 m` (chiều cao nhỉnh nhẹ ~0,15 m do cộng thêm khoảng tiếp xúc vỉa hè/mũ đội đầu, chiều dài và rộng nằm gọn trong ngưỡng người đi bộ); xe tải `s01f04 #2 Truck` có `l = 5,895 m`, `w = 2,275 m`, `h = 2,835 m` và xe con `s05f02 #7 Car` có `l = 4,135 m`, `w = 1,900 m`, `h = 1,595 m` đều nằm hoàn toàn trong khoảng kích thước tiêu chuẩn của bảng 4.3.
- **(c) Góc hướng `yaw_deg`:** Các phương tiện chạy cùng chiều phía trước và bên cạnh xe ego có `yaw_deg` tập trung quanh **`88,3° – 91,9°`** (tương đương $\approx 1,54 – 1,60\text{ rad}$, tức gần `90°` theo quy ước trục tọa độ của hệ thống CVAT/LiDAR trong bộ dữ liệu này); nếu có phương tiện chạy ngược chiều ở làn đối diện, góc hướng sẽ lệch xấp xỉ `180°` (quanh mốc `270°`).

Mật độ theo khoảng cách (4 cuboid ở các `dist_m` khác nhau):

| Cuboid (frame, idx, class) | dist_m (m) | Số điểm (`points_in`) |
|---|---|---|
| `s01f04`, `#2`, `Truck` | `12,4` | `3117` |
| `s05f02`, `#7`, `Car` | `15,1` | `355` |
| `s05f20`, `#5`, `Car` | `20,5` | `152` |
| `s05f20`, `#6`, `Car` | `37,1` | `102` |

- **Quy luật mật độ & đối chiếu bài giảng:** Khi khoảng cách tăng gấp **2,5 lần** (từ `15,1 m` lên `37,1 m` đối với cùng lớp `Car`), số điểm LiDAR trên bề mặt xe giảm từ `355` xuống còn `102` điểm (giảm khoảng **3,5 lần**, tỉ lệ nghịch theo khoảng cách), cùng xu hướng suy giảm với đường cong thực nghiệm trên slide bài giảng (xe con giảm từ ~1968 điểm ở 0–10 m xuống 718 điểm ở 10–20 m, 214 điểm ở 20–30 m và 72 điểm ở 30–40 m; riêng `s01f04 #2` đạt `3117` điểm vì là xe tải thùng lớn ở cự ly rất gần `12,4 m`).
- **Kết quả tự kiểm (`preview` và `selfcheck`):**
  - *Hai cuboid đã sửa sau kiểm tra:* (1) `s01f51 #8 Car`: ban đầu đáy hộp cao hơn mặt đường `0,60 m` (bị lơ lửng), đã kéo cạnh dưới trong khung `Side` hạ chạm sát mặt đường và hết cảnh báo; (2) `s04f29 #3` và `s04f44 #7`: ban đầu gắn nhầm nhãn `Car` với chiều dài `6,57 m` và `7,02 m`, đã đối chiếu ảnh `work/xem/` và chỉnh lại kích thước/phân lớp đúng chuẩn, giảm tổng cảnh báo từ 10 xuống còn 4.
  - *Các cảnh báo giữ nguyên:* `s01f04 #8 Pedestrian` (15 điểm sát mép) và `s01f04 #10 Car` (130 điểm sát mép) được giữ nguyên vì qua kiểm tra ảnh camera và 3D, các điểm nằm ngoài mép thuộc về cây xanh vỉa hè và dải phân cách nằm sát bên cạnh, không phải phần thân bị cắt hụt; `s05f02 #7` (`ground_gap_m = -1,04 m`) và `s05f02 #10` (`ground_gap_m = -0,28 m`) giữ nguyên vì mặt đường tại đoạn này có độ dốc tự nhiên, đáy hộp thực tế (`z_đáy = -1,76 m` và `-1,81 m`) vẫn ôm sát mặt đường dưới lốp xe.
- **Đổi kiểm:** Thực hiện bài lab độc lập trên máy cá nhân, tự đối chiếu chéo toàn bộ 10 ảnh chiếu ngược `work/xem/D13-2A202603124-*.jpg` với hai file `submission/D13-2A202603124-bangA.csv` và `submission/D13-2A202603124-tukiem.csv` để đảm bảo 100% số lượng cuboid (`85` cuboid) và tọa độ khớp hoàn toàn giữa CVAT và bảng xuất.

## 6. Ý nghĩa của nhãn, cách dùng AI và điều còn chưa chắc
- **(a) Góc nhìn từ mô hình học máy 3D:** Mô hình phát hiện vật thể 3D cho xe tự hành cần dự đoán vị trí, kích thước hình học thực và góc hướng trong không gian vật lý (đo bằng mét) để hệ thống lập kế hoạch quỹ đạo tính toán chính xác khoảng cách phanh và nguy cơ va chạm, điều mà tọa độ pixel 2D không thể cung cấp trực tiếp do phụ thuộc tiêu cự và góc nhìn camera. Với các vật thể ở xa có mật độ điểm thưa, cần quy tắc gán nhãn thống nhất (ngưỡng số điểm tối thiểu và suy luận amodal) để tránh đưa vào tập huấn luyện những nhãn nhiễu có độ bất định tâm/hướng quá lớn làm giảm độ hội tụ của mô hình.
- **(b) Sử dụng công cụ AI:**
  - *Công cụ:* Gemini.
  - *Câu hỏi chính:* Hướng dẫn khắc phục lỗi môi trường `.venv`/Docker CVAT local, giải thích quy tắc xác định hướng trục đỏ (`+X`), kích thước amodal khi vật bị khuất, và phân tích các cảnh báo hình học từ script `selfcheck`.
  - *Cách sử dụng kết quả:* Dùng để đối chiếu từng bước thao tác kỹ thuật trên CVAT và rà soát các cảnh báo độ cao đáy hộp, kích thước lớp nhãn trước khi tự tay chỉnh sửa trực tiếp trên dữ liệu point cloud và ảnh camera thực tế.
- **(c) Điều còn chưa chắc:** Với các xe con chạy ở làn ngoài cùng bên kia dải phân cách hoặc ở cự ly trên 35–40 m chỉ còn một mặt hông mảnh, việc ước lượng chiều rộng thực tế của xe và cao độ mặt đường tại vị trí đó vẫn còn độ sai số vài chục centimet do không có tia LiDAR chạm tới lốp xe phía khuất.