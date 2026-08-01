---
title: "Swift Concurrency (Phần 2): 4 ý tưởng dưới nắp capo và 20 hệ quả thực chiến"
description: "Swift Concurrency đứng trên đúng 4 ý tưởng. Hiểu chúng rồi thì mọi thứ 'khó hiểu' trở thành hệ quả bạn tự suy ra được."
icon: "article"
date: "2026-08-02T00:30:00+07:00"
lastmod: "2026-08-02T00:30:00+07:00"
draft: false
toc: true
weight: 999
---

# Tác giả : ChungHA

# Swift Concurrency (Phần 2): 4 ý tưởng dưới nắp capo và 20 hệ quả thực chiến

> Bài này mình dịch và biên soạn lại (sát nghĩa) từ bài viết rất hay của **Lev Litvak** — *"Swift Concurrency: From Ideas Under the Hood to Practical Consequences"*. Link gốc để ở cuối bài. Đây là phần 2, nối tiếp [phần 1](../swift-concurrency/) mình đã viết.

Hầu như anh em nào cũng đã dùng Swift Concurrency đời mới rồi, hoặc ít nhất cũng nghịch thử. Mình biết `async/await` giúp code ngắn và phẳng hơn hẳn completion handler. Mình rải `@MainActor` chỗ này chỗ kia, viết một hai con `actor`, rồi kiểu gì đó cũng dẹp được mấy lỗi `Sendable`.

Nhưng **biết cú pháp không có nghĩa là hiểu hệ thống**. Tại sao compiler không cho truyền cái class này giữa các task? Tại sao một `actor` — sinh ra để bảo vệ state của mình — lại để state bị đổi ngay giữa method của chính nó? Tại sao chỉ một cái semaphore trông vô hại lại treo cả app? Chừng nào còn phải học thuộc câu trả lời cho từng case, thì Swift Concurrency vẫn cứ như một mớ luật lệ vô tội vạ.

Ý của bài này là: **Swift Concurrency đứng trên đúng một nhúm ý tưởng thôi, và một khi nhìn ra được chúng thì mấy thứ "khó hiểu" hết khó hiểu luôn** — chúng biến thành những hệ quả mà bạn tự suy ra được. Cái gì chạy kiểu đó cũng có lý do cả, và lý do đó thì hiểu được.

Có **4 ý tưởng**: (1) hàm thật ra chạy thế nào mà chẳng cần chờ, (2) isolation rốt cuộc là gì và actor thật sự bảo vệ cái gì, (3) `Sendable` chứng minh điều gì và chứng minh cho ai, và (4) tại sao task sống trong hai thế giới, mà một trong hai lại là một cái cây. Mỗi ý tưởng kết lại bằng một loạt **hệ quả**, và mỗi hệ quả là một tình huống lấy ra từ code thật. Cả bài có 20 hệ quả, đánh số liền một mạch từ đầu tới cuối.

Bắt đầu từ ý tưởng đầu tiên nhé, vì mọi thứ còn lại đều dựng lên trên nó.

## Ý tưởng 1: Không có chuyện "chờ". Chỉ có một hàm bị cắt thành từng mảnh.

Quên chữ "await" đi một lát. **Chẳng có gì trong Swift Concurrency thật sự "chờ" cả.** Muốn thấy chuyện gì đang thực sự diễn ra, mình cần hai mảnh kiến thức nền: thread là gì, và dữ liệu của một hàm nằm ở đâu.

**Thread là gì.** Thread là một "người thợ" mà hệ điều hành cấp cho bạn. Nó chạy lệnh lần lượt, theo thứ tự, và mỗi lúc chỉ làm được đúng một việc. CPU core mới là phần cứng thật sự chạy thread: một con chip 6 core chạy được 6 thread *cùng lúc*. Hệ điều hành tạo được nhiều thread hơn số core, nhưng khi đó nó phải xoay vòng đưa thread lên/xuống core, mà mỗi thread thì **đắt**: nó cần vùng nhớ riêng, OS lại tốn thêm công quản lý. Nhớ cái giá đó nhé, vì cả cái mô hình bên dưới sinh ra cũng chỉ vì nó thôi.

**Dữ liệu của hàm nằm ở đâu: stack và heap.** Một chương trình có hai loại bộ nhớ. **Stack** thuộc về một thread: mỗi thread có đúng một cái. Khi thread chạy một hàm, mấy biến cục bộ của hàm được đặt lên stack của thread đó, rồi xoá sạch ngay khi hàm return. Nhanh, nhưng dính hai ràng buộc cứng: dữ liệu chết cùng hàm, và chỉ đúng một thread đó với tới được. **Heap** thì ngược lại: một vùng dùng chung, chẳng thuộc thread nào. Dữ liệu ở đó sống chừng nào còn có ai giữ tham chiếu tới nó, và thread nào cũng với tới được. Đây là chỗ instance của class sống — nên một object mới truyền đi khắp nơi được, và sống lâu hơn cả cái hàm đẻ ra nó.

Có một trường hợp trộn cả hai, cần nói rõ vì lát nữa sẽ dùng tới: **một biến cục bộ kiểu class**. Viết `let client = APIClient()` trong một hàm là dữ liệu tách làm hai ngay. Bản thân object, kèm mọi property của nó, được tạo trong heap. Còn biến cục bộ `client` chỉ là một *tham chiếu* tới object đó thôi: một giá trị nhỏ xíu (thực ra là địa chỉ của object trong bộ nhớ) nằm trên stack của thread, y như mọi biến cục bộ khác. Hàm return thì tham chiếu trên stack bị xoá, nhưng object trong heap vẫn sống chừng nào còn ai đó giữ tham chiếu.

> *Sơ đồ: Mỗi thread có một stack riêng, chứa biến cục bộ của mấy hàm nó đang chạy. Heap là vùng dùng chung, chẳng thuộc thread nào.*

Nhớ kỹ chỗ khác biệt then chốt này: **stack gắn với một thread và một hàm đang chạy, còn heap thì chẳng gắn với cái nào trong hai.** Chỉ đúng một sự thật đó thôi cũng đủ giải thích gần hết phần còn lại của mục này.

**Compiler làm gì với một hàm async.** Khi bạn viết một hàm `async`, compiler **cắt nó thành từng mảnh**. Đặt tên cho mỗi mảnh để dùng xuyên suốt bài luôn: một **chunk** là một khối code đồng bộ (synchronous) liền mạch nằm giữa hai vết cắt. Mà vết cắt chỉ xảy ra tại `await` thôi, không phải ở mọi dòng:

```swift
func loadTrack() async throws -> AudioTrack {
    let logTag = "track-load"                    // chunk 1
    let streamURL = try await resolveStream()    // ← cắt: chunk 1 kết thúc bằng việc bắt đầu lời gọi,
                                                 //        chunk 2 bắt đầu bằng việc nhận streamURL
    logger.log("got \(streamURL)")               // chunk 2
    let bytes = try await fetchAudio(streamURL)  // ← vết cắt thứ hai
    let track = decodeAudio(bytes)               //
    metrics.record(track, tag: logTag)           // ] chunk 3
    return track                                 //
}
```

Một chi tiết để khỏi lẫn lộn chỗ vết cắt nằm đâu: dòng có `await` được **chia sẻ** giữa hai chunk. Việc cuối cùng của chunk 1 là *bắt đầu* lời gọi `resolveStream()`. Nếu kết quả chưa có sẵn ngay thì vết cắt xảy ra đúng ngay lúc đó. Việc đầu tiên của chunk 2 là *nhận* kết quả rồi nhét vào `streamURL`. Nên dòng `await` không hề "nằm ngoài" các chunk — nó chính là **ranh giới** giữa chúng.

Giờ tới chỗ rắc rối. `streamURL` được tạo ở chunk 1 nhưng lại dùng ở chunk 2. `logTag` tạo ở chunk 1 mà dùng còn muộn hơn nữa, tận chunk 3. Giữa mấy chunk đó hàm **hoàn toàn không chạy**, và cái thread từng chạy chunk 1 cũng chẳng ngồi đó chờ: nó bỏ đi chạy code khác luôn. Mà code khác đó cần chỗ trên stack, thế là nó chiếm đúng chỗ chunk 1 đang xài rồi ghi đè lên. Nên stack chẳng mang được gì qua cái khe hở đó, cùng lý do nó không mang nổi biến cục bộ của một hàm đã return: **stack thuộc về bất cứ thứ gì đang chạy ngay lúc này**. (Còn một lý do thứ hai nữa, mấy trang sau sẽ nói rõ: chunk phía sau khe hở thậm chí có thể chẳng chạy trên cùng thread, mà thread này thì không với tới stack của thread kia.) Thế nên compiler cất mấy biến còn sống vào một object đặc biệt, rồi quăng object đó vào **heap** — vùng duy nhất sống sót qua mọi thứ và không thuộc thread nào.

**Cái object đó chính là continuation.** Cho rõ ai nằm ở đâu: mấy biến (`streamURL`, `logTag`) được lưu *bên trong* continuation, còn bản thân continuation là một object trong heap. Cái tên này lấy từ khoa học máy tính và mang đúng nghĩa đen: continuation là "tất cả những gì còn phải làm tính từ điểm này trở đi". Về mặt vật lý nó giống hệt một closure: một mẩu bộ nhớ chứa (a) mấy biến cục bộ đã lưu và (b) địa chỉ của đoạn code chạy tiếp theo, tức chunk kế. Khi kết quả `await` về, runtime nhặt cái object này lên, đưa cho một thread, rồi thread nhảy tới địa chỉ đã lưu cùng đống biến đã lưu. Hành động đó gọi là **resume** continuation. Chẳng có gì huyền bí đâu: state đã lưu cộng thêm một cái "bookmark" ghi "chạy tiếp từ đây".

> *Sơ đồ: Một continuation = mấy biến đã lưu + địa chỉ chunk kế. Resume tức là đưa nó cho scheduler chạy chunk 2 ở bất cứ đâu.*

**Vậy sao lại gọi là "await" nếu chẳng có gì chờ?** Bởi vì *có* một thứ chờ thật đấy: **dòng thời gian logic của chính hàm bạn**. Từ góc nhìn của code bạn viết, dòng tiếp theo đúng là không chạy cho tới khi có kết quả. Timeline logic của hàm tạm dừng ngay tại đó. Thứ *không* chờ mới là **thread**. Cái tên đó mô tả góc nhìn từ bên trong hàm, chứ không phải bộ máy bên dưới, và Swift kế thừa nó từ C# với JavaScript. Một cái tên thành thật hơn cho bộ máy phải là "suspension point" (điểm tạm dừng) — và đó cũng đúng là thuật ngữ mà tài liệu chính thức của Swift dùng: *await đánh dấu một suspension point tiềm năng*. Tiềm năng, vì nếu kết quả tình cờ có sẵn ngay thì chẳng cần cắt gì cả, hàm cứ thế chạy tiếp.

