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

# Tác giả : ChungHA (RxMobileTeam)

# ImmutableList vs List trong Jetpack Compose: Nghĩ lại "best practice" sau Strong Skipping Mode

Nếu bạn đã làm Compose được một thời gian, chắc chắn từng nghe lời khuyên "kinh điển": **muốn composable skip được recomposition thì đừng truyền `List<T>` vào, hãy dùng `ImmutableList<T>` của `kotlinx.collections.immutable`.**

Lời khuyên này không sai — ở thời điểm nó ra đời. Nhưng từ khi **Strong Skipping Mode** được bật mặc định, giả định nền tảng phía sau nó đã thay đổi. Và đây là lúc chúng ta nên ngồi lại, nhìn kỹ từng trường hợp xem `ImmutableList` có còn là "bắt buộc" hay không.

Bài này mình sẽ mổ xẻ 3 case cụ thể, kèm phân tích chi phí thật sự đằng sau mỗi lựa chọn.

## Ôn lại: tại sao ngày xưa phải dùng ImmutableList?

Trong Compose, một composable chỉ **skip** (bỏ qua recomposition) được khi tất cả tham số của nó **stable** và **không đổi**.

Vấn đề: `List<T>`, `Set<T>`, `Map<T>` trong Kotlin bị Compose compiler coi là **unstable**. Lý do là interface `List<T>` không hứa hẹn gì về tính bất biến — đằng sau một `List` có thể là một `MutableList` bị sửa ngầm ở nơi khác. Compiler không có cách nào biết chắc, nên đành coi là unstable cho an toàn.

```kotlin
// Trước Strong Skipping: MyList bị coi là KHÔNG skippable
// vì tham số items là List<T> -> unstable
@Composable
fun MyList(items: List<Item>) {
    Column {
        items.forEach { ItemRow(it) }
    }
}
```

Chỉ cần một tham số unstable, cả composable mất khả năng skip. Giải pháp ngày đó là dùng kiểu có cam kết bất biến:

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.toImmutableList

@Composable
fun MyList(items: ImmutableList<Item>) { // stable -> skippable
    Column {
        items.forEach { ItemRow(it) }
    }
}

// nơi gọi
MyList(items = viewModelItems.toImmutableList())
```

`ImmutableList` được đánh dấu stable, nên composable lấy lại được khả năng skip. Đó là toàn bộ lý do của best practice cũ.

## Strong Skipping Mode thay đổi điều gì?

> Nếu bạn chưa nắm rõ Strong Skipping hoạt động ra sao, có thể đọc bài [Strong Skipping & Lambda Memoization](../../compose/compose-skip-mode/) trước cho dễ theo dõi.

Tóm tắt nhanh: trước đây, chỉ cần **một** tham số unstable là composable mất quyền skip. **Strong Skipping Mode** đổi luật chơi:

- Restartable composable **vẫn có thể skip** kể cả khi nhận tham số unstable.
- Compose dựa nhiều hơn vào **so sánh ở runtime** thay vì những giả định bảo thủ ở compile time.

Cụ thể, với tham số unstable (như `List<T>`), Compose sẽ so sánh bằng **instance equality** — tức là so sánh tham chiếu (`===`). Nếu cùng một instance được truyền lại, composable vẫn skip bình thường.

Điều này khiến giả định cũ — "collection unstable = luôn luôn xấu" — **không còn đúng phổ quát nữa.** Và đó là điểm mấu chốt của cả bài viết này.

## Phân tích từng case

Để công bằng, ta xét cả hai theo đúng cơ chế Compose dùng:

- **`List<T>`** → Compose so sánh bằng **instance equality** (`===`), chi phí O(1).
- **`ImmutableList<T>`** → được coi stable, Compose so sánh bằng **structural equality** (`equals()`), chi phí O(N). Ngoài ra còn tốn chi phí **convert** `toImmutableList()` cũng O(N).

### Case 1: List không bao giờ đổi (cùng một instance)

Đây là trường hợp list được tạo một lần rồi giữ nguyên instance suốt vòng đời.

```kotlin
// list giữ nguyên instance qua các lần recomposition
val items = remember { loadStaticItems() } // List<Item>
MyList(items = items)
```

- `List<T>`: instance không đổi → `===` đúng → **skip**. Chi phí O(1).
- `ImmutableList<T>`: cũng skip, nhưng phải trả thêm chi phí convert `toImmutableList()` một lần O(N).

**Kết luận:** Cả hai đều skip được. `ImmutableList` không mang lại lợi ích gì, chỉ thêm một lần convert (may là chỉ một lần nên không đáng kể). **Hoà, nghiêng nhẹ về `List`.**

### Case 2: Instance đổi khi nội dung thực sự đổi

Đây là trường hợp "lành mạnh" và phổ biến nhất: nội dung đổi thì list mới được tạo, nội dung không đổi thì giữ nguyên instance.

```kotlin
// Mỗi lần data thực sự thay đổi mới sinh list mới
val items: List<Item> by viewModel.items.collectAsState()
MyList(items = items)
```

- `List<T>`: instance đổi → `===` sai → recompose (đúng như mong muốn vì nội dung đã đổi). Chi phí O(1).
- `ImmutableList<T>`: phải convert O(N) + so sánh `equals()` O(N), rồi cuối cùng **vẫn phải recompose** vì nội dung khác thật.

**Kết luận:** Recomposition là cần thiết ở đây. `ImmutableList` chỉ tốn thêm convert + so sánh structural O(N) — **toàn bộ là overhead thuần tuý.** `List<T>` thắng rõ ràng.

### Case 3: Nội dung KHÔNG đổi nhưng instance mới liên tục được tạo

Đây là case **duy nhất** mà `ImmutableList` thật sự có đất diễn.

```kotlin
// Mỗi lần recompose lại map ra một list MỚI dù nội dung y hệt
@Composable
fun Screen(state: UiState) {
    // .map { } tạo instance List mới mỗi lần -> === luôn sai
    MyList(items = state.rawItems.map { it.toUi() })
}
```

- `List<T>`: instance mới liên tục → `===` luôn sai → recompose thừa dù nội dung không đổi.
- `ImmutableList<T>`: dùng `equals()` nhận ra nội dung y hệt → **skip** được phần phía trên cây. Đây là lợi ích thật.

Nhưng — và đây là chữ "nhưng" quan trọng:

- Bạn vẫn trả chi phí **convert** O(N) + so sánh `equals()` O(N) mỗi lần.
- Nếu composable con là **lazy** (như `LazyColumn`), công việc vốn đã bị giới hạn trong vùng đang hiển thị, nên cái lợi từ việc "skip phần trên" có thể **không bù nổi** chi phí O(N) bỏ ra.

```kotlin
// Với LazyColumn, item chỉ compose theo vùng nhìn thấy
// nên lợi ích skip thượng nguồn của ImmutableList giảm đi nhiều
LazyColumn {
    items(uiItems) { ItemRow(it) }
}
```

**Mẹo quan trọng:** thay vì gồng dùng `ImmutableList`, hãy **biến Case 3 thành Case 2** ngay từ thượng nguồn — đừng tạo instance mới khi nội dung không đổi. Ví dụ với `StateFlow` + `distinctUntilChanged()`:

```kotlin
val items: StateFlow<List<Item>> =
    repository.itemsFlow
        .map { it.toUiItems() }
        .distinctUntilChanged() // lọc các instance trùng nội dung
        .stateIn(scope, SharingStarted.WhileSubscribed(5_000), emptyList())
