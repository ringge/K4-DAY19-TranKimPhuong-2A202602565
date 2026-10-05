# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Kim Phương  **MSSV:** 2A202602565  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     91.9
graph       196     93658     4492   0.00945    162.7

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     1.96
graph       0.83   1.67     5844       84   0.00092     3.08
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00112 | 0.00945 | ×8.44 |
| Indexing giây | 91.9 | 162.7 | ×1.77 |
| Mỗi câu: USD | 0.00013 | 0.00092 | ×7.08 |
| Mỗi câu: giây | 1.96 | 3.08 | ×1.57 |
| Mỗi câu: in_tok | 694 | 5844 | ×8.42 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> Khi indexing, GraphRAG gọi thêm LLM để trích xuất thực thể và quan hệ từ các bài báo nhằm xây dựng KG: số calls tăng từ 176 lên 196, in_tok từ 56072 lên 93658 và phát sinh 4492 out_tok, làm chi phí tăng khoảng 8.44 lần. Khi querying, GraphRAG bổ sung dữ kiện truy xuất từ Neo4j vào các đoạn văn bản lấy bằng vector search, khiến ngữ cảnh đầu vào tăng từ 694 lên 5844 token/câu và đầu ra tăng từ 48 lên 84 token/câu, nên chi phí tăng khoảng 7.08 lần. Các bước trích xuất, ghi/truy xuất graph và xử lý nhiều token hơn cũng làm thời gian indexing và querying tăng lần lượt khoảng 1.77 và 1.57 lần.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai nêu đúng định nghĩa tiền chất; Graph bổ sung dẫn chiếu Điều 2, khoản 4 Luật PCMT nhưng không tăng recall hay judge. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai xác định đúng Trần Thanh Tuấn và Trần Minh Tâm lãnh án tử hình; Graph bổ sung Điều 251 BLHS nhưng điểm bằng Flat. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nêu được mức án 36 tháng tù của Lê Minh Thành và liên hệ tội danh với Điều 251 BLHS, khung cơ bản 02–07 năm, còn Flat trả lời không đủ thông tin. |
| Q4 | cross-kb | 0.00 / 0 | 0.67 / 1 | Graph | Graph liên hệ hành vi của Hoàng Nato với Điều 255 BLHS nhưng chỉ đạt recall 0.67 và judge 1, còn Flat không đủ thông tin. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Graph nêu rõ Điều 250, khoản 4 cho vụ Cái Quang Huy vận chuyển hơn 9,6 kg MDMA, còn Flat chỉ nói “khoản b)” mà không xác định rõ điều luật. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph (recall); hòa (judge) | Graph đạt recall 0.33 so với 0.00 của Flat khi liệt kê các vụ liên quan MDMA, nhưng cả hai cùng judge 1 nên chưa trả lời đầy đủ, chính xác. |

## 3. Phân tích lỗi (20 điểm)

Chọn hai nhóm **E2 (thiếu ngữ cảnh luật)** và **E4 (phép đo sai)**. Bằng chứng gồm câu trả lời trong `ket_qua_benchmark_kg.txt`, đáp án chuẩn trong `data/benchmark_kg.json` và truy vấn chỉ đọc đã chạy khi Neo4j có 200 node / 378 cạnh (khớp số lượng trong benchmark); không dựng lại graph hay sửa file kết quả trong lần thu thập đó. Khi kiểm tra lại sau đó, graph chỉ còn 146 node / 289 cạnh, không có dữ liệu mang doc_id `news-100260924095400982` và không tìm thấy Person tên Dương Minh Tuấn hoặc có biệt danh chứa “Nato”; vì vậy truy vấn vụ Hoàng Nato dưới đây không trả dòng trên graph hiện tại. Các kết quả ghi bên dưới là bằng chứng của lần thu thập trước, không phải kết quả của graph đã thay đổi.

### Lỗi E2: Thiếu ngữ cảnh luật — lấy khung cơ bản làm mức phạt tối đa