**Ai chạy mấy cái chunk.** Swift Concurrency chạy chúng trên một **cooperative thread pool**: một nhóm thread do hệ thống lập riêng để chạy chunk, cỡ khoảng **một thread trên mỗi CPU core**, và con số đó **không bao giờ tăng thêm**. Một chiếc iPhone đời mới có 6 core (con A19 Pro của iPhone 17 Pro: 2 performance + 4 efficiency), nên pool cỡ 6 thread. Ngược hẳn với mô hình GCD cũ, nơi một thread bị block là hệ thống đẻ thêm cái nữa, rồi cái nữa, lên tới tận 64 thread — cái vụ gọi là *thread explosion* đó, mỗi thread lại ngốn bộ nhớ và chi phí lập lịch của OS. Cooperative pool chọn thoả thuận ngược lại: số thread nhỏ và cố định, đổi lại **code của bạn tuyệt đối không được block một thread trong pool**. Chẳng có gì ép buộc chuyện này đâu: compiler vẫn cho bạn làm, runtime cũng chẳng can thiệp. Giữ lời hứa đó là việc của bạn, và mấy hệ quả bên dưới chính là những gì xảy ra khi lời hứa bị phá.

**Pool dính dáng thế nào với đám thread còn lại.** Thread trong pool chẳng phải phần cứng gì đặc biệt, cũng không phải một loại thread riêng biệt. Một app có cả đống thread: main thread (thread vẽ UI), khoảng 6 thread của cooperative pool, cộng thêm thread do GCD tạo, do bộ máy networking, do thư viện bên thứ ba. Tất cả đều là thread OS bình thường, và OS scheduler rải hết chúng lên cùng 6 core vật lý. Nên thread pool **không sở hữu** core nào cả. Cái khiến chúng thành "pool" chỉ là công việc và luật lệ của chúng thôi: chúng là mấy thread mà Swift Concurrency dùng để chạy chunk, và chúng theo giao kèo "không bao giờ block". Main thread **không** nằm trong nhóm này: nó tồn tại riêng, lo chạy UI, và Swift Concurrency coi nó là một executor tách biệt (cái này quan trọng ở mục sau, lúc `@MainActor` xuất hiện).

Để ý cái điều mà mô hình pool ngụ ý: **một hàm đang suspended chẳng có "thread nhà" nào cả**. Nó không phải kiểu "đang tạm dừng trên thread 4, tí nữa quay lại đó". Lúc suspended, nó là một object trong heap, và câu hỏi "nó thuộc thread nào" *không có câu trả lời*, y hệt như mọi object khác trong heap. Không có thread affinity — cố ý thiết kế vậy luôn. (Main thread và `@MainActor` là ngoại lệ đặc biệt, có luật riêng, sẽ nói kỹ ở mục sau.)

> *Sơ đồ: Toàn bộ mô hình trên thiết bị 6 core — cắt tại mỗi await, state nằm trong continuation, thread nào rảnh cũng chạy được chunk kế tiếp.*

Bốn câu hỏi bật ra rất tự nhiên trước khi đi tiếp.

**Nếu số task nhiều hơn số thread thì sao?** Đó là chuyện bình thường, chẳng phải vấn đề gì. Một task đang suspended là object heap cỡ vài trăm byte, và nó **chẳng chiếm thread nào cả**. Mười ngàn task trên sáu thread là chuyện thường ngày: mấy chunk sẵn sàng nằm trong hàng đợi của scheduler, thread nhặt lần lượt, cái nào ưu tiên cao thì trước. So sánh cho dễ hình dung: một thread cần cỡ nửa megabyte stack, cộng thêm một chuyến vào kernel mỗi lần bị switch. Chính cái sự lệch nhau đó là toàn bộ lý do mô hình này ra đời: cả đống task suspended rẻ bèo, ghép lên vài cái thread đắt đỏ.

**6 thread nghĩa là mỗi lúc chỉ tải được 6 bài?** Không, và lý do sẽ làm rõ luôn thread thực ra dùng để làm gì. Thread chỉ cần cho đúng một việc: **thực thi code**, tức chạy lệnh CPU. Chunk của hàm async chạy trên thread pool, nhưng "chạy code thì cần thread" đúng với *mọi* mẩu code trong hệ thống. Đây là một lượt tải, mình chú thích rõ chỗ nào thật sự diễn ra ở đâu:

```swift
func fetchArticle(_ url: URL) async throws -> Article {
    let request = buildRequest(url)       // thread pool, chỉ một phần nghìn giây

    let (data, _) = try await URLSession.shared.data(for: request)
    // ← hàm suspend tại đây.
    //   Trong lúc truyền dữ liệu (99% thời gian) KHÔNG có code nào chạy cho lượt tải này,
    //   trên bất kỳ thread nào. Byte được chip mạng và OS chuyển đi.
    //   Khi response sẵn sàng, code hệ thống chạy thoáng qua trên một trong các
    //   "thread khác" ở sơ đồ trên và resume continuation của ta.

    return parseArticle(data)             // lại thread pool: việc CPU thật sự
}
```

Cái nghe chối tai là chỗ "không có code nào chạy". Vậy chứ *ai* đang chờ dữ liệu? Chẳng ai cả, theo đúng nghĩa đen. Hệ thống đời mới không cài đặt "chờ" bằng cách bắt một thread đứng trong vòng lặp hỏi đi hỏi lại "xong chưa?". Chúng làm kiểu **chuông cửa**. Yêu cầu được đẩy xuống OS rồi tới chip mạng — một thiết bị vật lý riêng, tự nó chuyển byte, chẳng cần CPU làm gì. OS ghi chú lại "khi nào dữ liệu của request này về thì báo cho app", và cho tới khi chuông reo (một tín hiệu phần cứng gọi là *interrupt*), thì **không tốn một lệnh nào** cho lượt tải này hết.

Giờ tính sổ cho 100 lượt tải cùng lúc: dòng đầu với dòng cuối của `fetchArticle` là mấy khoảnh khắc ngắn ngủi chạy code, còn phần truyền dữ liệu — chiếm gần như toàn bộ thời gian — tốn **0 thread và 0 CPU**. Đó là lý do cả 100 lượt truyền diễn ra song song thật sự. Thứ duy nhất bị chặn quanh mức 6 là **thực thi code đồng thời**: khi cả 100 bài về và cần parse, mấy chunk `parseArticle(data)` sẽ chạy tầm 6 cái một lúc. Mà cái trần đó không phải điểm yếu của mô hình đâu, nó là phần cứng: chip có 6 core, nên quá 6 phép tính thì vốn dĩ chẳng bao giờ chạy cùng một khoảnh khắc được. Thêm thread cũng chẳng tính nhanh hơn, chỉ tổ khiến chúng thay phiên nhau trên đúng 6 core đó mà lại còn phải trả thêm giá switch.

**Còn một hàm trộn lẫn cả hai loại việc thì sao?**

```swift
func prepareUpload() async throws -> Payload {
    let raw = try await api.fetchBlob()   // vết cắt của riêng nó ở đây
    return encrypt(raw)                   // chunk của riêng nó: việc CPU thuần
}

// Phía người gọi chỉ thấy một await:
let payload = try await prepareUpload()
```

Phía người gọi chỉ thấy đúng một `await`, và người gọi bị suspend suốt cả quãng đó. Nhưng cái `await` đó chẳng nói lên được gì về chuyện xảy ra bên trong. Bên trong `prepareUpload`, đúng cái mô hình đó lặp lại một cách đệ quy: nó có vết cắt riêng tại `api.fetchBlob()` và các chunk riêng. Chunk khởi động request chạy trên thread pool, rồi suspend để truyền dữ liệu (không tốn thread, như mình vừa thấy), rồi chunk `encrypt` chiếm thật một thread pool, vì mã hoá là việc CPU thuần túy. Nên đúng như bạn đoán: một phần vòng đời của hàm này xài thread pool, phần còn lại chẳng xài gì, dù người gọi chỉ thấy một `await` liền một mạch. **Await một hàm chỉ có nghĩa là "timeline của tôi tạm dừng cho tới khi nó return".** Còn nó xài bao nhiêu thread bên trong, và xài khi nào, thì mấy vết cắt của chính nó quyết định.

**Vậy khi nào thì toang?** Chỉ khi một chunk *đang nằm trên* một thread mà lại không chịu nhả nó ra: block nó, hoặc ôm nó quá lâu. Cái đó phá vỡ giao kèo của pool, và mấy hệ quả bên dưới đều là biến thể của đúng một vi phạm này thôi, cộng thêm vài hệ quả trực tiếp từ chuyện "không có thread nhà".

### Hệ quả 1. Sau await, bạn có thể tỉnh dậy trên một thread khác.

```swift
func trace() async {
    logCurrentThread()   // <NSThread: 0x...>{number = 4, ...}
    try? await Task.sleep(for: .seconds(1))
    logCurrentThread()   // <NSThread: 0x...>{number = 7, ...}
}

// Một helper đồng bộ, và nó cần thiết có chủ đích: xem lưu ý bên dưới.
func logCurrentThread() {
    print(Thread.current)
}
```

Cùng một hàm, mà thread khác nhau, và đây là hành vi **đúng** nhé: sau vết cắt, chunk kế nhảy tới bất cứ thread pool nào rảnh trước. Còn về cái helper — nó chứng minh luận điểm của mục này còn ngon hơn cả ví dụ: trong Swift 6 language mode, gọi thẳng `Thread.current` trong code async là **không compile được** luôn. Foundation đánh dấu nó không khả dụng từ ngữ cảnh async, chính vì câu trả lời có thể đổi tại mỗi `await`, và ngôn ngữ từ chối cho bạn dựa dẫm vào nó. Hỏi vòng qua một hàm đồng bộ chỉ là cách lách để demo thôi.

Một lưu ý về mấy con số, vì đứa nào nhìn cũng bị lú: `number = 7` **không** có nghĩa là pool có ít nhất 7 thread đâu. Con số đó chỉ là một cái định danh giữa *tất cả* thread của tiến trình, và như đã thấy, một app đang chạy có cả đống thread nằm ngoài pool. Khoảng 6 thread của pool thì mang bất cứ số nào mà chúng tình cờ nhận được. Với lại, tỉnh dậy đúng thread cũ là chuyện *có thể*, nhưng đó là trùng hợp thôi, không bao giờ là đảm bảo. Đây chính là lý do thread-local storage và mọi thứ đánh chỉ mục theo "thread hiện tại" đều **toang** khi qua `await`. (Có một ngoại lệ: code isolated tới `@MainActor` thì *luôn* resume trên main thread. Cái đó không phải thread affinity lẻn về bằng cửa sau, mà là **isolation** — chủ đề của mục sau.)

### Hệ quả 2. Đừng bao giờ giữ một lock qua await.

```swift
let mutex = NSLock()

func refreshToken() async {
    mutex.lock()
    let token = await requestToken()   // cắt: chunk 2 có thể chạy trên thread khác
    stored = token
    mutex.unlock()                     // có thể bị gọi từ một thread chưa từng lock
}
```

