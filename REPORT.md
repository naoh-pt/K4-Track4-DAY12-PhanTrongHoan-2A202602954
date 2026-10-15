# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Phan Trọng Hoàn |
| MSSV (đúng bằng `--student-id` đã chạy) | 2A202602954 |
| GitHub | naoh-pt |
| Seed chia dữ liệu / seed mô hình | 3825 / 8321, 8322, 8323 |
| Số seed huấn luyện (`--seeds`) | 3 |

## Kết quả cuối

Lượt chính thức trên Windows 11, CPU Intel Core i5 thế hệ 10, RAM 20 GB, Python 3.10.11 trong `.venv`:

```powershell
.\.venv\Scripts\python.exe -u run_all.py --student-id 2A202602954 --seeds 3
```

Không dùng `--quick`. Hoàn tất trong 1.360 giây (22 phút 40 giây); riêng fix ghi 1.116 giây trong [fix.json](evidence/fix.json).
Số liệu từ [openloop.json](evidence/openloop.json), [closedloop.json](evidence/closedloop.json), [gate.json](evidence/gate.json). Dải là min–max qua ba seed, không phải khoảng tin cậy.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,261 | 20,742 | BLOCK: R2, R3, R4, R6 |
| IDM | 1,537 | 0,104 | baseline |
| EgoMLP | 0,754–0,762 | 12,626–21,540 | BLOCK: R2, R3, R4, R5, R6 |
| PercMLP | 0,624–0,626 | 0,451–0,971 | BLOCK: R2, R4, R6 |
| PercMLP+ (sau khi sửa) | 0,836 ± 0,015 (trung bình ± độ lệch chuẩn) | 0,243–0,382 | BLOCK: R4, R6 |

Mã gốc chỉ lưu L2 trung bình/độ lệch chuẩn của PercMLP+, không lưu L2 từng seed trong `openloop.json`; vì vậy không suy ra hay bịa dải min–max này. Dải va chạm của cả ba mô hình học có đủ trong `closedloop.json`.

## Câu 1 · Dữ liệu

Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ba vấn đề chính là cột gia tốc xe dẫn không đáng tin, quỹ đạo trùng và nguy cơ rò rỉ nếu chia theo khung hình.
[data_audit.json](evidence/data_audit.json) ghi 992.679 dòng, 9.993 quỹ đạo ban đầu; loại 383 bản trùng, còn 9.610 quỹ đạo.
Tương quan `Acc_LV` với Δv/Δt chỉ 0,04535, so với 0,92909 của xe theo sau; do đó không dùng `Acc_LV` để khai thác phanh.
Mã gốc làm mượt `Speed_LV` bằng trung bình trượt 5 mẫu rồi lấy đạo hàm số với dt = 0,1 giây để tạo `a_lead`.
Quỹ đạo trùng theo hồ sơ tốc độ hai xe và gap phải được loại trước khi chia, tránh một mẫu xuất hiện ở cả train/test.
Chia theo quỹ đạo giữ các khung hình lân cận của cùng diễn biến ở cùng phía, tránh đánh giá quá lạc quan; `closedloop.json` xác nhận 2.883 quỹ đạo test với split seed 3825.

## Câu 2 · Phát lại

Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

[replay.json](evidence/replay.json) ghi chạy lại có 0/600 đoạn lệch; refactor float32 làm 600 đoạn khác chính xác nhưng 0 đoạn vượt dung sai, sai lệch lớn nhất 0,0001222 m/s².
Dung sai từng lệnh là `1e-3 + 1e-3 * |golden|`; NaN cùng vị trí đệm được coi bằng nhau. Thứ tự phép toán và độ chính xác float có thể tạo sai khác nhỏ mà hành vi vẫn tương đương theo tiêu chí lab.
Thêm AEB làm 11/600 đoạn vượt dung sai (12 đoạn khác chính xác), cho thấy cần phân biệt sai khác số học với thay đổi hành vi.
Mở đoạn 66 trong `work/replay_log.jsonl`: khung 75–77 có TTC khoảng 1,475; 1,462; 1,436 giây, dưới ngưỡng 1,5 giây nên lệnh khoảng −4,38…−4,35 đổi thành −6 m/s².
AEB chỉ đổi các lệnh thỏa điều kiện TTC và chưa phanh đủ mạnh, không đổi toàn bộ đoạn; danh sách đoạn lệch ở `behaviour_change_aeb`.
Phát nhanh nhất có 0 khung trễ/rớt; thời gian thực có 582 khung trễ và 576 khung rớt trên 59.696 khung, làm 372/600 đoạn vượt dung sai. Đây là chi phí xử lý mô phỏng, không phải phép đo độ trễ phần cứng.

## Câu 3 · Kịch bản

Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

