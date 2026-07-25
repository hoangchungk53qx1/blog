---
title: "Swift Concurrency GCD tới async/await, Task, Actor (Phần 1 Cơ bản)"
description: "Từ GCD tới async/await, Task, Actor và những thứ đi kèm cơ bản"
icon: "article"
date: "2026-07-25T09:00:00+07:00"
lastmod: "2026-07-25T09:00:00+07:00"
draft: false
toc: true
weight: 999
---

# Tác giả : ChungHA

# Swift Concurrency

Ai làm iOS giờ cũng đã đụng async/await rồi: code ngắn hơn, code nhìn flatten hơn, gắn @MainActor chỗ này chỗ kia, viết thử vài cái actor, và bằng cách nào đó dẹp được đống lỗi Sendable đi 

Anh em Android Dev nào đọc thấy like style suspend , async nhưng nhìn như sync

Nhưng biết xài cú pháp với hiểu hệ thống là hai chuyện khác nhau. Sao compiler không cho truyền class này giữa các task? Sao actor bảo là bảo vệ state mà state vẫn bị đổi ngay giữa method của mình? Sao chỉ một cái semaphore vô hại mà treo cả app? Chừng nào còn phải học thuộc từng case như thế, Swift Concurrency vẫn là một đống rule khó hiểu.

Mỗi app đều có một **Main Thread** lo phần UI (nút bấm, màn hình, animation). Bạn cũng có thể tạo thêm các thread khác để làm việc dưới nền (background).

### Tại sao chuyện này lại quan trọng?

- Nếu một tác vụ nặng (request mạng, xử lý ảnh, đọc database) chạy trên Main Thread, cả app sẽ đơ — nút bấm không phản hồi, cuộn màn hình cũng đứng.
- Vì thế, ta phải đẩy việc nặng xuống background, giữ cho Main Thread rảnh rang chỉ để lo cập nhật UI.

Ngày trước, dev iOS xử lý thủ công bằng:

- **GCD (Grand Central Dispatch)** — `DispatchQueue.global()` và `DispatchQueue.main`.

## GCD đã xử lý Concurrency thế nào?

Trước khi có Swift Concurrency, iOS dùng **Grand Central Dispatch (GCD)** để chạy các tác vụ bất đồng bộ. Dev đặt tác vụ lên các Dispatch Queue, và GCD tự động lo phần quản lý thread.

```swift
DispatchQueue.global().async {
    // Do on background
}

// Sau khi làm xong việc dưới background, Dev phải tự
// chuyển về Main Queue để cập nhật UI.
DispatchQueue.global().async {
    let data = fetchDataFromBackground()

    DispatchQueue.main.async {
        self.label.text = data
    }
}
```

### Những điểm dở của GCD

- Khi cần nhiều tác vụ bất đồng bộ, code lồng vào nhau tầng tầng lớp lớp và rất khó đọc.
- Dev phải tự tay chuyển qua chuyển lại giữa background queue và main queue.
- Chỉ cần quên chuyển về main queue một lần thôi là UI có thể crash hoặc đơ.
- Project càng lớn, code càng dài và càng khó hiểu vì đống closure lồng nhau.

```swift
DispatchQueue.global().async {
    api1()

    DispatchQueue.main.async {
        api2()

        DispatchQueue.global().async {
            api3()
        }
    }
}
```

**Phần sau mình sẽ nói điểm khác biệt giữa Swift Concurrency và GCD kỹ hơn về mặt thread**

## Vấn đề cốt lõi

GCD chỉ biết đúng một việc: *"chạy đoạn code này trên một thread nào đó."* Hết. Nó hoàn toàn không biết:

- Code của bạn đang đụng vào dữ liệu nào.
- Cùng dữ liệu đó có đang bị một thread khác đụng vào cùng lúc hay không.
- Hai thread có đang truy cập dữ liệu đó ở đúng cùng một thời điểm hay không.

### Data Race là gì?

Một **data race** xảy ra khi hai tác vụ background khác nhau cùng cố đọc và ghi vào **đúng một biến** trong bộ nhớ, ở **đúng cùng một thời điểm, có thể phần triệu giây**.

**Ví dụ mình hay lấy cho học viên "giao dịch kép ngân hàng":**
- Tưởng tượng hai vợ chồng có chung một tài khoản ngân hàng với 100$.
- Ở đúng cùng một mili-giây, người chồng rút 50$ tại một cây ATM ở Hà NỘI, còn người vợ tiêu 70$ tại một cửa hàng CircleK.
- Nếu hệ thống ngân hàng không có "khoá" (lock) hay "hàng đợi" (queue), cả hai giao dịch đều đọc số dư là 100$. Cả hai đều được duyệt. Kết quả là số dư âm, và database bị cook.

