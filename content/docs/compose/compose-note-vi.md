---
weight: 999
title: "Compose Recomposition"
description: ""
icon: "article"
date: "2025-08-21T17:45:43+07:00"
lastmod: "2025-08-21T17:45:43+07:00"
draft: true
toc: true
---

![Badge](https://hitscounter.dev/api/hit?url=https%3A%2F%2Fhoangchungk53qx1.github.io%2Fblog%2Fdocs%2Fcompose%2Fcompose-note-en%2F&label=Counter&icon=github&color=%23198754&message=&style=flat&tz=UTC)

Compose

a. Recomposition là gì? Khi nào diễn ra? Làm sao nhận biết và tối ưu recomposition không cần thiết?
* Recomposition là quá trình Jetpack Compose vẽ lại (rebuild) một phần hoặc toàn bộ UI tree khi dữ liệu (state) mà Composable phụ thuộc vào thay đổi.
* Diễn ra khi:
    * State (mutableStateOf, LiveData, StateFlow…) mà composable sử dụng bị đổi giá trị.
    * Các parameter truyền vào composable thay đổi.
* Nhận biết:
    * Dùng log hoặc debug tool: Log.d("Recomp", "Composable X recomposed!")
    * Dùng Layout Inspector (Android Studio) để theo dõi recomposition count.
* Tối ưu:
    * Tách nhỏ composable, chỉ nhận đúng những prop cần thiết.
    * Sử dụng remember, immutable object/class, @Stable.
    * Tránh truyền lambda/object tạo động trong body composable.
    * Tránh tạo list/collection mới trong mỗi recomposition.

b. Dùng remember, rememberSaveable, derivedStateOf như thế nào cho các use-case khác nhau?
* remember:
    * Lưu dữ liệu/memoization ngắn hạn (trong lifecycle của composable tree hiện tại).
    * Ví dụ: Cache object, callback, state UI chỉ cần nhớ trong một session nhỏ.
* rememberSaveable:
    * Lưu dữ liệu qua process death, cấu hình lại (configuration change).
    * Ví dụ: TextField value, form state, scroll position.
* derivedStateOf:
    * Tính toán dữ liệu phụ thuộc vào nhiều state khác, chỉ update khi input state thay đổi.
    * Ví dụ: List filter, tổng, view mode state tính từ nhiều biến nhỏ.

c. Khi nào nên dùng key trong LazyColumn/LazyList? Nếu không dùng key sẽ có rủi ro gì?
* Nên dùng key:
    * Khi hiển thị danh sách động (add, remove, move item).
    * Khi list chứa các item có thể thay đổi vị trí hoặc nội dung nhưng vẫn giữ identity.
* Rủi ro khi không dùng key:
    * Compose không biết item nào là cũ/mới → có thể reuse view sai, flicker UI, mất state tạm thời (input, scroll position…).
    * Performance giảm do phải vẽ lại nhiều hơn.
* Cách dùng:kotlinCopyitems(userList, key = { it.id }) { user -> ... }
* 

d. Compose xử lý slot API và composition local ra sao? Kể tên các trường hợp thực tế cần custom composition local.
* Slot API:
    * Compose cho phép truyền block UI vào component cha qua lambda (slot). Ví dụ: custom header, custom button content.
    * Ví dụ:
* kotlinCopy@Composable
* fun MyCard(content: @Composable () -> Unit) { ... }
* 
* CompositionLocal:
    * Cho phép truyền context hoặc data xuống sâu trong tree mà không cần truyền qua từng prop.
    * Dùng cho: theme, locale, spacing, user/session, permission state…
    * Ví dụ:
* kotlinCopyval LocalSpacing = compositionLocalOf { 8.dp }
* CompositionLocalProvider(LocalSpacing provides 16.dp) { ... }
* 

e. Làm sao custom layout composable? Tối ưu layout nặng trong Compose?
* Custom layout:
    * Dùng hàm Layout, hoặc modifier layout để build layout logic tùy ý.
    * Ví dụ:
* kotlinCopy@Composable
* fun MyCustomLayout(content: @Composable () -> Unit) {
*     Layout(content = content) { measurables, constraints ->
*         // Tính toán vị trí, size các con
*     }
* }
* 
* Tối ưu layout nặng:
    * Tránh lồng layout nhiều tầng.
    * Ưu tiên sử dụng composable chuẩn (Row, Column, Box) hoặc custom layout tối ưu.
    * Dùng Modifier.layoutId cho LazyLayout.
    * Reuse layout logic, không tạo quá nhiều composable nhỏ nếu không cần thiết.

f. Phân biệt @Composable, @Stable, @Immutable, @ReadOnlyComposable. Ảnh hưởng đến performance?
* @Composable: Đánh dấu hàm có thể dùng trong compose tree, được control bởi compose runtime.
* @Stable: Đảm bảo object/property không đổi bất ngờ, giúp Compose xác định khi nào skip recomposition.
* @Immutable: Tất cả property của class là val và immutable, Compose có thể yên tâm skip recomposition.
* @ReadOnlyComposable: Dùng cho hàm chỉ đọc, không side-effect; cho phép gọi từ bất kỳ thread và tối ưu runtime.
* Ảnh hưởng:
    * Gắn @Stable, @Immutable đúng chỗ giúp Compose skip recomposition, tối ưu hiệu suất.
    * @Composable là bắt buộc để Compose hiểu và quản lý hàm UI.

g. Khi truyền object mutable (list, class) vào Composable, có gì cần lưu ý? Giải pháp tối ưu?
* Lưu ý:
    * Nếu object thay đổi nhưng không tạo object mới, Compose không nhận biết để rebuild UI.
    * Nếu tạo object mới mỗi lần recomposition, sẽ gây hiệu suất kém.
* Giải pháp:
    * Dùng immutable object/data class, list bất biến.
    * Nếu phải truyền mutable object, đảm bảo luôn tạo object mới khi có thay đổi (copy, .toList()).
    * Dùng @Stable hoặc @Immutable để thông báo cho Compose về tính chất object.

h. Quá trình recomposition, skipping, invalidation trong Compose. Khi nào Compose tự động skip?
* Invalidation: Khi state hoặc prop mà Composable phụ thuộc vào thay đổi, Compose sẽ đánh dấu khu vực đó là “invalid”.
* Recomposition: Compose sẽ gọi lại Composable bị invalid, cập nhật UI tree.
* Skipping: Nếu Compose xác định input không đổi (dựa vào equals/hashCode/@Stable/@Immutable), nó sẽ tự động bỏ qua (skip) recomposition của Composable đó.
* Compose tự skip khi:
    * Parameter và state truyền vào không đổi (giá trị mới = giá trị cũ, object immutable).

i. Khi nào Compose có memory leak? Cách phát hiện và phòng tránh?
* Memory leak xảy ra khi:
    * Giữ reference đến context/activity/view/lifecycle bên ngoài scope composable.
    * Đăng ký listener hoặc callback nhưng không huỷ đúng lúc (vd: không removeListener trong DisposableEffect).
* Phát hiện:
    * Dùng profiler, LeakCanary, log warning trong DisposableEffect/LaunchedEffect.
* Phòng tránh:
    * Chỉ giữ reference sống trong scope phù hợp (không giữ context lâu).
    * Dùng DisposableEffect để cleanup resource.
    * Ưu tiên dùng remember, không truyền context vào biến toàn cục hoặc lambda lưu trữ lâu dài.

j. So sánh hiệu suất Compose với View truyền thống khi hiển thị list lớn (1k, 10k item). Làm sao đo và tối ưu?
* So sánh:
    * Compose với LazyColumn tối ưu lazy-loading tương tự RecyclerView, nhưng nếu không dùng key đúng hoặc code sai, dễ gây recomposition quá nhiều.
    * View truyền thống (RecyclerView) đã rất tối ưu cho list lớn nhờ view holder pattern.
* Đo hiệu suất:
    * Sử dụng Layout Inspector, Profiler của Android Studio, log recomposition count, track GC.
    * Đo fps, memory usage khi scroll list lớn.
* Tối ưu:
    * Luôn dùng key với LazyColumn.
    * Tránh rebuild item, tách item nhỏ.
    * Hạn chế logic/phép tính trong item composable.
    * Dùng Paging nếu data rất lớn.

