---
title: "WWDC27 có gì thay đổi"
description: "@State trong SwiftUI giờ đây là một Macro"
icon: "article"
date: "2026-06-13T09:00:00+07:00"
lastmod: "2026-06-13T09:00:00+07:00"
draft: false
toc: true
weight: 999
---

# Tác giả : ChungHA 

# WWDC27 có gì thay đổi: `@State` giờ đây là một Macro

Mỗi mùa WWDC trôi qua, Apple lại mang tới những thay đổi vừa nhỏ vừa lớn cho SwiftUI. Có những thứ rất "rầm rộ" trên sân khấu keynote, nhưng cũng có những thay đổi âm thầm dưới lớp vỏ framework mà nếu không để ý kỹ, bạn sẽ chẳng bao giờ nhận ra.

Một trong những thay đổi "âm thầm nhưng đáng giá" lần này là: **`@State` không còn là một Property Wrapper nữa, mà đã trở thành một Macro.**

Nghe qua thì có vẻ chỉ là chuyện đổi cách implement bên trong, code của bạn không cần sửa một dòng. Nhưng đằng sau nó là một cải tiến về hiệu năng và hành vi khởi tạo mà rất nhiều người trong chúng ta đã từng "đau đầu" vì nó. Cùng mình mổ xẻ trong bài viết này nhé.

## `@State` trước đây là gì?

Trước đây, `@State` là một **property wrapper**. Nếu bạn từng viết SwiftUI thì chắc chắn đã quen với nó:

```swift
struct PlayButton: View {
    @State private var isPlaying = false

    var body: some View {
        Button(isPlaying ? "Pause" : "Play") {
            isPlaying.toggle()
        }
    }
}
```

Điều quan trọng cần nhớ về SwiftUI là: **View là một giá trị (value), có thể bị tạo đi tạo lại rất nhiều lần.** Mỗi khi parent view re-render, struct `PlayButton` của bạn có thể bị khởi tạo lại từ đầu.

Vậy tại sao `isPlaying` không bị reset về `false` mỗi lần như vậy? Bởi vì giá trị của `@State` không nằm trong struct View, mà được **framework quản lý riêng**, và "đấu nối" (reconnect) lại vào view mỗi khi view được dựng lại.

Đó là cơ chế giúp state của bạn "sống sót" qua những lần view bị tạo lại.

## Vậy bây giờ thay đổi gì?

Từ phiên bản mới, `@State` được khai báo dưới dạng một **macro** thay vì property wrapper:

```swift
@attached(accessor, names: named(init), named(get), named(set))
@attached(peer, names: prefixed(`_`), prefixed(__), prefixed(`$`))
macro State()
```

Đừng quá lo lắng nếu nhìn đoạn khai báo trên thấy hơi "khó nuốt". Điều bạn cần nhớ là: **về phía developer, bạn không phải thay đổi gì cả.** Cú pháp vẫn y hệt như cũ:

```swift
@State private var isPlaying = false
```

Mental model về các prefix cũng được giữ nguyên:

```swift
@State private var count = 0

// mental model:
count   // wrapped value  - giá trị thật
$count  // binding        - dùng để truyền 2 chiều
_count  // backing storage - phần lưu trữ phía sau
```

Macro sẽ tự sinh ra các accessor (`init`, `get`, `set`) và các peer property (`_count`, `$count`...) ngay tại thời điểm biên dịch. Nói cách khác, những thứ trước đây property wrapper làm "ẩn dưới capo" thì giờ macro làm một cách tường minh hơn ở compile time.

> **Liên hệ với Kotlin một chút cho anh em Android dễ hình dung:**
>
> Nếu bạn quen Kotlin, có thể map khá sát như sau:
>
> - **Property Wrapper (cũ)** rất giống **property delegate** trong Kotlin — kiểu `val isPlaying by remember { ... }` hay `var x by Delegates.observable(...)`. Cả hai đều bọc một giá trị lại và chèn logic `get`/`set` vào lúc **runtime**.
> - **Macro (mới)** thì giống với **compiler plugin / KSP (Kotlin Symbol Processing)** hơn — code được **sinh ra ngay tại compile time**. Bạn viết `@State`, compiler "nở" (expand) nó ra thành các accessor và backing field thật, y như cách KSP sinh code cho Room, Moshi, hay cách `@Composable` được Compose compiler plugin biến đổi.
>
> Và điểm hay ho nhất — phần lazy initialization mình sắp nói tới — cũng có một "người anh em" rất quen thuộc bên Kotlin: từ khoá `by lazy { ... }`. Tinh thần giống hệt: **chỉ khởi tạo đúng một lần, vào lần truy cập đầu tiên, thay vì chạy đi chạy lại.**

## Cải tiến lớn nhất: Lazy Initialization 

Đây mới là phần đáng giá nhất của thay đổi này.

### Vấn đề trước đây

Hãy xem ví dụ với một `@Observable` class:

```swift
@Observable
final class ViewModel {
    var index = .zero

    init() {
        print("Init")
    }
}

struct MyView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        Button("Index: \(viewModel.index)") {
            viewModel.index += 1
        }
    }
}
```

Vấn đề nằm ở dòng `@State private var viewModel = ViewModel()`.

Vì View là value và bị tạo lại nhiều lần, nên biểu thức `ViewModel()` — tức là default value — **có thể bị thực thi lại mỗi khi parent view dựng lại struct này.** Nghĩa là `init()` của `ViewModel` bị gọi nhiều lần, in ra `"Init"` liên tục, dù cuối cùng SwiftUI chỉ giữ lại đúng một instance.

Với một init "nhẹ" thì không sao. Nhưng nếu trong `init()` bạn:

- Mở một subscription,
- Cấp phát tài nguyên nặng,
- Gọi network, đăng ký observer...