### Vì sao nó nguy hiểm?

 Data race dẫn tới sai về mặt số liệu, trạng thái UI bị sai (ví dụ hiển thị nhầm trạng thái chuyển khoản), và lỗi bộ nhớ (memory corruption) có thể khi thao tác với mảng.

### Ý tưởng cốt lõi

- Ngày trước, ta bảo vệ dữ liệu bằng cách dùng `DispatchQueue` một cách thủ công.
- **Vấn đề:** Compiler chẳng biết ta đang làm gì. Chỉ cần bạn quên dùng `DispatchQueue.main.async` hay quên khoá ở đúng **một** hàm background thôi, là data race xảy ra.
- Những bug kiểu này **vô hình**. Chúng vượt qua test ở máy bạn, vượt qua cả QA, rồi mới sai trên điện thoại của khách hàng — ngay giữa một giao dịch chuyển tiền quan trọng.
- **Cái lợi của Swift Concurrency:** Nó dời việc kiểm tra an toàn này từ **runtime** (trên máy người dùng) sang **compile-time**. Nếu có nguy cơ data race, app sẽ **không build được**.

### Ví dụ

```swift
class DangerousAccountBank {
    var balance: Double = 1000.0

    func withdraw(amount: Double) {
        // ❌ VẤN ĐỀ: Nếu hai thread background chạy cái này cùng lúc,
        // cả hai cùng đọc balance = 1000, cả hai cùng duyệt,
        // và bạn rút quá số dư mà không hề có lock!
        if balance >= amount {
            balance -= amount
        }
    }
}

let account = DangerousAccountBank()

// Thread A: Rút 700 từ Hà Lội
DispatchQueue.global().async {
    account.withdraw(amount: 700.0)
}

// Thread B: Rút 500 từ CircleK
DispatchQueue.global().async {
    account.withdraw(amount: 500.0)
}

// Kết quả: App build ra KHÔNG một warning nào, nhưng một
// data race âm thầm xảy ra và số dư bị sai.
```

## Vì sao lại cần Swift Concurrency hiện đại?

Tại **WWDC 2021**, Apple giới thiệu một hệ thống mới — **async/await, Task, và Actor** — với ba mục tiêu:

- Viết code đọc từ trên xuống dưới, theo một đường thẳng, dù thực chất nó chạy bất đồng bộ. 
- Để compiler tự bắt các race condition, thay vì trông chờ Dev phải cẩn thận.
- Làm cho việc xử lý lỗi đơn giản và nhất quán hơn.


### async và await — nền tảng

- **`async`**: đặt trước một hàm cần thời gian để hoàn thành (ví dụ callApi). Nó có nghĩa là "hàm này sẽ không trả kết quả ngay lập tức — nó sẽ tạm dừng rồi quay lại cùng với kết quả."
- **`await`**: dùng khi gọi một hàm như vậy. Nó có nghĩa là "dừng ở đây cho tới khi có kết quả, nhưng đừng chặn những thứ khác trong app trong lúc chờ."

```swift
func fetchUserName() async -> String {
    // Hình dung có một network call xảy ra ở đây
    return "Ali"
}

@MainActor
func showProfile() async {
    let name = await fetchUserName()
    print("Tên người dùng là: \(name)")
}
```

`await fetchUserName()` nghĩa là "chờ lấy kết quả, nhưng đừng block thread."

**Điểm quan trọng nhất:** `await` **KHÔNG** làm block Main Thread. Trong kiểu GCD cũ,Dev phải tự tay đổi thread — giờ đây compiler lo giúp bạn chuyện đó.

### Task — khởi động công việc bất đồng bộ

Một hàm `async` chỉ chạy khi có gì đó gọi nó từ một ngữ cảnh async. Để khởi động công việc async từ một chỗ bình thường (ví dụ khi bấm nút), ta dùng `Task`:

```swift
Button("Load Profile ChungHA") {
    Task {
        let name = await fetchUserName()
        self.userName = name
    }
}
```

`Task { }` tạo ra một "như 1 task async" mới, có thể chạy dưới background, và bên trong nó bạn thoải mái dùng `await`.

## Structured Concurrency — cóp tổ chức

Giả sử bạn cần hai thứ cùng một lúc: tên người dùng và ảnh đại diện. Chúng độc lập với nhau, nên chạy đồng thờig sẽ tốt hơn là chạy lần lượt cái này xong mới tới cái kia.