- **Hiện tượng:** Q4 hỏi hành vi của Hoàng Nato và mức phạt tối đa theo BLHS, nhưng GraphRAG trả lời “tối đa 7 năm” (recall 0.67 / judge 1). Đây là mức cao nhất của khoản 1 Điều 255, không phải khung cao nhất của cả điều luật: khoản 4 quy định tù 20 năm hoặc tù chung thân.
- **Bằng chứng từ benchmark:** `ket_qua_benchmark_kg.txt`, Q4 Graph:

> Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Hành vi này có thể bị phạt tù tối đa 7 năm theo Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy.

**Truy vấn đã chạy: kiểm tra cầu nối của vụ và chất liên quan.**

```cypher
MATCH (p:Person {name: 'Dương Minh Tuấn'})-[:INVOLVED_IN]->(k:Case)
      -[:CHARGED_WITH]->(:Crime)<-[:DEFINES]-(a:Article {id: 'Điều 255 BLHS'})
WHERE k.doc_id = 'news-100260924095400982'
OPTIONAL MATCH (k)-[:INVOLVES]->(s:Substance)
RETURN p.name AS nguoi, k.name AS vu_an, a.id AS dieu, collect(s.name) AS chat;
```

Kết quả thực tế:

```text
nguoi: Dương Minh Tuấn
vu_an: Vụ sử dụng ma túy etomidate của Hoàng Nato và Phan Kim Nhi
dieu: Điều 255 BLHS
chat: ["etomidate"]
```

**Truy vấn đã chạy: graph có đủ các khoản về hình phạt tù không?**

```cypher
MATCH (a:Article {id: 'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
WHERE cl.number IN [1, 2, 3, 4]
OPTIONAL MATCH (cl)-[:MENTIONS]->(s:Substance)
RETURN cl.number AS khoan, cl.penalty AS khung_phat, collect(s.name) AS chat
ORDER BY khoan;
```

Kết quả thực tế:

| khoan | khung_phat | chat |
| --- | --- | --- |
| 1 | phạt tù từ 02 năm đến 07 năm | [] |
| 2 | phạt tù từ 07 năm đến 15 năm | [] |
| 3 | phạt tù từ 15 năm đến 20 năm | [] |
| 4 | phạt tù 20 năm hoặc tù chung thân | [] |

Đã gọi trực tiếp `graph.context(question_Q4, ["news-100260920221957595"])` trên graph hiện tại: dữ kiện luật của Điều 255 chỉ có khoản 1 dưới đây; kiểm tra toàn bộ danh sách facts không có chuỗi “chung thân”. Đây là lần tái hiện truy xuất hiện tại với doc_id chỉ định, không phải prompt gốc đã lưu của lần benchmark.

```text
[Điều 255 BLHS - Tội tổ chức sử dụng trái phép chất ma túy] khoản 1: 1. Người nào tổ chức sử dụng trái phép chất ma túy dưới bất kỳ hình thức nào, thì bị phạt tù từ 02 năm đến 07 năm.
```

- **Nguyên nhân:** Lỗi ở bước Cypher truy xuất trong `src/graph.py`, `Neo4jGraph.context()`: nhánh đi từ Case sang Article chỉ lấy `cl.number = 1` hoặc khoản có chất trùng qua `Case-[:INVOLVES]->Substance<-[:MENTIONS]-Clause`. Điều 255 phân biệt các khung theo số lần phạm tội, số người, độ tuổi và hậu quả, không liệt kê tên chất cụ thể; vì vậy cả bốn khoản đều không có cạnh `MENTIONS`, khiến khoản 2–4 bị loại dù đã tồn tại trong graph. Cầu nối vụ → tội → luật vẫn hoạt động, và regex đã tách đủ khoản; không thể sửa lỗi này chỉ bằng thêm etomidate vào danh sách chất.
- **Đề xuất sửa:** Trong `src/graph.py`, sửa chính sách chọn khoản của `Neo4jGraph.context()` theo mục đích câu hỏi: với câu hỏi “tối đa”/“khung cao nhất”, lấy đầy đủ các khoản của Điều luật liên quan thay vì bắt buộc khớp chất, đồng thời dành chỗ cho các khoản này trước khi cắt `max_facts`. Bổ sung vào `GRAPH_PROMPT` yêu cầu dẫn khoản và phân biệt **khung cao nhất của điều luật** với **mức án áp dụng cho người cụ thể**; thiếu tình tiết thì không kết luận người đó thuộc khoản 4. Đánh đổi: prompt dài hơn, tăng token và độ trễ, nhưng không phải gọi thêm LLM để trích xuất hay dựng lại graph; cần kiểm tra Q4 nêu được khoản 4 và tù chung thân mà không khẳng định đó là án của Hoàng Nato.