Một cái lock kiểu mutex thì *nhớ* thread nào đã lock nó và đòi `unlock` phải từ đúng thread đó. Chunk 2 có thể chạy trên thread khác, nên `unlock()` phá vỡ cái kỳ vọng đó, và tài liệu Apple nói thẳng luôn hậu quả: unlock một lock từ thread khác là **undefined behavior**. Ngoài thực tế, cái "undefined" đó diễn ra thành một trong ba kết cục, từ ngon nhất tới tệ nhất. **Ngon nhất:** runtime phát hiện cú unlock lạ và crash tiến trình ngay lập tức (`os_unfair_lock` làm đúng vậy, kèm thông báo rõ ràng). Khó chịu, nhưng bạn tóm được bug ngay lần chạy test đầu tiên. **Ở giữa:** cái bản ghi sở hữu nội bộ của lock bị hỏng, rồi một cú `lock()` nào đó về sau, trông vô hại thôi, lại **deadlock vĩnh viễn**, thế là bạn đi debug nhầm chỗ. **Tệ nhất:** nó âm thầm *trông như* chạy ngon trên máy bạn, trên phiên bản OS của bạn, rồi ship thẳng ra production, xong toang theo một trong hai kiểu trên ngay trên máy người khác. Bug giờ vô hình trong code của bạn và chẳng cách nào tái hiện nổi trên máy bạn. Và không liên quan gì tới cả ba kết cục kia: trong lúc lock bị giữ qua suspension, mọi thread khác muốn nó đều bị block — mà riêng cái đó thôi đã là điều cấm rồi. **Lock dùng trong code async thì ổn, nhưng chỉ giữa hai `await` thôi, không bao giờ được vắt qua một cái.**

### Hệ quả 3. Một semaphore có thể treo cả app.

```swift
func fetchSync() -> Config? {
    let sem = DispatchSemaphore(value: 0)
    var result: Config?
    Task {
        result = await fetchConfig()
        sem.signal()
    }
    sem.wait()   // block thread hiện tại cho tới khi signal() được gọi
    return result
}
```

Semaphore là một blocking primitive: `wait()` chặn cái thread gọi nó cho tới khi có ai đó gọi `signal()`. Kế hoạch ở đây là "khởi động việc async, block cho tới lúc xong, trả kết quả về theo kiểu đồng bộ". Nhưng đây là cái bẫy: nếu bản thân `fetchSync` chạy trên một thread pool, thì `sem.wait()` rút luôn cái thread pool đó ra khỏi vòng phục vụ. Gọi nó từ đủ nhiều chỗ cùng lúc là **mọi** thread pool đều kẹt cứng trong `wait()`. Mấy chunk của `fetchConfig()` sẵn sàng chạy rồi đấy, nhưng muốn chạy chúng thì cần một thread pool rảnh, mà chẳng còn cái nào, và pool thì **không thể tăng thêm** để cứu bạn. Chẳng ai chạm tới được `signal()` cả. Trên máy 2 core, chỉ cần hai lời gọi cùng lúc là đủ đóng băng sạch mọi thứ. Đây là kiểu **deadlock production hay gặp nhất** trong mấy codebase bắc cầu giữa concurrency cũ và mới theo kiểu này. (Một mẹo debug hay phết: có một biến môi trường thu pool xuống còn đúng một thread lúc test, khiến mọi vi phạm kiểu này lòi ra ngay tức khắc.)

### Hệ quả 4. Thread.sleep cướp một core, Task.sleep tốn 0 đồng.

```swift
// Trong một vòng poll đợi job xong:
Thread.sleep(forTimeInterval: 1)        // thread này bị chiếm để ngồi không trong 1s
try await Task.sleep(for: .seconds(1))  // không thread nào bị dính trong 1s;
                                        // một timer sẽ đánh thức hàm dậy
```

Trước tiên, xem mỗi dòng làm gì. `Thread.sleep(forTimeInterval: 1)` bảo OS: cho *thread hiện tại* ngủ một giây. Thread chẳng làm gì, nhưng vẫn **bị chiếm**: suốt cả giây đó nó không chạy nổi chunk của ai khác. Còn `Task.sleep(for: .seconds(1))` suspend cái *hàm*, chứ chẳng phải thread nào: một vết cắt bình thường, continuation quăng vào heap, cộng một ghi chú trong timer của scheduler "một giây nữa thì đưa continuation này về lại hàng đợi sẵn sàng". Thread được nhả ra ngay tại vết cắt, rồi dành cả giây đó chạy chunk của task khác, hoặc nghỉ luôn nếu chẳng có việc. Task đang ngủ tốn **0 thread-time**. Thread cũng không "chờ để quay lại với nó" đâu, vì như mọi khi, chẳng có thread nào gắn với một hàm đang suspended cả.

Vì sao đây lại là hệ quả của mô hình: pool có khoảng 6 thread và chẳng bao giờ đẻ thêm cái thứ bảy. Một thread kẹt trong `Thread.sleep`, trong cái giây đó, chính là một **core bị mất**: một phần sáu toàn bộ sức tính toán của app đem đi ngồi không. Nhìn từ ngoài thì hai dòng trông y hệt nhau ("code dừng một giây"), và đó đúng là cái làm dòng đầu tiên nguy hiểm.

### Hệ quả 5. Thứ tự thực thi giữa các task không được đảm bảo.

```swift
for jobID in 1...5 {
    Task { print(jobID) }
}
// In 1...5 theo thứ tự bất kỳ.
```

Chunk ở đây nằm đâu? Mỗi thân `Task { }` tự nó được lập lịch như một chunk: tạo một task tức là "quăng code này vào hàng đợi của scheduler", còn một thân không có `await` bên trong thì đơn giản là một task gồm đúng một chunk. Vậy là năm thân task rơi vào scheduler, thread nhặt chúng lên theo ưu tiên và độ sẵn sàng, **chẳng hứa hẹn** cái nào tạo trước chạy trước. Scheduler đâu phải serial queue: khác với `DispatchQueue.main.async` vốn đảm bảo thứ tự FIFO, `Task { }` chỉ đảm bảo code *sẽ* chạy, chứ không đảm bảo *khi nào* so với đám anh em của nó. Cần thứ tự thì phải lấy nó một cách tường minh: `await` từng phần việc lần lượt, hoặc đẩy chúng qua một `AsyncStream` (nó giao giá trị theo đúng thứ tự chúng được tạo ra).

Một cảnh báo thực tế trước khi bạn tự chạy thử đoạn này, và nó minh hoạ cực hay chuyện cái đảm bảo thứ tự hẹp tới mức nào. Trong một project bật default của Swift 6.2 (xem Hệ quả 11), cái vòng lặp này nhiều khả năng đang nằm trong code của main actor, năm task khi đó xếp hàng trên main actor theo thứ tự, và main actor chạy job đúng theo thứ tự chúng được xếp, nên bạn sẽ thấy 1 tới 5 gọn gàng. Nhưng đó là tính chất của **main actor**, chứ không phải của `Task { }`. Bê nguyên cái vòng lặp đó vào một actor của riêng bạn thì cái output gọn gàng biến mất luôn, vì một lý do rất hay làm người ta sập bẫy: một `Task { }` viết bên trong method của actor chỉ *gia nhập* actor đó **nếu thân của nó có chạm vào actor**. Ở đây thân chỉ có mỗi `print(jobID)`, chẳng chạm gì, nên không có gì để gắn vào và task đi thẳng ra pool, nơi làm gì có thứ tự nào tồn tại. Mà kể cả với task *thuộc về* một actor thì lời hứa vẫn hẹp: actor đảm bảo mấy job của nó không bao giờ chồng lấn, chứ không đảm bảo thứ tự chạy, và nó còn có thể chạy job ưu tiên cao trước. Tóm lại, **actor không bao giờ hứa về thứ tự, nên đừng bao giờ dựa vào bất kỳ thứ tự nào.**

### Hệ quả 6. withCheckedContinuation trao continuation vào tay bạn, kèm nghĩa vụ.

Mọi thứ từ nãy tới giờ đều mặc định là compiler tự quản lý continuation một cách vô hình. Nhưng có đúng một API mà bạn trực tiếp chạm tay vào chúng: lúc bắc cầu cho code callback cũ.

```swift
func currentLocation() async -> Coordinate {
    await withCheckedContinuation { continuation in
        legacyLocate { coordinate in
            continuation.resume(returning: coordinate)
        }
    }
}
```

Compiler cắt `currentLocation` tại `await` như thường lệ rồi tạo continuation: state đã lưu cộng cái bookmark trỏ tới phần còn lại của hàm. Nhưng nó chẳng đoán được callback cũ của bạn bao giờ mới nổ, nên thay vì tự resume, nó trao luôn cái object continuation cho closure của bạn. Gọi `resume(returning:)` tức là: "kết quả đây rồi, lên lịch chạy phần còn lại của cái hàm đang suspended ngay bây giờ đi".

Cái này biến luật "gọi resume đúng một lần" từ thứ phải học thuộc thành thứ bạn tự suy ra được. Gọi **không lần nào**, thì cái object continuation nằm ì mãi trong heap: phần còn lại của `currentLocation` chẳng bao giờ chạy, đứa nào await `currentLocation` là bị suspend vĩnh viễn, và mọi thứ continuation đang giữ đều leak sạch. Gọi **hai lần**, thì bạn đang bảo runtime chạy lại phần còn lại của một hàm đã resume rồi, cái đó có thể làm hỏng dữ liệu chương trình. Biến thể `checked` bạn thấy ở trên thêm vài cái kiểm tra để bắt cả hai lỗi: nếu continuation bị vứt đi mà chưa từng resume, runtime log ra một cảnh báo; còn nếu nó bị resume lần thứ hai, runtime dừng chương trình bằng một fatal error có chủ đích, thay vì để mặc dữ liệu hỏng. Chính cái sự bảo vệ đó là lý do nên chọn `withCheckedContinuation` làm mặc định thay cho người anh em `unchecked`, dù có tốn chút overhead nhỏ.

**Chốt lại.** Chỉ một mô hình thôi: hàm bị cắt tại mấy chỗ `await`, state sống trong continuation ở heap, một pool thread nhỏ và cố định chạy bất cứ chunk nào sẵn sàng. Mọi luật trong mục này (không block, không giữ lock qua `await`, không phán đoán gì về thread, resume đúng một lần) đều chỉ là cái mô hình đó áp dụng vào, chứ không phải mấy sự thật rời rạc phải học thuộc từng cái. Cái mà mô hình *chưa* trả lời: nếu chunk nào cũng chạy được trên thread nào, vào lúc nào, thì ai bảo vệ dữ liệu của bạn khỏi bị hai chunk chạm cùng lúc? Đó là **isolation**, ý tưởng tiếp theo.

## Ý tưởng 2: Isolation là quyền sở hữu dữ liệu, không phải điều khiển thread.

Ý tưởng 1 để lại cho mình một mối nguy. Chunk nào cũng có thể nhảy lên thread pool bất kỳ vào lúc bất kỳ, nghĩa là hai chunk hoàn toàn có thể thò tay vào cùng một biến cùng một lúc. Thử xem cái giá phải trả, với ví dụ nhỏ nhất có thể.