[scenarios.json](evidence/scenarios.json) có lưới 60 × 45 = 2.700 ô, cruise 14 m/s, gap 2,760–59,312 m và tốc độ tiến gần 0–14 m/s.
Biên bất khả thi là `gap < closing*t_react + closing²/(2*9)` khi đang tiến gần: gồm khoảng cách mất lúc trễ và quãng đường triệt tiêu tốc độ tương đối bằng phanh tối đa.
Trễ 0 → 0,6 giây làm vùng bất khả thi và va chạm IDM tăng 3,630% → 10,222%, tức 6,593 điểm phần trăm; IDM không có va chạm còn tránh được trên lưới này.
FullBrake so với biên vật lý có 0 ô lệch xa biên ở cả hai độ trễ; vùng sát biên được bỏ khỏi kiểm tra vì sai khác do bước thời gian rời rạc.
Khai thác 614 ca phanh (6,389% quỹ đạo); [scenarios_idm.png](evidence/scenarios_idm.png) cho thấy log tập trung ở tốc độ tiến gần thấp, ít phủ vùng khó.
`human_brake_limit_p99.9` = 3,625 m/s², và 12,667% lưới đòi phanh mạnh hơn. Đây là giới hạn phân vị 99,9% về độ lớn phanh, không phải cực đại tuyệt đối dù dòng in dùng từ “mạnh nhất”.

## Câu 4 · Vòng hở và vòng kín

Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Theo `summary` trong [openloop.json](evidence/openloop.json) và `ranking` trong [closedloop.json](evidence/closedloop.json), xếp hạng dùng trung bình qua seed:

| Bộ điều khiển | L2 trung bình (m) | Hạng vòng hở | Va chạm trung bình (%) | Hạng vòng kín |
|---|---|---|---|---|
| constant-velocity | 1,261 | 3 | 20,742 | 4 |
| IDM | 1,537 | 4 | 0,104 | 1 |
| EgoMLP | 0,757 | 2 | 16,938 | 3 |
| PercMLP | 0,625 | 1 | 0,717 | 2 |

Ba cặp đảo thứ tự là IDM với constant-velocity, EgoMLP và PercMLP; xem [openloop_vs_closedloop.png](evidence/openloop_vs_closedloop.png).
L2 đo độ giống quỹ đạo người lái; IDM phản ứng theo gap và tốc độ tương đối nên có thể khác người lái mà vẫn tránh va chạm tốt hơn trong mô phỏng.
EgoMLP dự đoán tốt hơn constant-velocity trên log nhưng thiếu khoảng cách và tốc độ xe dẫn để phản ứng khi tình huống đổi.
Khi tự lái, trạng thái có thể lệch khỏi dữ liệu huấn luyện và sai số tích lũy; L2 trên cửa sổ độc lập không đo trực tiếp khả năng phục hồi đó.
Lát cắt xe dẫn sắp phanh chỉ chiếm 4,293% cửa sổ test; L2 lát cắt constant-velocity/IDM/EgoMLP/PercMLP là 1,934/1,589/1,054/0,863 m, đều cao hơn trung bình toàn bộ tương ứng.

## Câu 5 · Một lần chạy chưa đủ

EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

[closedloop.json](evidence/closedloop.json) ghi EgoMLP seed 8321/8322/8323 va chạm 480/364/621 trên cùng 2.883 quỹ đạo, tương ứng 16,649%/12,626%/21,540%.
Dải rộng 8,914 điểm phần trăm dù L2 chỉ dao động 0,754–0,762 m; điểm vòng hở ổn định không bảo đảm tự lái ổn định.
Wilson 95% của seed tốt nhất là [11,463%; 13,888%], seed xấu nhất [20,078%; 23,078%], không chồng nhau.
Wilson mô tả bất định của tỷ lệ từ số va chạm và số mẫu, không bao gồm toàn bộ bất định do seed, chuyển miền hay mô phỏng; các quỹ đạo cũng chưa chắc độc lập hoàn toàn.
Chỉ chọn seed 8322 có thể báo 12,626%, còn chọn 8323 sẽ báo 21,540%; chọn seed thuận lợi dễ tạo kết luận thiếu đại diện.
Vì vậy tôi báo dải, khoảng tin cậy và seed; gate dùng seed xấu nhất và R5 chặn EgoMLP vì dải vượt 1 điểm phần trăm.

## Câu 6 · Tác nhân phản ứng

Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng từ [agents.json](evidence/agents.json), tại trễ xe thử 0,6 giây; mã gốc chọn seed mô hình học có va chạm bất kỳ lớn nhất trong thế giới phản ứng:

| Bộ điều khiển | Va chạm bất kỳ: xe sau phát lại (%) | Xe sau phản ứng (%) |
|---|---|---|
| constant-velocity | 80,630 | 80,630 |
| IDM | 80,852 | 10,222 |
| EgoMLP | 82,778 | 82,778 |
| PercMLP | 81,444 | 27,704 |
| PercMLP+ | 80,593 | 12,333 |