```

Hoặc đơn giản là `remember` đúng key để không map lại vô cớ:

```kotlin
val uiItems = remember(state.rawItems) { state.rawItems.map { it.toUi() } }
```

Khi instance đã ổn định, bạn quay về Case 1/Case 2 và `List<T>` thường là đủ.

## Bảng tổng kết

| Trường hợp | `List<T>` | `ImmutableList<T>` | Người thắng |
|---|---|---|---|
| **Case 1** – cùng instance, không đổi | skip, O(1) | skip, + convert O(N) một lần | `List` (hoà, nghiêng `List`) |
| **Case 2** – instance đổi khi nội dung đổi | recompose, O(1) | recompose, + convert & `equals()` O(N) | **`List`** |
| **Case 3** – nội dung không đổi, instance mới liên tục | recompose thừa | skip nhờ `equals()`, nhưng tốn O(N) | Tuỳ — `Immutable` chỉ thắng khi không phải lazy |

## Kết luận

Sau khi Strong Skipping Mode trở thành mặc định, việc cố convert `List<T>` sang `ImmutableList<T>` chỉ để "lấy lại khả năng skip" **không còn cần thiết một cách phổ quát nữa.**

Những điều cần nhớ:

1. **Strong Skipping** cho phép composable skip ngay cả với tham số unstable như `List<T>`, dựa trên instance equality (`===`).
2. **Case 2** (instance đổi khi nội dung đổi) là phổ biến nhất, và ở đây `List<T>` luôn tốt hơn — `ImmutableList` chỉ là overhead.
3. `ImmutableList` chỉ thật sự có lợi ở **Case 3**, mà ngay cả Case 3 cũng thường nên xử lý từ thượng nguồn (`distinctUntilChanged`, `remember` đúng key) để biến nó về Case 2.
4. Với `LazyColumn`, lợi ích của `ImmutableList` càng nhỏ.
5. Nếu bạn **đang** dùng `ImmutableList` rồi thì không cần vội bỏ — nó không gây hại, chỉ là không bắt buộc.

Triết lý cuối cùng rất "đời": **đừng tối ưu non.** Đừng mặc định `ImmutableList` là bắt buộc cho hiệu năng Compose. Hãy **profile** khi gặp bottleneck thật, rồi mới tối ưu đúng chỗ. Như tác giả bài gốc nói: *"Tối ưu khi bạn thấy bottleneck thật là đủ."*

---

*Bài viết tham khảo ý tưởng từ [ImmutableList vs. List in Jetpack Compose — dev.to/brian_stark_09127](https://dev.to/brian_stark_09127/immutablelist-vs-list-in-jetpack-compose-rethinking-best-practice-after-strong-skipping-mode-55p0).*