```swift
var votes = 0

// hai chunk chạy cùng lúc trên hai thread, mỗi cái làm:
votes += 1
```

`votes += 1` nhìn như một hành động, nhưng thực ra là ba: đọc giá trị đang có, cộng một, rồi ghi kết quả về. Hai thread hoàn toàn có thể xen kẽ ba bước đó. Cả hai cùng đọc 0, cả hai cùng cộng một, cả hai cùng ghi 1, thế là một lượt bình chọn bay màu vĩnh viễn. Đây là **data race**, mà trong Swift nó không chỉ dừng ở mức "thi thoảng ra số sai" đâu. Ngôn ngữ tuyên bố thẳng data race là **undefined behavior**, cùng hạng với vụ unlock lạ ở Hệ quả 2: cái gì cũng có thể xảy ra, kể cả hỏng bộ nhớ, và nó hiếm tới mức có thể lọt qua sạch mọi lần test của bạn.

**Cách cũ và cách mới.** Cách cổ điển để chặn chuyện này là đi điều khiển *thread*: bọc biến trong một lock, hoặc dồn hết mọi truy cập qua một serial queue, rồi *cầu trời* là mọi chỗ trong codebase đều nhớ làm đúng như vậy. Chính cái phần "cầu trời" mới là vấn đề. Chẳng có gì cản một đồng đội mới (hay chính bạn, sáu tháng sau) thò tay vào `votes` trực tiếp cả, mà compiler thì im re. Swift Concurrency lật ngược hẳn cách tiếp cận. Thay vì đi quản thread nào chạy lúc nào, nó **gán cho mỗi mẩu state thay đổi được một người chủ**, rồi bắt compiler soi quyền sở hữu trên từng dòng, ngay tại compile time. Người chủ đó gọi là **isolation domain**: một tập dữ liệu kèm một luật cực đơn giản — *chỉ code thuộc domain này mới được đụng vào dữ liệu này*. Mọi function và closure trong chương trình đều thuộc đúng một domain (hoặc không thuộc domain nào, cái này cũng là một trạng thái rõ ràng). Compiler biết domain của từng dòng lẫn domain của từng biến, nên hễ có dòng nào thò tay qua ranh giới là nó **từ chối compile** luôn.

**Actor là một cỗ máy đẻ ra domain.** Khai một cái là bạn có nguyên gói: state, code sở hữu nó, và cả phần cưỡng chế đi kèm.

```swift
actor VoteBox {
    var votes = 0           // state do domain này sở hữu

    func cast() {           // code thuộc về domain này
        votes += 1          // được phép: cùng domain
    }
}
```

Mọi thứ khai bên trong actor đều thuộc domain của nó. Code bên ngoài thì không, nên nó không thể với tới `votes` như với tới property của một class thường được. Nó vẫn lấy được giá trị, nhưng phải đi qua **cửa trước** của actor, kèm `await` — cái cửa đó làm gì thì mình sẽ thấy ngay đây. Tới đây thì mới chỉ là dán nhãn lên code với dữ liệu thôi. Phần hay ho nằm ở chỗ truy cập từ bên ngoài chạy ra sao: **actor đảm bảo mỗi lúc nhiều nhất một job chạy bên trong domain**. Một lời gọi từ ngoài biến thành một job trong hàng đợi của actor, các job vào từng cái một, cái nào đang ở trong thì mấy cái còn lại đứng xếp hàng chờ. Thế là data race chết: cái chuỗi đọc-cộng-ghi hết cửa xen kẽ với ai, đơn giản vì lúc đó chẳng có ai khác ở trong phòng cả.

> *Sơ đồ: Người gọi xếp hàng ngoài cửa. Mỗi lúc chỉ một job ở trong, nên chẳng gì xen kẽ được với nó. Job đang ở trong vẫn chạy chunk của mình trên các thread pool bình thường.*

**Actor không phải là một thread.** Đây là chỗ hiểu nhầm phổ biến nhất, nên mình mổ xẻ luôn. Actor không sở hữu thread, không tạo thread, và method của nó cũng chẳng chạy "trên thread của actor", vì làm gì có cái thread nào như thế tồn tại. Actor chỉ gồm hai thứ: một hàng đợi ở cửa, và cái luật "một job bên trong". Cái job đang ở trong vẫn chạy các chunk của nó trên đúng cái cooperative pool từ Ý tưởng 1, trên bất cứ thread nào đang rảnh. Mười ngàn actor trên 6 thread vẫn ngon lành, cùng lý do mười ngàn task vẫn ngon: một actor mà ngoài cửa không ai xếp hàng thì chỉ là một object nằm trong heap, chẳng tốn thread nào cả.

```swift
actor VoteBox {
    var votes = 0
    func cast() {
        votes += 1
        print(Thread.current)   // các lời gọi khác nhau chạy trên các thread pool khác nhau
    }
}
```

Các lời gọi vẫn chạy từng cái một, dù thread thì cứ đổi xoành xoạch. Cái bảo vệ state là **hàng đợi ở cửa**, chứ không phải bản thân mấy con thread.

**Vì sao gọi actor lại cần await.** Từ bên ngoài, mọi lời gọi đều phải qua cửa, mà cửa thì có thể đang bận. Nên lời gọi là một suspension point tiềm năng, và Ý tưởng 1 đã nói rõ điều đó nghĩa là gì: hàm của bạn có thể bị cắt ngay tại đây, biến thành continuation trong heap, rồi nhả thread ra. **Chẳng có con thread nào đứng xếp hàng ở cửa đâu.** Hàng đợi là một hàng các continuation, thế nên một ngàn người gọi cùng chờ một actor cũng chẳng tốn gì.

```swift
let current = await voteBox.votes   // ngay cả đọc một property từ bên ngoài
await voteBox.cast()                // vượt ranh giới = suspension point tiềm năng
```

**`@MainActor` cũng là cỗ máy đó, thêm một tính chất đặc biệt.** Main actor là một actor toàn cục dùng chung, mà cửa của nó dẫn tới đúng một executor cụ thể: **main thread**. Đánh dấu code là `@MainActor` thì nó thuộc domain của main actor, tức là hai chuyện xảy ra cùng lúc: nó tuân theo cửa (một job mỗi lúc), và các chunk của nó chạy cụ thể trên main thread chứ không phải trên pool. Nhớ cái ngoại lệ ở Hệ quả 1: code `@MainActor` *luôn* resume lại trên main thread sau `await`. Giờ thì thấy vì sao rồi. Đó không phải thread affinity lẻn về, mà đơn giản là cửa của main actor dẫn tới đúng một người thợ — main thread. Khác biệt giữa "`@MainActor`" và "main thread" chính là khác biệt giữa một khái niệm compile-time và một người thợ vật lý: domain là tập dữ liệu với code mà compiler soi tư cách thành viên, còn main thread là người thợ mà cửa của main actor tình cờ dẫn tới. UIKit với SwiftUI được đánh dấu `@MainActor` chính là để biến cái luật "UI phải được đụng từ main thread" — vốn là luật runtime mà bạn chỉ phát hiện ra qua mấy lần crash — thành một luật compile-time để compiler cưỡng chế giúp bạn.

**`nonisolated` là lối ra tường minh.** Đôi khi một method của actor chẳng đụng gì tới state thay đổi được, thế mà bắt mọi người gọi phải qua cửa (và qua `await`) thì vô nghĩa. Đánh dấu `nonisolated` là gỡ nó khỏi domain: nó gọi được từ bất cứ đâu mà **khỏi cần `await`**, đổi lại compiler cấm nó đụng vào state thay đổi được của actor. Đọc mấy hằng `let` bất biến thì vẫn thoải mái, vì dữ liệu đã không bao giờ đổi thì lấy đâu ra race.

```swift
actor VoteBox {
    let pollID: String   // bất biến
    var votes = 0

    nonisolated func describe() -> String {
        "Poll \(pollID)"   // ổn: pollID không bao giờ đổi
        // "votes: \(votes)" sẽ không compile: state thay đổi được, sai domain
    }
}
```

Giờ tới các hệ quả. Mình đánh số tiếp từ Ý tưởng 1, vì tất cả đều là hệ quả của cùng một mô hình. Cái đầu tiên ở đây đứng sau kha khá bug actor ngoài đời thực.

### Hệ quả 7. Actor reentrancy: cửa bảo vệ chunk, không phải cả method.

Trước hết, vì sao cái này *suy ra được từ các ý tưởng* chứ không phải một luật riêng phải học thuộc. Từ Ý tưởng 1: tại một `await`, hàm bị cắt, biến thành continuation trong heap rồi nhả thread, và không gì trong mô hình được phép ngồi block chờ cả. Giờ thử hỏi: cửa của actor *nên* làm gì trong lúc job bên trong đang suspend tại một `await` để chờ một lời gọi mạng? Nếu cửa cứ khoá chặt, actor sẽ làm đúng cái điều bị cấm: ngồi block chờ việc bên ngoài, chặn cửa mọi người, chẳng được tích sự gì, có khi cả mấy giây. Nên **cửa mở ra mỗi khi job bên trong suspend**. Tức là đơn vị mà actor serialize là **chunk**, chứ không phải cả method. Mọi thứ bên dưới là cái giá của chuyện đó.

Đây là một đợt flash sale, kiểm tra tồn kho trước khi trừ hàng, y như mọi sàn thương mại điện tử vẫn làm.

```swift
actor FlashSale {
    var stock = 100

    func purchase(_ qty: Int) async -> Bool {
        guard stock >= qty else { return false }        // kiểm tra: đủ hàng không
        let approved = await payment.authorize(qty)     // ← rời khỏi phòng ở đây
        guard approved else { return false }
        stock -= qty                                    // trừ kho
        return true
    }
}
```

Gọi `purchase(70)` hai lần cùng lúc, tồn kho đang là 100. Trực giác bảo actor serialize các lời gọi nên lần thứ hai phải fail. Thực tế: **cả hai đều thành công, và tồn kho tụt xuống -40.**

Cái suy luận ở trên giải thích hết. Khi lời gọi đầu chạm `await payment.authorize`, nó suspend: biến thành continuation và **rời khỏi phòng**. Cửa giờ trống trơn. Lời gọi `purchase(70)` thứ hai bước vào, kiểm tra `stock >= 70` với con số 100 vẫn còn nguyên, qua ngon, rồi cũng suspend tại `authorize`. Sau đó cả hai được duyệt, cả hai continuation quay lại (từng cái một, cửa vẫn hoạt động đàng hoàng), và cả hai cùng trừ 70. Từng chunk lẻ đều được serialize hoàn hảo, mà logic vẫn toang, vì **cái giả định trước `await`** ("tồn kho ít nhất là 70") không còn đúng sau `await` nữa.

Tính chất này gọi là **reentrancy**: trong lúc một job đang suspend tại `await`, actor có thể chạy các job khác. Luật rút ra: **sau mỗi `await` bên trong một actor, state của bạn có thể đã đổi, nên mọi giả định phải kiểm tra lại từ đầu.**