--> gióng async await bên Coroutine thôi

### async let — conccurency

```swift
func loadProfile() async {
    async let name = fetchUserName()
    async let image = fetchProfileImage()

    let finalName = await name
    let finalImage = await image
}
```

Ở đây, `fetchUserName()` và `fetchProfileImage()` cùng khởi động một lúc (đồng thời), và ta chỉ `await` chúng khi thực sự cần kết quả. Cách này nhanh hơn nhiều so với làm tuần tự.

### TaskGroup — khi số lượng tác vụ không cố định

Nếu bạn cần tải một số lượng  động (ví dụ danh sách 10, 20, hay nhiều ảnh hơn), Dùng **TaskGroup** — nó chạy tất cả đồng thời và thu kết quả về khi từng cái xong.

```swift
func downloadAllImages() async {
    await withTaskGroup(of: String.self) { group in
        for id in 1...5 {
            group.addTask {
                await downloadImage(id: id)
            }
        }

        for await image in group {
            print(image)
        }
    }
}
```

--> Syntax đối với mình trông chán đời.

## Actor bảo vệ shared state như thế nào?

Nếu ta có một class quản lý dùng chung (ví dụ `TransactionManager`), ta không thể để nhiều tác vụ đụng vào nó một cách tự do.

**Ví dụ** một `actor` giống như một **1 người giao dịch** ngồi sau tấm kính.

- Bạn (Thread A) không thể tự chui người qua quầy mà chộp lấy ngăn kéo tiền.
- Bạn phải xếp hàng và nhờ giao dịch viên.
- Nếu một người khác (Thread B) cũng muốn lấy tiền, họ phải đứng chờ sau bạn.

### Ví dụ

```swift
class MyBankAccount {
    var balance = 1000

    func withdraw(amount: Int) {
        if balance >= amount {
            balance -= amount
        }
    }
}

let account = MyBankAccount()

DispatchQueue.global().async {
    account.withdraw(amount: 700)
}

DispatchQueue.global().async {
    account.withdraw(amount: 700)
}

// Thread 1 đọc balance -> Balance = 1000
// Thread 2 đọc balance -> Balance = 1000
// Thread 1: 1000 - 700 = 300
// Thread 2: 1000 - 700 = 300
// Trong trường hợp này kết quả có thể ra -400, hoàn toàn sai
// -> đây chính là race condition
```

### Giải pháp bằng actor

```swift
actor MyBankAccount {
    var balance = 1000

    func withdraw(amount: Int) {
        if balance >= amount {
            balance -= amount
        }
    }
}

let account = MyBankAccount()

Task {
    await account.withdraw(amount: 700)
}

Task {
    await account.withdraw(amount: 700)
}

// Balance = 1000, rút 700 -> Balance = 300
// Task tiếp theo: 700 > 300 -> Rút thất bại
// Luồng đi: Task 1 -> Actor -> Task 2 (tuần tự, an toàn)
```

Actor đảm bảo mỗi lúc chỉ có một tác vụ được đụng vào trạng thái bên trong nó, nên data race không thể xảy ra.

## Vì sao cần `@MainActor` trên một số hàm cụ thể?

**Ý tưởng:**

- Trong iOS, mọi cập nhật UI đều **phải** xảy ra trên Main Thread. Nếu bạn cập nhật một property `@Published` của SwiftUI hay một biến `@State` từ background thread, app sẽ giật hoặc crash.
- **Ví dụ** `@MainActor` giống như (stage manager) của một nhà hát. Chỉ có anh quản lý sân khấu mới được phép di chuyển đạo cụ và chỉnh ánh đèn (tức là UI) trên sân khấu.
- Nếu một background actor (một thread nền đang lấy sao kê ngân hàng) muốn cập nhật màn hình, nó phải đưa dữ liệu cho anh quản lý sân khấu (`@MainActor`) để đưa lên màn hình.
- Trong Swift, **Main Actor** là một actor đặc biệt chạy tác vụ trên main thread — cũng chính là thread cập nhật UI của app.
- Main Actor đảm bảo mọi đoạn code làm thay đổi giao diện (UI) đều chạy an toàn trên main thread.

### Vì sao ta đánh dấu một số hàm bằng `@MainActor`

Nếu ViewModel của bạn lấy dữ liệu API trên background thread, thì cái hàm thực sự **gán** dữ liệu đó vào các property UI (ví dụ `self.accountBalance = newBalance`) phải được đánh dấu `@MainActor`. Điều này đảm bảo việc cập nhật UI tự động "nhảy" về main thread một cách an toàn.