### Lỗi E4: Phép đo sai — recall từ khóa không phản ánh đúng độ bao phủ vụ việc

- **Hiện tượng:** Q6 hỏi những vụ liên quan MDMA. Flat có recall 0.00 nhưng judge 1; Graph có recall 0.33 nhưng cũng judge 1. Recall đang đếm chuỗi tên riêng, nên mô tả đúng một vụ bằng tên sự kiện vẫn có thể không được tính điểm; ngược lại, đủ tên riêng chưa chứng minh câu trả lời đúng và không có vụ thừa.
- **Bằng chứng từ benchmark:** Trích Q6 trong `ket_qua_benchmark_kg.txt`:

```text
Flat — recall=0.00, judge=1:
1. Vụ việc của Đức liên quan đến số viên nén hình tam giác màu hồng - xám được xác định là MDMA.
2. Vụ việc của Thành liên quan đến 5 viên nén màu trắng được xác định là ma túy MDMA.
3. Vụ việc của Đông liên quan đến 0,686g ma túy MDMA được thu giữ trong buồng chữa bệnh.

Graph — recall=0.33, judge=1:
1. **Vụ vận chuyển ma túy từ Đức về Việt Nam**: Tổng khối lượng hơn 9,6kg MDMA.
2. **Vụ góp 14 triệu đồng mua ma túy tại Hà Nội**: Vụ này có liên quan đến 5 viên MDMA.
3. **Vụ tổ chức sử dụng ma túy tại Sầm Sơn**: Tại đây, công an thu giữ 0,686g ma túy MDMA.
4. **Vụ thao túng giám định tâm thần tại Viện Pháp y tâm thần Trung ương**: Vụ án này cũng liên quan đến ma túy MDMA.
```

`data/benchmark_kg.json` đặt `must_include` của Q6 là `["Cái Quang Huy", "Lê Minh Thành", "Pháp y tâm thần"]`. Đối chiếu bài gốc: `news-100260917203001265.md` xác nhận vụ vận chuyển từ Đức và hơn 9,6 kg MDMA là vụ Cái Quang Huy; `news-100260918080821054.md` xác nhận vụ góp 14 triệu đồng là vụ Lê Minh Thành và ba thanh niên. Như vậy, hai mô tả đầu của Graph nhận diện được vụ chuẩn nhưng không chứa tên riêng mà phép đo yêu cầu.

Đã chạy trực tiếp `bench_kg.keyword_recall()` trên hai câu trả lời đã lưu:

| Chuỗi bắt buộc | Flat có chuỗi? | Graph có chuỗi? |
| --- | --- | --- |
| Cái Quang Huy | Không | Không |
| Lê Minh Thành | Không | Không |
| Pháp y tâm thần | Không | Có |
| Recall tính lại | 0.00 | 0.33 |

Thử trên bản sao trong bộ nhớ, chỉ thêm “Cái Quang Huy” và “Lê Minh Thành và ba thanh niên” vào tên hai vụ đầu của câu trả lời Graph; không sửa các mục 3–4, không gọi lại LLM và không ghi đè benchmark:

```text
Q6 Graph original recall: 0.33
Q6 Graph renamed case labels recall: 1.00
Other items unchanged: True
```

Điểm 1.00 này vẫn không phát hiện việc mục 3 gán nơi thu giữ 0,686 g MDMA cho Sầm Sơn: `data/drug_news/news-100260930085028036.md` ghi số MDMA đó được thu tại **buồng chữa bệnh của Đông ở Viện Pháp y tâm thần Trung ương**. Vì vậy judge 1 là phù hợp với câu trả lời chỉ đúng một phần; không có cơ sở đổi thành judge 2 chỉ vì nhận diện được tên vụ.