```swift
func purchase(_ qty: Int) async -> Bool {
    guard stock >= qty else { return false }
    let approved = await payment.authorize(qty)
    guard approved, stock >= qty else { return false }   // kiểm tra lại sau await
    stock -= qty
    return true
}
```

### Hệ quả 8. Code đồng bộ trong actor là một transaction, và await kết thúc nó.

Đây là suy luận của Hệ quả 7 đọc ngược lại. Cửa chỉ mở khi job bên trong suspend. Ý tưởng 1 nói suspension chỉ xảy ra tại một `await`, vì đó là chỗ duy nhất compiler cắt. Không `await` thì không cắt, không cắt thì không mở cửa. Nên **mọi thứ nằm giữa hai `await` chạy một mạch, chẳng job actor nào chen ngang được**. Một khối code đồng bộ bên trong actor thực chất là một **transaction**: đọc, quyết định, sửa, không ai xen vào được. Nên chiêu thực chiến khi thiết kế actor là: giữ mỗi chuỗi "quyết định rồi sửa" không có `await` nào ở giữa, và coi mỗi `await` là một ranh giới transaction. Nếu phần kiểm tra với phần mutation nằm gọn trong cùng một khối đồng bộ, cái bug oversell ở trên là chuyện **bất khả thi**.

```swift
func purchaseIfPossible(_ qty: Int) -> Bool {   // hoàn toàn không có await bên trong
    guard stock >= qty else { return false }
    stock -= qty                                // kiểm tra + trừ kho: một transaction
    return true
}
```

### Hệ quả 9. Đừng ghé thăm actor cho từng việc nhỏ: mỗi cú "nhảy" đều có giá.

Cái này suy thẳng ra từ "vượt ranh giới là một suspension point tiềm năng". Mỗi lần vượt ranh giới domain đều có thể suspend bạn, đẩy một continuation qua scheduler, còn với `@MainActor` thì phải chuyển qua chuyển lại giữa pool và main thread. Một cú nhảy lẻ thì rẻ, nhưng một vòng lặp mà vòng nào cũng vượt ranh giới thì trả cái giá đó cả ngàn lần.

```swift
// Tệ: 10 000 lần vượt ranh giới
for entry in entries {
    await ledger.append(entry)
}

// Tốt: một lần vượt, một transaction
await ledger.appendAll(entries)
```

Cái bản năng đó cũng đúng với `@MainActor`: gộp các cập nhật UI vào một lượt ghé, thay vì nhảy về main thread cho từng cái label một. Và nhân tiện vừa nhắc tới main actor, hai hệ quả kế tiếp nói riêng về nó. Nên đọc theo đúng thứ tự.

### Hệ quả 10. Một chunk đồng bộ dài trên main actor vẫn đóng băng UI.

Suy luận: một thread mỗi lúc chỉ làm một việc (sự thật đầu tiên của Ý tưởng 1), mà cửa của main actor dẫn tới đúng một thread, và cái thread đó còn ôm thêm việc thứ hai: chạy event loop của app, vẽ frame, phản ứng với cú chạm. Trong lúc một chunk main-actor đang chạy, main thread **không làm được gì khác**, kể cả vẽ. Actor chẳng thay đổi điều đó. Chunk của bạn ngốn hai giây CPU thì màn hình đơ hai giây, y hệt cái thời chưa có Swift Concurrency.

```swift
@MainActor
func reload() async {
    let raw = await service.load()   // suspension: main thread rảnh,
                                     // UI vẽ và phản hồi bình thường

    let report = buildReport(raw)    // chunk đồng bộ CHẠY TRÊN main thread:
                                     // 2 giây dựng báo cáo = 2 giây UI đóng băng

    show(report)
}
```

Thứ *không* làm đơ gì hết chính là **chờ**. Tại một `await`, job main-actor suspend và main thread được trả về cho event loop của nó: frame cứ vẽ, nút cứ phản hồi. Còn việc chạy trên các thread pool thì không block main thread, vì đó là những người thợ khác nhau. Nói cho thật chuẩn: chúng *có* tranh nhau cùng đám core vật lý (sơ đồ trước cho thấy mọi thread đều xài chung core), nhưng OS scheduler ưu tiên main thread cao hơn, nên việc trên pool trên thực tế không làm khựng UI.

Mô hình cũng chỉ luôn cho bạn cách sửa: một cú đơ là một chunk nặng CPU sống nhầm domain. Đá nó ra ngoài, bằng `@concurrent` (xem Hệ quả 11) hoặc một actor riêng, rồi chỉ quay về main actor cho đúng bước cuối rẻ tiền — cập nhật UI.

### Hệ quả 11. Một method async chạy ở đâu? Khai báo của nó quyết định, và Swift 6.2 đã đổi câu trả lời mặc định.

Vì sao mô hình biến chuyện này thành một hệ quả: Ý tưởng 2 nói mỗi dòng code thuộc đúng một domain. Nên câu hỏi "method này chạy ở đâu" thực chất là câu hỏi "method này thuộc domain nào", mà cái đó được chốt một lần duy nhất, ngay tại **khai báo** của method. **Chỗ gọi chẳng bao giờ có quyền quyết định.** Cụ thể, `await` ở chỗ gọi chỉ đánh dấu một suspension khả dĩ thôi, nó không gửi gì đi đâu cả. Đây là bốn kiểu khai báo khả dĩ, và cũng là chỗ Swift 6.2 bước vào, vì nó đổi luôn ý nghĩa của cái thứ ba.

```swift
actor DocumentStore {
    func fetch(_ id: String) -> Article? { ... }
    // Isolated tới DocumentStore. Chạy trong domain của store,
    // các chunk của nó đi ra cooperative pool.
}

@MainActor
func updateBadge() async { ... }
// Isolated tới main actor. Chạy trên main thread.

func loadSettings() async { ... }
// Không ghi domain. ĐÂY là thứ Swift 6.2 đã đổi:
//   trước 6.2: một hàm như vầy luôn nhảy ra pool
//   từ 6.2:    nó chạy trong domain của kẻ GỌI nó
// Hành vi mới có cách viết tường minh, và bạn có thể yêu cầu nó
// cho từng hàm bất kể cấu hình project:
//   nonisolated(nonsending) func loadSettings() async { ... }
// Cấu hình project 6.2 đơn giản biến cái này thành mặc định
// cho mọi hàm async không nói khác đi.

@concurrent
func renderThumbnail() async -> UIImage { ... }
// Pool, luôn luôn, theo yêu cầu tường minh. Bất kể ai gọi.
```

Giờ thử một chỗ gọi, gọi cả bốn từ main actor.

```swift
@MainActor
func onOpen() async {
    await documentStore.fetch("id")  // chạy trong domain của store (thread pool)
    await updateBadge()              // chạy trên main thread, không vượt ranh giới
    await loadSettings()             // default 6.2: main actor. Trước 6.2: pool
    await renderThumbnail()          // pool, tường minh
}
```

Bốn cái `await` nhìn y hệt nhau, mà bốn đích đến khác nhau, và mọi câu trả lời đều đã nằm sẵn ở khai báo. Còn một default 6.2 nữa cùng tinh thần: một app module có thể biến `@MainActor` thành domain mặc định cho mọi kiểu và hàm chưa đánh dấu (project Xcode mới bật sẵn cái này), nên code app sinh ra là đã thuộc về main actor, và chỉ bước ra ngoài ở đúng chỗ nó nói rõ.

Chỗ này dẫn tới câu hỏi mà lúc này bạn nên tự đặt ra: nếu `loadSettings` trước kia chạy trên pool mà giờ chạy trên main actor, thì cùng một hàm chưa từng làm đơ UI bao giờ, giờ lại có thể làm đơ, chỉ vì cấu hình project? Đúng, có thể, và đây là phần tính toán chính xác khi nào thì xảy ra. **Chờ không bao giờ chiếm thread**, dưới mọi cấu hình: trong lúc `await`, hàm là một continuation trong heap, chuyện đó chẳng tốn gì của main thread. Cái mà cấu hình dịch qua dịch lại giữa các domain chỉ là mấy **chunk đồng bộ** của hàm thôi. Với một hàm networking điển hình, đó là vài microgiây code bao quanh vài giây chờ, nên chẳng có gì đổi mà bạn thấy được. Một cú đơ chỉ ló ra trong đúng một kịch bản: hàm có việc CPU nặng thật sự, và nó đã âm thầm dựa hơi vào cú nhảy ra pool ngày xưa.

Và đây là **tính năng, không phải bug**. Cách sắp xếp cũ làm code kiểu đó trông an toàn một cách tình cờ: việc nặng rời khỏi main thread nhờ một luật vô hình mà nửa team còn chẳng biết là có, và trong source thì chẳng dòng nào nói ra. Cách sắp xếp mới làm code *có nghĩa đúng như những gì nó viết*. Việc CPU nặng thì phải được **gọi tên**, bằng `@concurrent` hoặc một actor riêng, và một khi đã đặt tên, chỗ nó chạy được ghi rõ trong khai báo, không phụ thuộc default nào hết. Default giờ chỉ định đoạt số phận của đám code chưa đánh dấu, mà code chưa đánh dấu thì nên nhẹ. Bạn mất đi một tấm lưới an toàn kiểu ăn may, đổi lại được một codebase mà "cái này chạy ở đâu" trả lời được bằng cách đọc khai báo, chứ không phải bằng cách học thuộc mấy "văn hoá dân gian" của từng phiên bản compiler.

**Chốt lại.** Data race chết khi dữ liệu có người chủ và compiler đứng ra soi quyền sở hữu. Một actor là một domain với một cái cửa, mỗi lúc một job bên trong, và không có thread riêng. Cửa mở tại mỗi `await`, cho bạn cả *đảm bảo transaction* (ở giữa các `await`) lẫn *cái bẫy reentrancy* (khi vắt qua chúng). Nhưng có một câu hỏi tới giờ đã quá hạn rồi. Job liên tục chuyền giá trị qua cửa: đối số đi vào, kết quả đi ra. Nếu mình trao một object thay đổi được vào một actor rồi vẫn giữ một tham chiếu tới nó ở ngoài, thì cả hai phía giờ đều đụng được nó, và cái cửa coi như chẳng bảo vệ được gì. Vậy ai kiểm tra xem cái gì được phép đi qua? Kiểm tra đó có tồn tại, nó mang một cái tên mà bạn đã gặp trong lỗi compiler không biết bao nhiêu lần, và nó chính là ý tưởng tiếp theo: **Sendable**.

## Ý tưởng 3: Sendable là một bằng chứng, không phải một hành vi.

Ý tưởng 2 kết lại bằng một lỗ hổng ngay ở cửa. Nên mình mở màn Ý tưởng 3 bằng cách... rơi thẳng vào cái lỗ đó luôn:

