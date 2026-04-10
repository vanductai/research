KFM - IT - Tài Văn - SC002120 🐳, [10/4/26 06:00]
Mình mới tạo ra thêm một project/hệ thống multi-agents chạy trên Claude Cowork mà mình rất ưng, mang tên “Debate Arena”, một hệ thống giúp bạn nghiên cứu các vấn đề phức tạp bằng cách tự động dựng lên một cuộc tranh luận giữa các "chuyên gia AI" với các quan điểm trái ngược.

A. Hệ thống này là gì?

Hãy tưởng tượng bạn đang phân vân một vấn đề khó, ví dụ: từ to tát chính trị quốc tế như “Mỹ tấn công Iran là đúng hay sai?” hay đến nhỏ nhặt đời sống hàng ngày như “nên mua xe xăng hay xe điện?”

Nếu bạn hỏi các AI phổ biến hiện nay, như ChatGPT hoặc Claude, bạn sẽ nhận được một câu trả lời kiểu "hai mặt đều có lý, tùy bối cảnh...". Nghe có vẻ cân bằng, nhưng thật ra, AI không thực sự cố thắng lập luận, có khả năng các con số thống kê có thể sai hoặc bịa, và đặc biệt kết luận nước đôi, không giúp bạn quyết định gì cả

Debate Arena được xây dựng để giải quyết vấn đề này bằng cách tổ chức một cuộc tranh luận thật sự. Thay vì một AI trả lời, hệ thống sẽ tự tổ chức một cuộc tranh luận có quy củ: hai bên đối lập, một trọng tài, một người kiểm chứng sự thật, và ba giám khảo chấm từ ba góc độ khác nhau. Cuối cùng bạn nhận được một báo cáo có kết luận  và quan trọng hơn là một phần Tổng hợp (Synthesis) giúp bạn phân biệt đâu là bất đồng về dữ liệu (có thể tra cứu để giải quyết) và đâu là bất đồng về giá trị (cần phán xét của chính bạn).

B. Tại sao cần một hệ thống như vậy?

Khi một AI đơn lẻ trả lời câu hỏi tranh luận, có 5 vấn đề lớn:

1. Cùng một cái đầu. Một AI là một AI; nó có các thói quen suy nghĩ giống nhau dù đang đóng vai ai. Nếu bạn bảo nó "viết lập luận ủng hộ" rồi "viết lập luận phản đối", cả hai đều mang cùng blind spots , chỗ mà model không nhìn thấy vì thiếu trong dữ liệu huấn luyện.

2. Không có phản biện thật. Không ai thử thách lập luận yếu. Lập luận kém đi qua mà không bị lật tẩy.  Nói một cách khác, AI có thể không tự thách thức chính mình, không cãi lại chính mình để kiểm tra xem lập luận của mình có vững không

3. Số liệu có thể bịa. AI có thể hallucinate, tức là đưa ra con số, ngày tháng, tên nghiên cứu nghe rất chính xác nhưng thực ra bịa ra. Không ai check.

4. Nghiêng về câu trả lời an toàn và chỉ nhìn từ một góc độ quen thuộc. Model AI được huấn luyện để không gây tranh cãi, nên thường nghiêng về kết luận trung lập phổ biến, dù câu trả lời đúng có thể ngược lại.  Đây giống như trả lời nửa vời, nghe hợp lý nhưng không giúp bạn quyết định được gì

5. Không có tổng hợp thật.  Câu trả lời kiểu "có mặt A, có mặt B, bạn tự quyết" không giúp gì. Bạn cần biết: *Điểm gì thật sự còn bất đồng? Điểm gì 2 bên đồng ý mà chỉ diễn đạt khác? Bạn nên làm gì?

Debate Arena được thiết kế để trị từng khuyết tật đó.  Cụ thể, Debate Arena cố gắng lôi AI ra khỏi các khuyết tất của nó, để tự ép nó tranh luận với chính mình dưới nhiều vỏ bọc khác nhau, phải kiểm chứng từng sự kiện qua nguồn độc lập, và phải bị chấm điểm và đánh giá bởi các giám khảo AI có lăng kính không giống nhau.

C. Hệ thống hoạt động ra sao?

Bạn mở Claude Cowork và gõ thẳng câu hỏi vào khung chat.  Ví dụ: Liệu Việt Nam có nên áp dụng giờ làm 4 ngày/tuần trên toàn quốc trong 5 năm tới?"  Rồi gửi. Sau đó bạn không cần làm gì nữa 0 hệ thống tự chạy qua 11 bước liền mạch trong một khoảng thời gian (topic càng phức tạp hoặc nhiều tranh cãi thì khoảng thời gian càng dài) . Đây là những gì diễn ra bên trong:

Bước 1: AI  Moderator)đọc câu hỏi của bạn, viết lại thành một luận đề chính xác, định nghĩa các từ mơ hồ (*"giờ làm 4 ngày"* nghĩa là 32 giờ giữ nguyên lương? hay chỉ sắp lại 40 giờ vào 4 ngày?), và liệt kê những gì cả hai bên cần thừa nhận trước khi tranh luận. Bước này quan trọng để hai bên không "nói chuyện lệch nhau".

