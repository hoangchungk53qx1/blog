---
title: "ImmutableList vs List trong Jetpack Compose"
description: "Nghĩ lại về best practice sau khi có Strong Skipping Mode"
icon: "article"
date: "2026-06-13T10:00:00+07:00"
lastmod: "2026-06-13T10:00:00+07:00"
draft: false
toc: true
weight: 999
---

# Tác giả : ChungHA 

# ImmutableList vs List trong Jetpack Compose: Nghĩ lại "best practice" sau Strong Skipping Mode

Anh em nào làm Compose lâu lâu một chút chắc đều thuộc nằm lòng câu này: "Muốn composable skip recomposition thì đừng có truyền `List<T>` vào, phải xài `ImmutableList<T>` của `kotlinx` cơ."

Câu đó không sai, ít nhất là ở thời điểm nó ra đời. Nhưng từ ngày **Strong Skipping Mode** được bật mặc định, cái nền móng phía sau lời khuyên này đã khác đi rồi. Vậy nên hôm nay mình muốn ngồi lại, soi từng trường hợp một xem `ImmutableList` có còn là thứ "bắt buộc phải có" nữa hay không.

Trong bài này mình sẽ bóc tách 3 tình huống cụ thể, kèm theo chi phí thật sự mà mỗi lựa chọn phải trả. Đi thôi.

## Ôn lại một chút: ngày xưa vì sao cứ phải ImmutableList?

Trong Compose, một composable chỉ **skip** được (tức bỏ qua recomposition) khi tất cả tham số của nó vừa **stable** vừa **không đổi**.

Mà rắc rối ở chỗ: `List<T>`, `Set<T>`, `Map<T>` trong Kotlin đều bị Compose compiler xem là **unstable**. Vì sao? Vì cái interface `List<T>` chẳng hứa hẹn gì về chuyện bất biến cả — đằng sau một `List` rất có thể là một `MutableList` đang bị sửa lén ở đâu đó. Compiler không tài nào biết chắc được, nên thôi cứ cho là unstable cho lành.

```kotlin
// Trước Strong Skipping: MyList KHÔNG skip được
// vì tham số items kiểu List<T> -> bị coi là unstable
@Composable
fun MyList(items: List<Item>) {
    Column {
        items.forEach { ItemRow(it) }
    }
}
```

Chỉ cần đúng một tham số unstable thôi là cả composable mất luôn khả năng skip. Thế là người ta nghĩ ra cách xài kiểu dữ liệu có cam kết bất biến hẳn hoi:

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.toImmutableList

@Composable
fun MyList(items: ImmutableList<Item>) { // stable -> skip được
    Column {
        items.forEach { ItemRow(it) }
    }
}