```swift
final class Photo {
    var caption = ""
}

actor Gallery {
    var queued: [Photo] = []
    func upload(_ photo: Photo) {
        queued.append(photo)        // actor giờ giữ một tham chiếu
    }
}

let photo = Photo()
await gallery.upload(photo)          // photo giờ ở bên trong domain...
photo.caption = "sửa từ ngoài"       // ...và vẫn ở bên ngoài. Cùng một object,
                                     // hai domain, state thay đổi được. Data race
                                     // quay lại, và actor không hề hay biết.
```

Cửa chỉ soi *ai* được vào. Nó không thèm soi *người ta cầm theo cái gì*. Mình đưa một tham chiếu tới một object mutable qua cửa, rồi giữ lại một bản sao của cái tham chiếu đó ở ngoài, thế là hai domain cùng sờ được `photo.caption` một lúc. Cả bộ máy hoành tráng của Ý tưởng 2 vẫn còn nguyên đó mà thành vô dụng. Nên phải có thêm một chốt kiểm tra thứ hai, lần này soi **hành lý**: giá trị nào thì an toàn để mang qua ranh giới domain?

**Tiêu chí là gì?** Một giá trị an toàn để đưa qua ranh giới nếu việc đưa nó qua **không thể đẻ ra state mutable mà cả hai domain cùng nhìn thấy một lúc**. Chỉ có vậy thôi, hết. `Sendable` là cái tên Swift đặt cho tính chất này, và đây là chỗ nghe lạ tai cho tới khi bạn thông: **conform `Sendable` không phải là đi cài đặt cái gì cả**. Protocol này rỗng tuếch, không method, không sinh ra dòng code nào, runtime cũng chẳng đổi gì. Nó chỉ tồn tại như một *lời tuyên bố* — "giá trị của kiểu này an toàn để gửi qua ranh giới" — còn việc của compiler là **soi bằng chứng** cho lời tuyên bố đó. Nên `Sendable` nó khác hẳn mọi protocol bạn từng gặp: bạn không viết implementation cho nó, bạn phải làm nó 'qua được' bước **verify** của compiler.

Có ba cách tử tế để chứng minh cái lời tuyên bố đó.

**Bằng chứng thứ nhất: bằng copy.** Struct với enum là value type, mà đã vượt ranh giới thì bên nhận nhận được một *bản sao*. Sửa trên bản sao thì chẳng lan đi đâu, nên state mutable dùng chung không tài nào xuất hiện — miễn là mọi stored property của nó cũng `Sendable`, cái này compiler soi đệ quy xuống tận đáy. Kiểu hay bị nghi ngờ: struct có property `var`, có method `mutating`, thế chẳng phải state mutable còn gì? Đúng, nhưng đó là state mutable **mà mình bạn sở hữu**. Method `mutating` sửa bản sao *của bạn*, còn bản sao mà domain kia cầm vẫn nguyên xi. Tính mutable của một struct nằm trong cái *biến* đang giữ nó, chứ không bao giờ nằm ở thứ gì dùng chung. Nên hỏi thẳng "value type có Sendable mặc định không" thì câu trả lời là **có**, với đúng hai điều kiện: nó là value type, và mọi thứ nhét bên trong cũng `Sendable`. Compiler còn tự áp cái bằng chứng này giúp bạn nữa, nhưng để ý cái ranh giới: **chỉ với kiểu non-public thôi**. Struct với enum non-public mà thoả điều kiện thì tự động conform ngầm, khỏi cần gõ một chữ. Còn kiểu public thì *không bao giờ* được auto conform, vì `Sendable` gắn lên một kiểu public là một *lời hứa trong API của bạn* ("kiểu này còn an toàn để gửi qua ranh giới ở mọi phiên bản về sau"), mà lời hứa với người lạ thì là của bạn, không phải của compiler. Nhớ cái vụ chia đôi public/non-public này nhé, cuối ý tưởng nó quay lại đấy.

**Bằng chứng thứ hai: bằng tính bất biến.** Một class mà mọi stored property đều là `let` thuộc kiểu `Sendable`, và bản thân class là `final` để không subclass nào lẻn vào thêm bất ngờ. Ở đây chia sẻ một tham chiếu vẫn ổn, hai domain nhìn chung một object thật, nhưng chẳng bên nào sửa được gì. **Dữ liệu đóng băng thì lấy đâu ra race.**

**Bằng chứng thứ ba: bằng cái cửa.** Actor thì tự động là `Sendable`, và giờ mình nói được chính xác vì sao: chia sẻ một tham chiếu tới actor là chia sẻ *quyền xếp hàng ở cửa*, chứ không phải quyền vào thẳng state. Domain nào cầm tham chiếu đi nữa thì mọi lần chạm state vẫn phải qua cửa, một job một lúc. Cái tham chiếu này an toàn để truyền đi chính bởi vì nó **không** cho ai truy cập thẳng vào bất cứ thứ gì.

Thế còn cái `Photo` của mình, một class có mỗi một `var`? **Không một bằng chứng nào trong ba cái áp được.** Copy thì không (nó là reference type), bất biến thì trượt (có `var` chình ình đó), cửa cũng chẳng có. Compiler không hề chê class bạn viết dở đâu. Nó nói một điều hẹp hơn và thật lòng hơn nhiều: *chẳng có bằng chứng nào cho thấy kiểu này an-toàn-qua-ranh-giới, nên tôi không cho nó qua*. Mọi cái lỗi `Sendable` bạn từng gặp đều đúng là câu này đấy.

> *Sơ đồ: Sendable chính là trạm hải quan đứng ở ranh giới domain — bản sao, class đóng băng, và tham chiếu có cửa canh thì cho qua; còn state mutable không ai bảo vệ thì chặn lại.*

### Hệ quả 12. Vì sao struct lọt qua còn class thì không.

```swift
struct Thumbnail: Sendable {        // bằng chứng bằng copy, kiểm tra từng thành viên
    let url: String
    let width: Int
}

final class Photo {                 // một var, không cửa, reference type: không bằng chứng
    var caption = ""
}

@MainActor
func share(_ t: Thumbnail, _ p: Photo) async {
    await gallery.accept(t)         // compile được: actor nhận một bản sao
    await gallery.edit(p)           // lỗi: gửi đi có nguy cơ gây data race
}
```

Cái dòng báo lỗi bạn gặp hằng ngày, giờ đọc ra đúng bản chất của nó. Compiler không cằn nhằn về style đâu, nó đang báo một cái **bằng chứng bị trượt**: giá trị này sắp vượt một ranh giới, mà không lập luận an toàn nào trong ba cái đứng vững. Cách sửa suy ra thẳng từ chính ba bằng chứng, chọn một cái: biến nó thành value type (bằng chứng bằng copy), đóng băng nó thành mấy hằng `let` trong một `final class` (bằng chứng bằng bất biến), hoặc nâng nó lên thành actor (bằng chứng bằng cửa).

### Hệ quả 13. Closure cũng vượt ranh giới, và @Sendable là cùng một kiểm tra cho chúng.

Một closure thì lưu được, truyền đi được như bất kỳ mẩu dữ liệu nào, và nó cũng vượt ranh giới domain được luôn, ví dụ khi bạn đưa một completion handler cho đám code sẽ chạy nó ở chỗ khác. Nhưng ngó kỹ xem closure thực ra là gì: một **reference type**. Nó vác theo một cái hộp trong heap chứa mấy biến đã capture, và truyền closure đi là truyền một tham chiếu tới đúng cái hộp dùng chung đó, chứ không phải một bản sao. Đó đúng là thứ khiến closure thành "hàng nguy hiểm": một `var` đã capture nằm trong hộp chính là state mutable, mà domain nào giữ closure cũng nhìn thấy chung cái hộp đó. Nên cùng cái trạm hải quan ấy áp cho closure luôn, viết là `@Sendable`: mọi thứ capture vào phải `Sendable`, và biến đã capture thì không được là mutable:

```swift
let photo = Photo()                             // class từ Hệ quả 12
let thumb = Thumbnail(url: "u", width: 64)      // struct từ Hệ quả 12
var count = 0

let job: @Sendable () -> Void = {
    count += 1          // lỗi: var đã capture là state thay đổi được dùng chung
    print(photo)        // lỗi: capture một class không Sendable
    print(thumb)        // ổn: một struct Sendable được capture dưới dạng bản sao
}
```

Đây cũng là lý do mấy API chạy closure của bạn "ở chỗ khác", như `Task.detached`, bắt closure phải `@Sendable`: closure sắp đi qua một domain khác, nên hành lý của nó bị soi ở đúng cái ranh giới như mọi thứ khác thôi.

### Hệ quả 14. Đôi khi một giá trị không-Sendable vẫn vượt qua, và đó không phải lỗ hổng: "sending" và region analysis.

Ba bằng chứng ở trên đều lập luận rằng *chia sẻ* là an toàn. Còn một lập luận thứ tư thuộc kiểu khác hẳn: chứng minh **chẳng có chia sẻ nào hết**. Nếu compiler nhìn ra được rằng phía gửi không còn giữ một tham chiếu nào tới object, thì object đâu có bị chia sẻ qua ranh giới — nó đang *dời* qua, trọn gói. Trước một chủ, sau một chủ, không bao giờ hai. Flow analysis của Swift (tên tính năng là *region-based isolation*) theo dõi đúng chuyện này, và từ khoá `sending` gắn lên một tham số là ký cái giao kèo "tôi lấy hẳn giá trị này khỏi tay anh":

```swift
actor Gallery {
    func store(_ photo: sending Photo) { ... }     // lấy quyền sở hữu
}

let photo = Photo()             // Photo vẫn không phải Sendable
photo.caption = "hello"
await gallery.store(photo)      // compile được: không tham chiếu nào ở lại

// print(photo.caption)         // thêm dòng này và bằng chứng sụp đổ:
                                // lỗi, 'photo' được dùng sau khi đã bị gửi đi
```

Có một chuyện không tự dưng xảy ra: phía nhận phải khai báo là nó nhận luôn quyền sở hữu, bằng `sending` trên tham số (giá trị trả về cũng đánh dấu kiểu vậy được). Còn cái *tự* xảy ra mà bạn khỏi viết gì là: cả đống điểm vào của thư viện chuẩn đã khai báo sẵn rồi — tham số closure của `Task.init`, `addTask` trong task group, `resume(returning:)` trên continuation. Nên mỗi khi một giá trị không-Sendable vượt ranh giới "một cách tự động", lý do luôn y hệt nhau: bạn đang gọi một API mà tác giả của nó đã gõ `sending` giúp bạn rồi, và compiler chỉ cần verify phần việc của bạn trong giao kèo — là không còn tham chiếu nào ở lại.

Đây chính là lời giải cho cái thắc mắc nhiều người vướng: "sao cùng một giá trị không-Sendable, qua ranh giới chỗ này thì im ru, mà chỗ kia lại lỗi?" Chỗ kia, còn một cái tham chiếu nào đó sống sờ sờ ở phía bạn. Chỗ này, compiler chứng minh được bạn chả giữ gì, nên chẳng có gì bị chia sẻ, và `Sendable` chưa bao giờ cần tới đây cả.

### Hệ quả 15. @unchecked Sendable là bạn tự ký vào bằng chứng.