Ví dụ iOS 16+, chứ iOS 17 không cần ``ObservableObject``

```swift
class BankViewModel: ObservableObject {
    @Published var amount: String = "Loading..."

    func fetchAmount() async {
        let result = await callAmountAPI()
        amount = result // Không có gì đảm bảo nó chạy trên main thread
    }
}
```

```swift
class BankViewModel: ObservableObject {
    @Published var amount: String = "Loading..."

    @MainActor
    func fetchWeather() async {
        let result = await callAmountAPI() // phần này chạy trên background thread
        // giờ thì chắc chắn dòng dưới chạy trên main thread
        amount = result
    }
}
```

--> Thấy giống bảo toàn context Coroutine không?


## Sendable — báo cho compiler biết một kiểu là an toàn để share

Khi dữ liệu được "gửi" từ thread này sang thread khác (ví dụ bên trong một `Task`), Swift cần chắc chắn rằng kiểu dữ liệu đó là thread-safe. Điều này được thể hiện qua protocol `Sendable`. Nếu một kiểu không phải `Sendable` mà lại bị dùng xuyên nhiều thread, compiler sẽ cảnh báo hoặc báo lỗi — bắt được bug tiềm tàng trước cả khi nó kịp xảy ra.

```swift
struct Profile {
    var name: String
    var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

func processProfile() async {
    let profile = Profile(name: "ChungHA", age: 28)

    Task {
        // Compiler sẽ báo WARNING/ERROR:
        // "Capture of 'profile' with non-sendable type 'Profile'
        //  in a '@Sendable' closure"
        print(profile.name)
    }
}
```

Một `struct` vốn dĩ đã là sendable khi dùng nội bộ. Nhưng nếu nó được dùng ra ngoài module thì nên khai báo `Sendable` một cách tường minh:

```swift
public struct Profile: Sendable {
    var name: String
    var age: Int
}

func processProfile() async {
   let profile = Profile(name: "ChungHA", age: 28)

    Task {
        print(profile.name) // không còn warning nữa
    }
}
```

## `nonisolated` là gì?

**Ý tưởng:**

- Đôi khi Xcode toàn warning *""You cannot access this actor property synchronously"* Nhưng bạn biết property đó là an toàn, vì nó là một hằng (biến `let`) hoặc chỉ là một hàm helper đơn giản nào đó

- `nonisolated` chính là cách nói với Swift: "Cái hàm hoặc property cụ thể này không đụng vào bất kỳ dữ liệu thay đổi nào. Cứ để ai cũng đọc được ngay mà không cần `await`."

Còn không `nonisolated` thì await bình thường đấy nhé (

**Đoạn này Phần 2 mình sẽ nói kỹ hơn chút nữa, ko đơn giản actor và nonisolated như bẹn nghĩ**

### Ví dụ

```swift
actor BankBranch {
    let code: String = "1998"      // Hằng 
    var vaultCash: Double = 1000000.0    // Trạng thái thay đổi được

    // Đánh dấu nonisolated vì nó chỉ đọc hằng branchCode
    nonisolated func getCodeDetails() -> String {
        return "Code: \(code)" // no need awiat
    }
}
```

## Tóm tắt nhanh

| Khái niệm | Vai trò |
|---|---|
| **async / await** | Viết code bất đồng bộ mà đọc như code tuần tự, không block Main Thread |
| **Task** | Khởi động một công việc async từ ngữ cảnh thường sync (nút bấm...) |
| **async let** | Chạy đồng thời một số cố định các tác vụ độc lập |
| **TaskGroup** | Chạy đồng thời  một số lượng động các tác vụ, thu kết quả khi xong |
| **actor** | Bảo vệ shared state, mỗi lúc chỉ một tác vụ đụng vào -> chống data race |
| **@MainActor** | Đảm bảo code cập nhật UI chạy trên Main Thread |
| **Sendable** | Đánh dấu kiểu an toàn để truyền qua lại giữa các thread |
| **nonisolated** | Cho phép đọc phần dữ liệu bất biến của actor mà không cần `await` |

Tóm lại theo mình đọc vào file keep thì idea Swift Concurrency dời gánh nặng "phải cẩn thận với thread" từ vai Dev (ở runtime) sang cho compiler (ở compile-time). Bạn viết code thẳng như suy nghĩ, còn compiler lo phần safe.

Mọi người chờ phần 2 để tiếp tục đào sâu hơn nữa nhé

---