Bước 2: AI Persona Picker chọn ra hai nhân vật từ một thư viện gồm 12 nhân vật chuyên gia giả lập; ví dụ: một nhà kinh tế học tân cổ điển, một chuyên gia y tế công cộng, một nhà xã hội học phê phán, một luật sư hiến pháp, một doanh nhân vận hành, một nhà sử học... Hệ thống chọn sao cho hai nhân vật có *góc nhìn xa nhau nhất có thể*. Nếu một người nghĩ theo lối "hiệu quả kinh tế" thì người kia sẽ nghĩ theo lối "công bằng xã hội".

KFM - IT - Tài Văn - SC002120 🐳, [10/4/26 06:00]
Lý do: để buộc AI rút ra những lập luận *không giống nhau* từ kho tri thức của nó, thay vì cho ra hai bản na ná chỉ khác về câu chữ.

Bước 3: Hai AI tranh luận qua 4 vòng. Mỗi vòng hai bên chạy song song và độc lập (nghĩa là mỗi bên không được đọc bài của bên kia trong khi đang viết bài của mình):

- Vòng 1 - Mở đầu: mỗi bên trình bày 3 lập luận chính
- Vòng 2 - Phản bác: mỗi bên đọc lập luận đối phương và bác lại, đồng thời đặt 3-5 câu hỏi chất vấn
- Vòng 3 - Trả lời và phản bác tiếp: mỗi bên trả lời câu hỏi của đối phương và bảo vệ lập luận mình
- Vòng 4 - Kết luận: mỗi bên tóm tắt và giải thích vì sao mình nên thắng

Mỗi bài viết đều phải có một bảng gọi là verified_facts[] (bảng kiểm chứng sự kiện) liệt kê nguồn cho từng con số, từng trích dẫn - giống như footnotes trong một bài báo học thuật.

Bước 4 - AI Referee kiểm tra sau vòng 2 để xem có bên nào "chơi xấu" không:

- Họ có bóp méo lời đối phương trước khi phản bác không (gọi là *strawman* - "ngụy biện người rơm")?
- Họ có đi lạc đề không?
- Họ có lén đổi định nghĩa giữa chừng không?
- Họ có né tránh câu hỏi chất vấn không?

Nếu phát hiện vi phạm nghiêm trọng, bên sai phải viết lại phần rebuttal (tối đa 1 lần).

Bước 5 - AI FactCheck đọc toàn bộ các bài tranh luận và tự lên web tra từng số liệu, từng trích dẫn, từng sự kiện lịch sử mà hai bên nêu. Điểm đặc biệt: AI này không được phép tin đường link mà bên tranh luận cung cấp - nó phải tự search độc lập, tra lại từ đầu. Mỗi nguồn tìm được sẽ được xếp hạng theo độ tin cậy:

- Tier A - nguồn có thẩm quyền cao nhất: báo cáo chính phủ, nghiên cứu đã được phản biện học thuật (*peer-reviewed*), IPCC, WHO, số liệu của Tổng cục Thống kê
- Tier B - báo chí lớn có tiêu chuẩn biên tập (Reuters, AP, BBC, FT, Economist), think tank uy tín
- Tier C - tạp chí chuyên ngành, blog của chuyên gia có danh tính, Wikipedia
- Tier D - web chung chung, mạng xã hội, nguồn không rõ danh tính

Một sự kiện chỉ được đánh dấu "đã xác minh" nếu có ít nhất 2 nguồn độc lập, trong đó có ít nhất 1 nguồn ở Tier A hoặc B. Cuối cùng FactCheck xuất ra một bảng: cái nào đã xác minh ✅, cái nào sai ❌, cái nào chỉ đúng một phần ⚠️, cái nào không tra được ❓.

Bước 6 - Ẩn danh đổi nhãn (Blind Relabel): trước khi gọi các giám khảo vào chấm, hệ thống âm thầm đổi tên tất cả file "PRO" và "CON" thành "Side A" và "Side B" theo thứ tự ngẫu nhiên, rồi khóa bảng mapping vào một file riêng. Mục đích: ngăn giám khảo thiên vị chỉ vì thấy chữ "PRO" hay "CON", một hiệu ứng tâm lý mà con người (và AI) thường mắc phải.

Bước 7 - Ba giám khảo chấm điểm song song, mỗi người theo một lăng kính đánh giá khác nhau:

- Giám khảo U (Utilitarian - Vị lợi): chấm theo tiêu chí "ai tạo ra nhiều phúc lợi cho nhiều người hơn thì thắng". Quan tâm đến hậu quả, số liệu, quy mô tác động.

- Giám khảo R (Rights-based - Quyền con người): chấm theo tiêu chí "ai tôn trọng các quyền cơ bản và công bằng thủ tục thì thắng". Quan tâm đến quyền cá nhân, ngay cả khi hậu quả không tối ưu.

- Giám khảo P (Pragmatic - Thực tiễn): chấm theo tiêu chí "ai có giải pháp khả thi trong thực tế chính trị, thể chế, ngân sách thì thắng". Một giải pháp đẹp nhưng không thực hiện được sẽ bị đánh giá thấp.