Vì sao lại cần một cái "lối thoát hiểm" thì suy ra thẳng từ bản chất của `Sendable`. Mấy bằng chứng mà compiler soi được đều mang tính **cấu trúc**: nó đọc được kiểu, đọc `let` với `var`, value với reference. Nhưng an toàn đôi khi lại nằm ở **kỷ luật**, thứ mà cấu trúc chẳng lộ ra ngoài, và ca kinh điển là một class tự bảo vệ state của nó bằng một cái lock nội bộ:

```swift
final class ViewCounter: @unchecked Sendable {
    private let lock = NSLock()
    private var hits: [String: Int] = [:]

    func bump(_ photoID: String) {
        lock.lock()
        defer { lock.unlock() }
        hits[photoID, default: 0] += 1     // mọi truy cập đều đi qua lock
    }
}
```

Lời tuyên bố ở đây là đúng — cái lock làm nó an toàn thật — nhưng không một phép kiểm tra cấu trúc nào verify nổi, vì an toàn nằm ở kỷ luật chứ đâu nằm ở cấu trúc. `@unchecked` chính là lối thoát hiểm: "tôi lấy thẩm quyền của chính mình mà khẳng định lời tuyên bố `Sendable`, thôi ngừng kiểm tra đi". Nhưng phải hiểu rõ cái chữ ký này tốn cái gì. Compiler **ngừng kiểm tra kiểu này mãi mãi**: không chỉ code hôm nay, mà cả cái `var` trông vô hại mà một đồng đội thêm vào năm sau, lúc nó chẳng để ý gì tới quy ước lock. Có những chỗ dùng chính đáng thật (class được lock bảo vệ, wrapper bọc thư viện C thread-safe), nhưng mỗi lần dùng là bạn hạ một *đảm bảo của compiler* xuống thành một *đảm bảo của code-review*. Một điều đáng biết: riêng ca lock này, kiểu `Mutex` hiện đại trong framework `Synchronization` giữ luôn giá trị được bảo vệ *bên trong chính nó* và là `Sendable` một cách đàng hoàng, kiểm tra được — khỏi cần đụng tới `@unchecked` trong code mới nữa.

### Hệ quả 16. ~Sendable: một lời "cái này KHÔNG được vượt ranh giới" tường minh (Swift 6.4).

Cái này suy ra từ chuyện conformance vốn là một *lời tuyên bố*. Một lời tuyên bố thì hoặc đưa ra, hoặc không, nhưng *không đưa ra* một lời tuyên bố thì chẳng nói lên điều gì cả. Khi một kiểu đơn giản là không conform `Sendable`, bạn chịu, chẳng biết vì sao. Có thể tác giả quên. Có thể họ chưa làm tới. Có thể họ đã cân nhắc kỹ rồi và quyết là nó *không bao giờ được* vượt ranh giới. Ba khả năng đó nhìn từ ngoài giống hệt nhau, mà nếu kiểu đó tới từ thư viện của người khác thì bạn còn chẳng mở được nội bộ của nó ra mà đoán. Swift 6.4 (lúc viết bài đang beta cùng Xcode 27) thêm một cách nói thẳng cái quyết định đó ra:

```swift
public struct ExportSettings: ~Sendable {
    var destination: String   // về cấu trúc thì cái này có thể Sendable,
}                             // nhưng tác giả nói: đừng dựa vào điều đó
```

`~Sendable` nghĩa là: kiểu này đã được cân nhắc rồi, và nó *không được phép* conform. Về mặt cơ chế nó làm gì thì tuỳ kiểu đó sống ở đâu, và đây đúng là chỗ cái vụ chia đôi public/non-public từ Bằng chứng thứ nhất quay lại. Với một value type **non-public**, `~Sendable` tắt cái conformance tự động (cái mà compiler cấp cho khi mọi thành viên đều `Sendable`): không có marker thì kiểu này là `Sendable`, gắn marker vào thì không. Còn với một kiểu **public**, như ở đây, cái conformance tự động đó *chưa từng tồn tại*, nên chẳng có gì để mà tắt và marker cũng không đổi gì chuyện compile được. Cái nó làm thật sự — và đây mới là việc chính trong cả hai trường hợp: **nó ghi lại quyết định của tác giả**, để người đọc API thấy "đã cân nhắc, cố tình không Sendable" thay vì một sự im lặng muốn hiểu sao cũng được, và nó giữ cho tác giả rảnh tay thêm nội bộ không-Sendable ở phiên bản sau, vì đã có client nào được phép dựa vào chuyện kiểu này vượt ranh giới đâu. Có thêm mảnh này, hệ thống ôm trọn cả ba câu trả lời khả dĩ: chứng minh lời tuyên bố (`Sendable`), tự ký tên chịu trách nhiệm (`@unchecked Sendable`), hoặc từ chối thẳng thừng (`~Sendable`). Chẳng ai còn phải ngồi đoán sự im lặng nghĩa là gì nữa.

**Chốt lại.** Ba ý tưởng xong. Hàm bị cắt thành các chunk mà thread pool nào rảnh cũng chạy được. Dữ liệu sống trong các domain, và mỗi domain một lúc chỉ có một job làm việc bên trong. Giá trị muốn vượt giữa các domain thì phải có bằng chứng rằng việc vượt là an toàn, và compiler chính là kẻ soi bằng chứng. Còn đúng một câu hỏi nữa, mà nó núp ngay trước mắt từ cái `Task { }` đầu tiên xuất hiện trên mấy trang này: **task chính xác là cái gì, ai sở hữu nó, chuyện gì xảy ra khi chẳng còn ai cần kết quả của nó nữa, và vì sao chữ "structured" cứ lặp đi lặp lại hoài?** Đó là ý tưởng cuối cùng: **cái cây task**.

## Ý tưởng 4: Task sống trong hai thế giới: trong cây và ngoài cây.

Chữ `Task` xuất hiện suốt cả bài rồi, mà lần nào mình cũng chỉ giải thích nửa vời. Giờ trả nợ: **task rốt cuộc là cái gì?** Nó là *đơn vị sở hữu một câu chuyện async đang chạy*. Nói cụ thể, nó là một object trong heap ôm hết mọi thứ mà câu chuyện đó cần: continuation hiện tại (state đã lưu cộng cái bookmark từ Ý tưởng 1), một mức priority, một cờ hủy (cancellation flag), và vài giá trị đính kèm. **Mọi dòng code async trong app của bạn đều chạy như một phần của đúng một task.**

Và đây là cả cái ý tưởng gói gọn trong mấy câu. `async let` và task group đẻ ra task **gia nhập một cái cây**. `Task { }` đẻ ra task **không gia nhập**. Cái cây đó chính là cái mà chữ "structured" trong *structured concurrency* muốn nói. Trong cây, vòng đời, lỗi và cancellation được lo giúp bạn hết. Ngoài cây, cả ba thứ đó bạn tự lo.

Hai thế giới này ở đâu ra? `async let` và `group.addTask` đẻ ra **con** (children): những task gắn với cái scope đã sinh ra chúng. `Task { }` thì đẻ ra một **gốc** (root): một task chẳng gắn với ai. Cần cái loại "gắn" đó làm gì? Thiếu nó — mà đó là trạng thái bình thường của thế giới GCD — việc làm cứ sống lâu hơn cái hàm bấm nút chạy nó, và ba câu hỏi mất luôn lời đáp: khi nào thì việc xong, lỗi của nó chạy đi đâu, và "hủy nó" thì hủy cái gì mới đúng. Cái cây trả lại cả ba, bằng cách bắt việc song song **lồng vào nhau y như cách các lời gọi hàm lồng vào nhau**. Mỗi hệ quả bên dưới sẽ diễn lại cùng một tình huống trong cả hai thế giới.

### Hệ quả 17. Ai chờ ai.

```swift
func openPage() async {
    Task {
        await syncHistory()         // vẫn đang chạy...
    }
    print("openPage returned")      // ...khi dòng này in ra
}

func buildPage() async throws -> ProductPage {
    async let details = api.details()   // bắt đầu, chạy song song
    async let reviews = api.reviews()   // bắt đầu, chạy song song
    return try await ProductPage(details: details, reviews: reviews)
}   // dòng này không đạt tới cho tới khi CẢ HAI request hoàn tất
```

`Task { }` đẻ ra một task **độc lập**. Thằng tạo ra nó chẳng thèm chờ: `openPage` return luôn, còn `syncHistory` thì cứ thế chạy tiếp một mình, y hệt cách `DispatchQueue.async` cư xử bên GCD. `async let` thì đẻ ra một **con**, và con phải theo cái luật duy nhất của cây, gọi là **luật con**: *con không thể sống lâu hơn cái scope đã sinh ra nó*. Nên `buildPage` sẽ không return — không return bình thường, cũng không throw ra — chừng nào cả hai con chưa xong. Luật này chỉ áp cho con thôi, nên chẳng ai chờ `syncHistory` cả, cái `Task { }` kia có nằm lồng sâu tới đâu trong bất cứ thứ gì cũng vậy.

Nói rõ luôn chữ *scope*, vì cả mục này treo hết vào nó: scope là cái khối code nơi bạn khai báo `async let`, tức vùng nằm giữa `{` và `}` của nó. Hay gặp nhất là thân hàm, nhưng không nhất thiết: thân một `if`, một `do`, hay một vòng lặp cũng là một scope riêng. Thoát khỏi khối bằng đường nào cũng vậy (thoát bình thường, `return`, `throw`, `break`), nghĩa là mọi con khai báo trong đó đều đã xong tính tới lúc ấy. Còn với task group, scope là cái closure bạn truyền cho `withTaskGroup`:

```swift
func load() async {
    if showReviews {
        async let reviews = api.reviews()
        ...
    }   // ← scope kết thúc TẠI ĐÂY, không phải cuối hàm:
        //   tại dấu ngoặc này con đã bị hủy và đã được chờ xong
    doOtherWork()
}
```

> *Sơ đồ: Con sống bên trong scope của cha. `Task { }` tuy được tạo ra từ trong scope nhưng KHÔNG thuộc cái cây — nó tự dựng cây riêng của mình.*

Còn một chuyện nữa về bản trong-cây, vì cái biến `details` ở đó không phải như bạn tưởng đâu:

```swift
async let details = api.details()   // 'details' không phải một giá trị: nó là một handle
                                    // tới một con vẫn đang chạy
let loaded = try await details      // đọc nó thì ổn, và 'loaded' là một giá trị
                                    // bình thường bạn lưu đâu cũng được
self.cached = details               // bị compiler cấm: lưu cái handle chưa-await
                                    // sẽ cho phép ai đó await con này SAU KHI
                                    // scope đã chết
```

Đó chính là toàn bộ ý nghĩa của cái ràng buộc nổi tiếng "async let không lưu, không return được": *kết quả* của con thì đi đâu cũng được, nhưng *cái handle trỏ tới con đang sống* thì không được rời scope của nó, đơn giản vì con không được phép sống lâu hơn scope.

### Hệ quả 18. Một lỗi trong việc song song.