// chỗ gọi
MyList(items = viewModelItems.toImmutableList())
```

`ImmutableList` được đánh dấu là stable, nên composable lấy lại được khả năng skip. Đó là toàn bộ câu chuyện đằng sau cái best practice ngày xưa.

## Strong Skipping Mode đã thay đổi điều gì?

> Nếu bạn chưa rõ Strong Skipping chạy thế nào, nên ghé đọc bài [Strong Skipping & Lambda Memoization](../../compose/compose-skip-mode/) trước cho dễ theo dõi nhé.

Nói ngắn gọn thế này: hồi trước chỉ cần dính **một** tham số unstable là composable mất quyền skip. **Strong Skipping Mode** đổi luôn cái luật đó:

- Restartable composable **vẫn skip được** kể cả khi nhận tham số unstable.
- Compose giờ tin vào việc **so sánh lúc runtime** nhiều hơn, thay vì đoán mò một cách bảo thủ ngay từ compile time.

Cụ thể là với tham số unstable (kiểu `List<T>`), Compose sẽ so bằng **instance equality** — tức so địa chỉ tham chiếu (`===`). Nếu vẫn là cùng một instance được truyền vào, composable cứ thế mà skip ngon lành.

Chính điều này khiến cái niềm tin cũ — "collection unstable là luôn luôn dở" — **không còn đúng trong mọi trường hợp nữa.** Và đây mới là cái lõi của cả bài viết hôm nay.

## Soi từng trường hợp một

Cho công bằng, mình xét cả hai theo đúng cách Compose vận hành:

- **`List<T>`** → Compose so bằng **instance equality** (`===`), nhẹ tênh, O(1).
- **`ImmutableList<T>`** → được coi là stable nên Compose so bằng **structural equality** (`equals()`), tốn O(N). Chưa kể còn phải trả thêm chi phí **convert** `toImmutableList()` cũng O(N) nữa.

### Trường hợp 1: List không bao giờ đổi (vẫn cùng một instance)

Đây là khi list được tạo ra một lần rồi giữ nguyên y vậy suốt vòng đời.

```kotlin
// list giữ nguyên instance qua mọi lần recomposition
val items = remember { loadStaticItems() } // List<Item>
MyList(items = items)
```

- `List<T>`: instance không đổi → `===` đúng → **skip**. Chi phí O(1).
- `ImmutableList<T>`: cũng skip được đấy, nhưng phải gánh thêm một lần convert `toImmutableList()` mất O(N).

**Chốt:** Cả hai đều skip. `ImmutableList` chẳng được lợi gì hơn, lại còn tốn thêm một lần convert (may là chỉ một lần nên cũng không đáng kể lắm). **Coi như hoà, nhưng nghiêng về `List`.**

### Trường hợp 2: Instance đổi khi nội dung thật sự đổi

Đây là kịch bản "lành mạnh" và hay gặp nhất: nội dung đổi thì sinh list mới, nội dung y nguyên thì giữ instance cũ.

```kotlin
// Chỉ khi data thật sự đổi mới đẻ ra list mới
val items: List<Item> by viewModel.items.collectAsState()
MyList(items = items)
```

- `List<T>`: instance đổi → `===` sai → recompose (đúng ý mình vì nội dung đã khác thật). Chi phí O(1).
- `ImmutableList<T>`: phải convert O(N) + so `equals()` O(N), rồi rốt cuộc **vẫn phải recompose** vì nội dung khác thật mà.

**Chốt:** Recompose ở đây là việc bắt buộc rồi. `ImmutableList` chỉ tốn thêm convert với so structural O(N) — **toàn bộ là công cốc.** `List<T>` thắng đậm.

### Trường hợp 3: Nội dung KHÔNG đổi nhưng cứ liên tục đẻ instance mới

Đây mới là chỗ **duy nhất** mà `ImmutableList` có đất diễn thật sự.

```kotlin
// Mỗi lần recompose lại map ra một list MỚI dù nội dung y chang
@Composable
fun Screen(state: UiState) {
    // .map { } đẻ ra instance List mới mỗi lần -> === luôn sai
    MyList(items = state.rawItems.map { it.toUi() })
}
```

- `List<T>`: instance mới liên tục → `===` luôn sai → recompose thừa dù nội dung chẳng đổi.
- `ImmutableList<T>`: nhờ `equals()` nó nhận ra nội dung y hệt → **skip** được phần trên của cây. Đây đúng là lợi ích thật.

Nhưng — và cái "nhưng" này quan trọng lắm:

- Bạn vẫn phải trả chi phí **convert** O(N) + so `equals()` O(N) mỗi lần.
- Nếu composable con là **lazy** (kiểu `LazyColumn`), thì khối lượng việc vốn đã bị giới hạn trong vùng đang nhìn thấy rồi, nên cái lợi "skip phần trên" nhiều khi **không bõ** so với chi phí O(N) bỏ ra.

```kotlin
// Với LazyColumn, item chỉ compose theo vùng đang hiển thị
// nên cái lợi skip thượng nguồn của ImmutableList teo đi nhiều
LazyColumn {
    items(uiItems) { ItemRow(it) }
}
```

**Mẹo đáng giá:** thay vì gồng mình dùng `ImmutableList`, hãy **kéo Trường hợp 3 về Trường hợp 2** ngay từ đầu nguồn — đừng đẻ instance mới khi nội dung không đổi. Ví dụ với `StateFlow` + `distinctUntilChanged()`:

```kotlin
val items: StateFlow<List<Item>> =
    repository.itemsFlow
        .map { it.toUiItems() }
        .distinctUntilChanged() // lọc bỏ mấy instance trùng nội dung
        .stateIn(scope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

Hoặc đơn giản hơn, cứ `remember` với đúng key để khỏi map lại vô cớ:

```kotlin
val uiItems = remember(state.rawItems) { state.rawItems.map { it.toUi() } }
```

Khi instance đã ổn định rồi thì bạn quay về Trường hợp 1/2, và lúc đó `List<T>` thường là quá đủ.

## Bảng tổng kết

| Trường hợp | `List<T>` | `ImmutableList<T>` | Ai thắng |
|---|---|---|---|
| **TH 1** – cùng instance, không đổi | skip, O(1) | skip, + convert O(N) một lần | `List` (hoà, nghiêng `List`) |
| **TH 2** – instance đổi khi nội dung đổi | recompose, O(1) | recompose, + convert & `equals()` O(N) | **`List`** |
| **TH 3** – nội dung không đổi, instance mới liên tục | recompose thừa | skip nhờ `equals()`, nhưng tốn O(N) | Tuỳ — `Immutable` chỉ thắng khi không phải lazy |

## Lời kết

Từ khi Strong Skipping Mode thành mặc định, chuyện cố convert `List<T>` sang `ImmutableList<T>` chỉ để "lấy lại khả năng skip" **đã không còn cần thiết trong mọi trường hợp nữa.**

Mấy ý cần nhớ:

1. **Strong Skipping** cho phép composable skip được kể cả với tham số unstable như `List<T>`, dựa trên instance equality (`===`).
2. **Trường hợp 2** (instance đổi khi nội dung đổi) là hay gặp nhất, và ở đây `List<T>` luôn ngon hơn — `ImmutableList` chỉ tổ tốn công.
3. `ImmutableList` chỉ thật sự có lợi ở **Trường hợp 3**, mà ngay cả TH3 cũng nên xử lý từ đầu nguồn (`distinctUntilChanged`, `remember` đúng key) để kéo nó về TH2.
4. Với `LazyColumn`, cái lợi của `ImmutableList` lại càng bé.
5. Nếu bạn **đang** xài `ImmutableList` rồi thì cũng chẳng cần vội bỏ — nó không gây hại, chỉ là không còn bắt buộc thôi.

Triết lý cuối cùng nghe rất đời: **đừng tối ưu non.** Đừng mặc định cứ `ImmutableList` là bắt buộc cho hiệu năng Compose. Cứ **profile** đàng hoàng, thấy bottleneck thật rồi mới tối ưu đúng chỗ. Như tác giả bài gốc nói: *"Thấy bottleneck thật rồi tối ưu là vừa đẹp."*

---