Mỗi giám khảo cho ra một bảng điểm 9 tiêu chí và tuyên bố người thắng theo lăng kính của mình. Có ba giám khảo với lăng kính khác nhau là một thiết kế đẹp của hệ thống: nhiều khi Side A thắng theo góc vị lợi nhưng lại thua theo góc quyền con người - đây không phải là lỗi, mà là insight quan trọng nhất về bản chất của vấn đề.

Một điểm thú vị và đáng chú ý là hệ thống hiện đang có một catalog 30 lỗi lập luận** (fallacy catalog) -  trọng tài và giám khảo đều biết cách phát hiện:

- *Strawman* (bóp méo lời đối phương trước khi phản bác)
- *False dichotomy* (ép chọn một trong hai khi thực ra có nhiều lựa chọn hơn)
- *Slippery slope* (sợ hãi tưởng tượng về chuỗi hậu quả xa mà không có cơ sở)
- *Cherry-picking* (chỉ trích dẫn dữ liệu ủng hộ mình, lờ đi dữ liệu phản biện)
- Và 26 lỗi khác...

KFM - IT - Tài Văn - SC002120 🐳, [10/4/26 06:00]
Bước 8  - Meta-Judge đọc cả ba bảng điểm, tổng hợp lại (tính điểm trung bình và độ biến động giữa các giám khảo), mở khóa mapping ẩn danh để lộ Side A là PRO hay CON, rồi viết báo cáo cuối cùng. Điểm đặc biệt: báo cáo cuối có một phần gọi là Synthesis (Tổng hợp) với 4 mục bắt buộc - và đây là phần giá trị nhất của toàn bộ hệ thống, bao gồm các nôi dung:

- Chỗ cả hai bên đều đúng: lấy cái tốt nhất của mỗi bên và ghép lại thành một bức tranh đầy đủ hơn

- Bất đồng còn lại, chia rõ thành:
  + Bất đồng về dữ kiện - ví dụ: *"Thử nghiệm tuần 4 ngày ở Iceland có thực sự tăng năng suất không?"* → trả lời được bằng dữ liệu, một bên đúng một bên sai
   + Bất đồng về giá trị - ví dụ: *"Nghỉ ngơi có nên là một quyền cơ bản không?"* → không có dữ liệu nào giải quyết được, phụ thuộc vào hệ giá trị bạn tin

 - Câu hỏi còn bỏ ngỏ: những vấn đề cả hai bên đều không trả lời được, cần nghiên cứu thêm. 

-  Khuyến nghị hành động: cụ thể nếu bạn đang phải ra quyết định về vấn đề này, đây là điều bạn nên làm hoặc có thể làm hoặc xem xét làm.

E. Khi nào nên dùng hệ thống này?

Theo mình, hệ thống này phù hợp cho những lúc bản thân cần:

- Đánh giá một đề xuất chính sách 
- Kiểm tra sức mạnh một quyết định chiến lược trong kinh doanh hoặc pháp lý
- Nghiên cứu một vấn đề có tranh cãi về giá trị (đặc biệt về đạo đức)
- Học sâu về một lĩnh vực bằng cách đọc cuộc đối đầu giữa hai trường phái đối lập
- Chuẩn bị tài liệu phân tích cho cuộc họp, bản đề xuất, hoặc memo nội bộ

Lời cuối bài, mình cũng muốn làm rõ rằng mục tiêu cuối cùng của hệ thống này không phải là tìm ra câu trả lời tuyệt đối cho mọi vấn đề.  Mà để giúp một người biết suy nghĩ sẽ kết luận gì sau khi cân nhắc phiên bản mạnh nhất của cả hai bên, với tất cả sự thật đã được xác minh, và với ranh giới rõ ràng giữa bất đồng về dữ và bất đồng về giá trị,  mà vẫn phụ thuộc vào hệ giá trị của chính bản thân người đặt ra câu hỏi.

P/s 1: Mọi người có thể theo dõi 1 debate tại đây về topic "Mỹ tấn công Iran là đúng hay sai".  Khá hay, do có một số luận điểm mà mình thật sự không biết và không ngờ, khiến chính bản thân mình nghi ngờ về những gì mà mình biết và mình tin. https://drive.google.com/drive/folders/1zLUXlOGpGnyRWQ1ClZ9RhGsthKD9t186.  Files tại folder workspace là nội dung debate và đánh giá của các judges. Files tại folder output là kết quả và tổng hợp chung cuộc.  Nhưng mọi người nên đọc toàn bộ các files.

P/s 2: Mình cũng sẽ làm 1 clip về quá trình tạo ra hệ thống này chỉ để chứng tỏ cho mọi người hiểu thêm rằng, hệ thống tuy nghe phức tạp, rối rắm hay kể cả là cao siêu, nhưng với sức mạnh của AI (mà cụ thể ở đây là Claude, Claude Code và Claude Cowork), thì việc tạo nên nó hay những hệ thống tương tự khác là hoàn toàn có thể làm được bởi bất kỳ ai, kể cả những người không biết code.