Giả sử `api.details()` throw lỗi sau 1 giây, còn `api.reviews()` không bao giờ throw, chỉ chạy mất 5 giây là xong. Mình bấm chạy cả hai song song, trong từng thế giới một.

Trong cây:

```swift
func buildPage() async throws -> ProductPage {
    async let details = api.details()   // không try tại khai báo:
    async let reviews = api.reviews()   // lỗi nổi lên ở nơi giá trị được đọc
    return try await ProductPage(details: details, reviews: reviews)
}
```

Tới mốc 1 giây thì `details` throw. Luật con cấm `buildPage` return khi `reviews` còn đang chạy, nên runtime **hủy** `reviews` rồi chờ nó dừng hẳn. Chờ lâu hay mau là tùy chính `reviews`: một lời gọi mạng `URLSession` phản ứng với cancel gần như tức thì, nên ở đây lỗi về tới người gọi sau cỡ 1 giây. Còn một con mà phớt lờ cancel (xem Hệ quả 19) thì bị chờ đủ cả 5 giây. Nhưng dù kiểu gì thì vẫn đảm bảo một điều: tới lúc lỗi rời khỏi `buildPage`, **chẳng còn gì đang chạy nữa**.

Ngoài cây:

```swift
func fireBoth() {
    Task { let details = try await api.details() }
    Task { let reviews = try await api.reviews() }
}
```

Tới mốc 1 giây task đầu throw, và lỗi thì **bay hơi** luôn: chẳng ai await cái task này, nên lỗi không có chỗ nào để đi. Task thứ hai chẳng hay biết gì, cứ chạy nốt 5 giây của nó. Không dọn dẹp, không giao lỗi, hai thằng chẳng liên quan gì tới nhau.

Khi bạn chưa biết trước sẽ có bao nhiêu con, vẫn cái giao kèo đó nhưng gói lại thành một object, **task group**, luật y hệt:

```swift
let prices = try await withThrowingTaskGroup(of: Price.self) { group in
    for id in itemIDs {
        group.addTask { try await fetchPrice(id) }   // một con cho mỗi item
    }
    var result: [Price] = []
    for try await price in group { result.append(price) }
    // nếu một lượt fetch bất kỳ throw, lỗi được re-throw ngay tại đây,
    // group hủy các fetch còn lại, chờ chúng xong,
    // và cả hàm thoát ra cùng lỗi đó
    return result
}   // dù thế nào: không fetch nào còn chạy qua dòng này
```

### Hệ quả 19. Hủy (Cancellation).

Cái sự thật làm ai cũng ngã ngửa đầu tiên: **chẳng có gì tự dừng cả, ở cả hai thế giới**. `.cancel()` làm đúng một việc thôi: **giương một lá cờ** lên task. Khác nhau giữa hai thế giới chỉ là lá cờ đó đi được xa tới đâu.

Trong cây:

```swift
let pageTask = Task { try await buildPage() }
pageTask.cancel()       // cờ đi xuống theo các liên kết cha-con:
                        // buildPage, rồi details, rồi reviews
```

Ngoài cây:

```swift
let parent = Task {
    Task { await syncHistory() }      // một gốc mới: không có liên kết cha
}
parent.cancel()                       // cờ chỉ được giương trên parent.
                                      // syncHistory không bao giờ biết:
                                      // không có liên kết nào cho cờ đi qua
```

(Một task độc lập vẫn hủy được, nhưng phải *trực tiếp*: giữ lấy handle của nó rồi tự tay gọi `cancel()`. Đúng cái chuyện mà Hệ quả 20 sắp nói tới đấy.)

Giờ tới phần chung cho cả hai thế giới: khi lá cờ tới được một task thì sao. Tự thân nó thì **chẳng làm gì cả**, và Ý tưởng 1 đã giải thích vì sao không thể mạnh hơn được. Một task đang suspended là một continuation nằm trong heap: chẳng có code nào của nó đang chạy, thì lấy gì mà dừng. Còn một chunk đang chạy là code sống trên một thread pool: giết nó giữa hai lệnh thì để lại dữ liệu ghi dở dang với cái lock đang cầm. Nên lá cờ cứ nằm đó, đã giương lên, và task thì **tự dừng mình tại các checkpoint do chính nó chọn**:

```swift
func compress(_ photos: [Photo]) async throws -> [Data] {
    var results: [Data] = []
    for photo in photos {
        try Task.checkCancellation()      // điểm dừng an toàn: throw nếu cờ đã giương
        results.append(shrink(photo))     // giữa các checkpoint, việc cứ tiếp tục
    }
    return results
}
```

Cái này gọi là **cooperative cancellation** (hủy có hợp tác), và cái tên nói toạc luôn vì sao nó chạy như vậy: muốn dừng thì cần cả hai phía hợp tác. Một phía giương cờ (thằng gọi `cancel()`), phía kia là code đang bị dừng, chịu khó kiểm tra cờ tại các checkpoint của chính nó. Thiếu nửa sau, chẳng có gì xảy ra hết. Và nó áp y chang trong cả hai thế giới: cờ tới bằng đường nào cũng vậy, đi xuống theo cây hay bằng một `cancel()` trực tiếp lên handle, thì cũng chỉ code của chính task đó mới phản ứng. Mấy API "chờ" thì kiểm tra giúp bạn: `Task.sleep` bên thư viện chuẩn throw `CancellationError` khi cờ giương lên, còn `URLSession` (nằm trong Foundation) làm request fail với một `URLError` mã `.cancelled`. Nên code mà chủ yếu là `await` thì tự động ngoan, còn mấy vòng lặp đồng bộ dài của riêng bạn thì bạn tự lo. Khi cancel "không ăn thua" trong một codebase, chẩn đoán gần như lúc nào cũng là: có người giương cờ rồi, mà chẳng ai kiểm tra nó cả.

Còn với mấy ca mà kiểm tra tại checkpoint là chưa đủ, kiểu có thứ phải phản ứng *ngay cái khoảnh khắc* cờ giương (đóng một kết nối, resume một continuation đã lưu), thì có `withTaskCancellationHandler`: nó chạy handler của bạn ngay lúc task bị hủy, chẳng chờ code chạm tới checkpoint nào cả.

### Hệ quả 20. Vòng đời của Task { }.

`Task { }` **thừa hưởng** ba thứ từ chỗ nó được tạo ra: isolation domain, mức priority, và các task-local value. Còn `Task.detached` thì **chẳng thừa hưởng gì hết**:

```swift
@MainActor
final class CartModel {
    var lines: [Line] = []

    func reload() {
        Task {
            lines = await api.cart()        // thừa hưởng domain của main actor:
                                            // dòng này chạy trên main actor, an toàn
        }

        Task.detached {
            await self.trimCache()          // không thừa hưởng gì: pool, ưu tiên mặc định
                                            // và 'self' giờ phải vượt một ranh giới,
                                            // nên trạm hải quan Sendable từ Ý tưởng 3 áp dụng
        }
    }
}
```

Cái mà cả hai thằng đều không có là **một chỗ trong cây**, nên chẳng bao giờ có ai tự động hủy chúng. Đó là chủ đích của công cụ, không phải lỗ hổng: `Task { }` sinh ra để làm mấy việc *không được ăn theo số phận của kẻ tạo ra nó*, kiểu lưu một bản nháp mà lẽ ra phải xong kể cả khi người dùng đã rời đi. Cái giá của sự độc lập đó lộ rõ nhất trong một cảnh cụ thể. Giả sử `CartModel` ở trên điều khiển một màn hình giỏ hàng: màn hình hiện lên, `reload()` được gọi, rồi người dùng lập tức nhảy đi chỗ khác, thế là màn hình đóng và model bị deallocate theo luôn. Cái request giỏ hàng thì *chẳng quan tâm*. Nó cứ chạy tới hết rồi giao về mớ dòng giỏ hàng mà chẳng ai thèm xem, và trong mấy ca nặng hơn (upload, timer, vòng lặp vô tận) đám task bị bỏ quên kiểu này cứ chồng chất lên. Vòng đời là bạn tự lo, làm tay hoặc giao scope cho framework (modifier `.task { }` của SwiftUI tự hủy task khi view biến mất):

```swift
private var reloadTask: Task<Void, Never>?

func reload() {
    reloadTask?.cancel()                // dừng cái trước, nếu có
    reloadTask = Task { [weak self] in
        let cart = await Api.cart()
        self?.lines = cart
    }
}

deinit { reloadTask?.cancel() }
```

Quy tắc đặt nó ở đâu gói gọn trong hai dòng. `Task { }` thuộc về **cái rìa của thế giới async**, nơi code đồng bộ (một button handler, một delegate method) bấm nút chạy task đầu tiên mà chẳng có ai sẵn đó để làm cha nó. Còn bên trong code async, việc song song nên là **con**, và một cái `Task { }` mà bạn bắt gặp ở đó thì đang nợ bạn lời đáp cho một câu hỏi: **ai hủy nó?**

## Bốn ý tưởng trên một trang

**Hàm chẳng chờ gì cả**: chúng bị cắt tại mỗi `await` thành các chunk, state của chúng nằm trong continuation ở heap, và một pool cố định cỡ một thread mỗi core sẽ chạy bất cứ chunk nào đã sẵn sàng.

**Dữ liệu không thuộc về thread**: nó thuộc về các isolation domain, mỗi domain có một cái cửa cho mỗi lúc đúng một job vào, và cửa mở ra tại mỗi suspension — thứ vừa là đảm bảo transaction vừa là cái bẫy reentrancy.

**Giá trị chỉ vượt được giữa các domain khi có bằng chứng là an toàn**, và `Sendable` chính là cái bằng chứng đó, chứ không phải một hành vi: bằng copy, bằng tính bất biến, bằng cửa, hoặc bằng tính duy-nhất đã được verify.

**Task sống trong hai thế giới**: trong cây, nơi con không được sống lâu hơn scope còn lỗi, dọn dẹp với cancellation thì được lo giúp bạn; và ngoài cây, nơi cả ba thứ đó bạn tự lo.

Mỗi hệ quả trong bài, cả hai mươi cái, đều chỉ là một trong bốn câu trên đem áp vào một tình huống cụ thể. Đó đúng là điều mình muốn nói với anh em. Nếu sau bài này mà còn một hành vi nào đó của Swift Concurrency với bạn vẫn cứ trông tùy tiện, thì quăng nó xuống phần bình luận đi: hoặc là bốn ý tưởng giải thích được nó, hoặc là mình còn nợ bài viết này một ý tưởng thứ năm.

---

*Bài này mình dịch và biên soạn lại (sát nghĩa) từ bài gốc của **Lev Litvak** — "Swift Concurrency: From Ideas Under the Hood to Practical Consequences" ([LinkedIn](https://www.linkedin.com/pulse/swift-concurrency-from-ideas-under-hood-practical-lev-litvak-l1kze/)). Công lao nội dung thuộc hết về tác giả gốc; mình chỉ dịch lại cho anh em Việt mình dễ tiếp cận thôi.*