- **Nguyên nhân:** Lỗi ở phép đo `keyword_recall()` trong `bench_kg.py`: `k.lower() in answer.lower()` chỉ kiểm tra chuỗi con, không đối chiếu danh tính vụ, diễn đạt tương đương, nơi xảy ra/thu giữ hay vụ bị liệt kê thừa. `must_include` của Q6 trong `data/benchmark_kg.json` dùng tên người để đại diện vụ án, trong khi câu hỏi hỏi **vụ việc** và LLM có thể dùng mô tả sự kiện. Judge đánh giá theo đáp án chuẩn nên không cùng ý nghĩa với recall; chênh lệch hai chỉ số không tự động chứng minh judge sai.
- **Đề xuất sửa:** Trong `data/benchmark_kg.json`, bổ sung cho Q6 các vụ chuẩn theo doc_id và nhóm tên/mô tả tương đương, giữ lại đáp án chuẩn để kiểm tra nội dung. Trong `bench_kg.py`, bổ sung phép đánh giá coverage theo từng vụ (mỗi vụ chỉ tính một lần), báo precision khi có vụ thừa và lưu cả `reason` của judge thay vì chỉ `score`; giữ recall từ khóa nhưng ghi rõ đó là chỉ số khớp chuỗi, không phải độ đúng ngữ nghĩa. Đánh đổi: danh sách mô tả tương đương cần bảo trì và có nguy cơ khớp nhầm nếu quá rộng; đối chiếu ngữ nghĩa bằng judge giúp xử lý cách diễn đạt mới nhưng tốn thêm token nếu mở rộng prompt, còn lưu `reason` từ lượt judge hiện có không cần gọi thêm LLM. Kiểm tra bằng hai cách diễn đạt của cùng vụ phải có coverage như nhau, đồng thời câu có tên đúng nhưng gán sai nơi thu giữ không được coi là hoàn toàn đúng.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> **Nên dùng KG khi câu hỏi cần nối thông tin giữa nhiều nguồn hoặc suy luận nhiều bước**, như liên kết người → vụ án → tội danh → Điều luật → khung hình phạt. Trong benchmark này, Q3 tăng từ recall/judge **0.00/0** của Flat lên **1.00/2** của Graph, còn Q5 tăng từ **0.60/1** lên **1.00/2**; trung bình sáu câu, recall tăng từ **0.43 lên 0.83** và judge từ **1.00 lên 1.67**. Lợi ích này đi kèm chi phí indexing **0.00945 USD so với 0.00112 USD (×8.44)** và thời gian **162.7 so với 91.9 giây**; mỗi câu tốn **0.00092 USD so với 0.00013 USD (×7.08)**, mất **3.08 so với 1.96 giây**. Vì vậy, KG phù hợp khi nhu cầu truy vấn xuyên nguồn đủ quan trọng để chấp nhận chi phí xây dựng graph, token và độ trễ cao hơn.
>
> **Flat RAG là đủ khi câu trả lời nằm trong một đoạn luật hoặc một bài báo**, và ưu tiên là chi phí thấp, phản hồi nhanh: ở Q1–Q2, cả hai pipeline đều đạt **recall 1.00 / judge 2**, nên Graph chưa mang lại lợi ích về điểm số. KG cũng không bảo đảm câu trả lời luôn đúng: Q4 vẫn chỉ đạt **0.67/1** do thiếu khoản luật về khung cao nhất, còn Q6 đạt **0.33/1**, không hơn Flat về judge dù recall cao hơn. Do đó, cần sửa chính sách lấy ngữ cảnh luật và đánh giá độ bao phủ vụ việc trước khi tin cậy các câu hỏi mức phạt tối đa hoặc tổng hợp; các kết quả trên chỉ chứng minh hiệu quả trong sáu câu của bộ benchmark này, không đủ để kết luận GraphRAG luôn tốt hơn Flat RAG.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................                                                                            [100%]
48 passed in 0.03s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openrouter:openai/gpt-4o-mini | embedding = openrouter:openai/text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00065. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Lê Văn Đông

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> …