T = 1,4 giây có khoảng cách Wasserstein nhỏ nhất trong lưới thử: 0,346 so với 0,897 của xe không phản ứng; headway trung vị người lái là 1,852 giây.
Xe sau phản ứng dùng IDM với trễ 0,6 giây, gap ban đầu 21,6 m. Khớp headway tốt hơn chỉ xác nhận theo phép đo này, không bảo đảm mọi hành vi đều chân thực.
Xe sau phát lại giữ tốc độ bất kể xe thử phanh khiến IDM bị tính 70,630% va chạm phía sau; với xe sau phản ứng, tỷ lệ phía sau IDM là 0%.
Va chạm xảy ra trong thế giới mô phỏng đã chọn nhưng dễ dẫn tới quy lỗi sai cho IDM nếu coi tác nhân phát lại là người lái chân thực.
IDM trông hơi kém constant-velocity khi xe sau phát lại nhưng tốt hơn nhiều khi xe sau phản ứng; giả định về tác nhân ảnh hưởng trực tiếp kết luận.

## Câu 7 · Cổng phát hành và sửa

Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

[gate.json](evidence/gate.json) chặn PercMLP ở R2/R4/R6: cận trên Wilson log 1,400% > 1,104%; va chạm còn tránh được 17,481% > 2%; va chạm thế giới phản ứng 27,704% > 12,222%.
R4/R6 phù hợp với thiếu độ phủ cut-in khó; R2 cùng L2 tốt nhưng vẫn va chạm khi tự lái phù hợp với sai số tích lũy. Evidence tổng hợp chưa xác định nguyên nhân riêng của từng ca.
[fix.json](evidence/fix.json) ghi 20.000 cut-in và 2 vòng DAgger mỗi seed: cut-in mở rộng tình huống; DAgger thu trạng thái mô hình tự đi vào rồi gắn nhãn dự báo 3 giây của IDM tại chính trạng thái đó.
Sau sửa, L2 tăng 0,625 → 0,836 m (lát cắt 0,863 → 1,009 m), nhưng va chạm log giảm 0,451–0,971% → 0,243–0,382%; R2 chuyển Fail → Pass với cận trên 0,682%.
PercMLP+ vẫn BLOCK: R4 = 2,111% (57/2.700 ô) > 2% (54 ô), R6 = 12,333% > 12,222%; R1/R3/R5/R7 tiếp tục Pass. Tôi giữ nguyên ngưỡng và báo đúng kết quả.
Không có bằng chứng tốt hơn IDM: IDM va chạm log 0,104% và có 0% diện tích va chạm còn tránh được. Ở cruise ngoài miền 20 m/s, PercMLP+ còn 0,667–1,667% diện tích này trong khi IDM vẫn 0% (`held_out_cruise_speed`).
Tôi chưa đề xuất phát hành; cần phân tích ô thất bại seed 8323, mở rộng kiểm thử độc lập và độ bền trước tác nhân/cảm biến khác. Quy trình gộp cut-in và DAgger nên chưa tách được đóng góp nhân quả của từng cách sửa.

## Câu 8 · Bài lab này không chứng minh được gì

Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Thứ nhất, mô phỏng chỉ chuyển động dọc 1 chiều, dt = 0,1 giây; cut-in là xe đã chen xong nên chưa kiểm tra điều khiển ngang hoặc toàn bộ quá trình đổi làn.
Thứ hai, xe dẫn phát lại không phản ứng với xe thử; câu 6 cho thấy thay tác nhân có thể đổi mạnh kết luận. Khớp headway chưa đủ xác nhận mọi phản ứng giao thông.
Thứ ba, kiểm thử nhiễu trong `closedloop.json` chỉ dùng độ lệch chuẩn gap 1 m và tốc độ xe dẫn 0,5 m/s, chưa bao quát che khuất, mất phát hiện hay lỗi cảm biến có tương quan.
Thứ tư, 2.883 quỹ đạo test ngắn cùng nguồn và 3 seed chưa đại diện mọi đường, thời tiết, kiểu lái; Wilson không bao gồm sai lệch phân bố và giả định mô phỏng.
Ngoài miền cruise 20 m/s, `fix.json` vẫn ghi PercMLP+ va chạm còn tránh được 0,667–1,667%; cải thiện trên lưới thử không bảo đảm tổng quát sang thực tế.
Gate chỉ áp dụng trong giả định lab. Trước quyết định thực tế cần dữ liệu độc lập đa dạng, tác nhân tương tác được kiểm chứng và thử nghiệm an toàn có kiểm soát; bài này chưa chứng minh hệ thống an toàn để triển khai.