thì đây là một sự lãng phí (và đôi khi là bug) thực sự.

### Cách "lách" cũ mà nhiều người dùng

Để tránh việc init bị gọi nhiều lần, một pattern phổ biến là dùng **optional state** rồi khởi tạo trong `.task`:

```swift
@State private var viewModel: ViewModel?

var body: some View {
    MyView(viewModel: viewModel)
        .task {
            viewModel = ViewModel()
        }
}
```

Cách này hoạt động, nhưng nó làm code xấu đi: phải xử lý optional ở khắp nơi, và logic khởi tạo bị tách rời khỏi nơi khai báo.

### Sau khi đổi sang Macro

Với cách implement bằng macro, **default value được khởi tạo một cách lười (lazy)** — đúng **một lần duy nhất** vào thời điểm SwiftUI tạo storage cho state, chứ **không phải** mỗi lần view bị dựng lại.

Nghĩa là với ví dụ trên, `print("Init")` giờ chỉ chạy đúng một lần. 🎉

Và pattern optional-state-rồi-khởi-tạo-trong-`.task` ở trên giờ **không còn cần thiết** cho trường hợp khởi tạo bằng default value nữa. Bạn cứ viết tự nhiên:

```swift
@State private var viewModel = ViewModel()
```

là đủ.

## Một lưu ý quan trọng: khởi tạo trong `init` của View thì khác

Đừng nhầm lẫn giữa **default value** và việc **gán giá trị trong initializer của View**. Hai thứ này hành xử khác nhau.

```swift
struct MyView: View {
    @State private var viewModel: ViewModel

    init(id: Item.ID) {
        viewModel = ViewModel(id: id)
    }

    var body: some View {
        // ...
    }
}
```

Trong trường hợp này, `init(id:)` của View **vẫn có thể bị gọi nhiều lần** (vì View vẫn là value bị tạo lại). Cải tiến lazy initialization ở trên **không áp dụng** cho phần gán trong initializer.

Vì vậy, **đừng dùng pattern này để inject dependency dựa trên tham số từ parent** — vì bạn không kiểm soát được số lần nó chạy. Lazy init chỉ "thần kỳ" với default value, không phải với assignment trong `init`.

## Nhắc lại các pattern state để dùng cho đúng

Nhân tiện nói về `@State`, mình tóm tắt lại khi nào dùng cái gì cho khỏi nhầm:

```swift
// View này sở hữu local value state.
@State private var count = 0

// View này sở hữu một Observable reference.
@State private var book = Book()

// Child có thể thay thế value hoặc reference của parent.
@Binding var book: Book?

// Child cần binding tới các property của một Observable object.
@Bindable var book: Book
```

Diễn giải nhanh:

- **`@State`**: dùng khi view này là chủ sở hữu của state, dù đó là value type hay reference (`@Observable`).
- **`@Binding`**: dùng khi child cần **thay thế** chính cái biến mà parent đang giữ (ví dụ set `book = nil`).
- **`@Bindable`**: dùng khi bạn đã có một `@Observable` object và muốn tạo binding tới **property bên trong** nó (ví dụ `$book.title` để đưa vào `TextField`).

Ví dụ minh hoạ cho `@Observable` được truyền xuống child mà không cần binding:

```swift
@Observable
class Library {
    var name = "My library of books"
}

struct ContentView: View {
    @State private var library = Library()

    var body: some View {
        LibraryView(library: library)
    }
}

struct LibraryView: View {
    var library: Library

    var body: some View {
        Text(library.name)
    }
}
```

Vì `Library` là `@Observable` reference, child chỉ cần nhận trực tiếp object là đủ để observe và mutate property của nó, không cần `@Binding`:

```swift
struct BookCheckoutView: View {
    var book: Book

    var body: some View {
        Button(book.isAvailable ? "Check out book" : "Return book") {
            book.isAvailable.toggle()
        }
    }
}
```

Chỉ khi child cần **đổi luôn cái reference** (ví dụ xoá hẳn book về `nil`) thì mới cần `@Binding`:

```swift
struct ContentView: View {
    @State private var book: Book?

    var body: some View {
        DeleteBookView(book: $book)
            .task {
                book = Book()
            }
    }
}

struct DeleteBookView: View {
    @Binding var book: Book?

    var body: some View {
        Button("Delete book") {
            book = nil
        }
    }
}
```

Và khi cần binding tới property cụ thể của một observable object thì dùng `@Bindable`:

```swift
struct BookEditorView: View {
    @Bindable var book: Book

    var body: some View {
        TextField("Title", text: $book.title)
    }
}
```

## Tổng kết

Thay đổi `@State` từ property wrapper sang macro là một ví dụ điển hình cho triết lý của SwiftUI: **giữ nguyên trải nghiệm của developer ở bề mặt, nhưng cải tiến mạnh mẽ ở bên dưới.**

Những điều cần nhớ:

1. **Cú pháp không đổi** — bạn vẫn viết `@State private var ...` như cũ.
2. **Default value giờ được khởi tạo lười** — chạy đúng một lần khi tạo storage, không còn bị gọi lại mỗi lần view re-render.
3. **Pattern optional-state + `.task`** để né init nhiều lần giờ **không còn cần thiết** cho trường hợp default value.
4. **Cẩn thận với việc gán trong `init` của View** — nó vẫn chạy nhiều lần, đừng dùng cho dependency injection.
5. Nhớ phân biệt rõ **`@State` / `@Binding` / `@Bindable`** để chọn đúng công cụ.

Một thay đổi nhỏ trên slide WWDC, nhưng lại gỡ bỏ được một trong những "workaround" mệt mỏi nhất khi làm việc với `@Observable`. Đôi khi những cải tiến đáng giá nhất lại là những thứ âm thầm như vậy.

---
