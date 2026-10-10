# BÁO CÁO CÁ NHÂN BÀI TẬP LỚN FOURSMART
## README_11_11 – CART & CHECKOUT

**Họ tên:** Nguyễn Quang Hưng  
**GitHub (theo lịch sử commit):** Nguyễn Hưng  
**Nhánh:** `feature/cart-checkout`  
**Repository:** https://github.com/LaiHuy05/FourSmart-new

# I. MỤC TIÊU BÁO CÁO

Trong báo cáo này, em tập trung trình bày khả năng sử dụng AI Agent, kiểm soát mã nguồn và kết hợp các công cụ kiểm thử thông qua việc phân tích các nghiệp vụ giỏ hàng và thanh toán của dự án FourSmart.

Em chú trọng việc giải thích nguyên lý hoạt động, khả năng sửa chức năng và kiểm soát những trường hợp có thể gây lỗi dữ liệu.

# II. Ý 1 – XÂY DỰNG PHẦN MỀM VỚI AI AGENT 

## 2.1. Nhận thức về AI Agent

Theo em, AI Agent là công cụ có khả năng hỗ trợ lập trình viên phân tích yêu cầu, tìm kiếm mã nguồn, đề xuất giải pháp và hỗ trợ kiểm thử.

Tuy nhiên, việc giao hoàn toàn trách nhiệm viết code cho AI có thể gây ra những vấn đề khi xử lý dữ liệu quan trọng, đặc biệt là thanh toán và tồn kho.

Vì vậy, em xác định AI là công cụ hỗ trợ còn việc kiểm soát nghiệp vụ và tính đúng đắn của chương trình thuộc trách nhiệm của lập trình viên.

## 2.2. Quy trình sử dụng AI

Em xác định quy trình:

1. Phân tích luồng giỏ hàng và thanh toán.
2. Xác định Model, Controller và Service liên quan.
3. Cung cấp ngữ cảnh cho AI.
4. Yêu cầu giải thích nghiệp vụ và đề xuất thay đổi.
5. Kiểm tra câu truy vấn, transaction và điều kiện bảo mật.
6. Thử nghiệm với các tình huống thành công, thất bại.
7. Xem Git diff và commit thay đổi hợp lệ.

## 2.3. Ví dụ áp dụng AI vào bài toán kiểm tra tồn kho

Các file liên quan:

- `mvc/model/CartModel.php`
- `mvc/service/CheckoutService.php`
- `mvc/model/ProductModel.php`

Trong `CheckoutService::createOrder()`, chương trình kiểm tra tồn kho của từng sản phẩm trước khi tạo đơn hàng.

Ví dụ, nếu giảng viên yêu cầu hệ thống không cho phép thanh toán với giỏ hàng có sản phẩm hết hàng, em có thể dùng prompt:

> Hãy phân tích luồng thanh toán trong CheckoutService.php của dự án FourSmart. Xác định nơi kiểm tra tồn kho, giải thích cách sử dụng database transaction và đề xuất kiểm thử khi số lượng đặt vượt tồn kho. Không thay đổi cấu trúc MVC hoặc tên bảng dữ liệu. Đặc biệt kiểm tra khả năng dữ liệu bị cập nhật một phần khi phát sinh lỗi.

Sau khi phân tích code, em thấy phương thức `createOrder()` đã sử dụng:

```php
$connection->beginTransaction();
```

Khi xử lý thành công:

```php
$connection->commit();
```

Khi xảy ra lỗi:

```php
$connection->rollBack();
```

Em hiểu rằng transaction giúp nhóm những thay đổi dữ liệu liên quan vào một đơn vị xử lý.

Khi có lỗi trong quá trình tạo đơn, rollback giúp tránh tình trạng đã lưu một phần dữ liệu nhưng đơn hàng chưa được xử lý hoàn chỉnh, với điều kiện các thao tác tham gia cùng transaction và hệ quản trị hỗ trợ transaction.

Ngoài ra, em cũng cần kiểm tra các tình huống hai người đồng thời mua sản phẩm gần hết hàng, vì transaction thông thường chưa tự bảo đảm ngăn được mọi lỗi cạnh tranh dữ liệu.

# III. Ý 2 – KIỂM SOÁT MÃ NGUỒN 

## 3.1. Ý 2a – Hiểu và sửa tính năng

### 3.1.1. Cấu trúc mã nguồn

Các file chính:

- `mvc/model/CartModel.php`
- `mvc/controller/client/cart_controller.php`
- `mvc/controller/client/checkout_controller.php`
- `mvc/service/CheckoutService.php`
- `mvc/model/OrderModel.php`
- `mvc/view/client/cart.php`
- `mvc/view/client/pay.php`

### 3.1.2. Phân tích CartModel

CartModel quản lý dữ liệu giỏ hàng và chi tiết giỏ hàng.

Những phương thức quan trọng gồm:

- `findByUser()`: Tìm giỏ hàng của người dùng.
- `detailsByCart()`: Lấy chi tiết giỏ.
- `addItem()`: Thêm sản phẩm.
- `increaseQuantity()`: Tăng số lượng.
- `decreaseQuantity()`: Giảm số lượng.
- `deleteItem()`: Xóa sản phẩm khỏi giỏ.
- `detailBelongsToUser()`: Kiểm tra chi tiết giỏ hàng thuộc đúng người dùng.
- `clearDetails()`: Xóa chi tiết giỏ hàng.

Em hiểu rằng kiểm tra quyền sở hữu dữ liệu giúp ngăn người dùng thao tác lên giỏ hàng của tài khoản khác.

### 3.1.3. Phân tích CheckoutService

CheckoutService gồm ba phương thức chính:

**summary():** Tính tổng số lượng và tổng giá trị giỏ hàng.

**validateCustomer():** Kiểm tra thông tin khách hàng như tên, email, số điện thoại và địa chỉ.

**createOrder():** Thực hiện quy trình tạo đơn hàng.

Luồng tạo đơn:

1. Tính tổng tiền và tổng số lượng.
2. Tạo mã đơn hàng.
3. Bắt đầu database transaction.
4. Kiểm tra tồn kho từng sản phẩm.
5. Tạo đơn hàng.
6. Lưu chi tiết từng sản phẩm trong đơn hàng.
7. Trừ tồn kho.
8. Xóa những sản phẩm đã thanh toán khỏi giỏ.
9. Commit transaction khi thành công.
10. Rollback nếu xuất hiện lỗi.

Em hiểu rằng việc xử lý tạo đơn hàng phải bảo đảm tính nhất quán giữa đơn hàng, chi tiết đơn, tồn kho và giỏ hàng.

### 3.1.4. Ví dụ sửa code trực tiếp

Nếu thầy yêu cầu số lượng sản phẩm trong giỏ hàng không được giảm xuống 0, em sẽ tìm phương thức `decreaseQuantity()` trong CartModel.
Trong mã nguồn hiện tại có câu truy vấn:

```sql
UPDATE cartdetail
SET cd_quantity = cd_quantity - 1
WHERE cd_id = ?
AND cd_quantity > 1
```

Điều kiện `cd_quantity > 1` giúp ngăn thao tác giảm số lượng xuống 0 từ giá trị 1.

Em hiểu rằng điều kiện này được đặt trong SQL nhằm bảo vệ dữ liệu ngay tại thời điểm cập nhật.

Nếu thầy yêu cầu giới hạn tối đa 10 sản phẩm cho một chi tiết giỏ hàng, em cần phân tích cả chức năng tăng số lượng và thêm sản phẩm để xác định nơi áp dụng điều kiện.

Ngoài việc kiểm tra ở Controller, em cũng cân nhắc điều kiện cập nhật trực tiếp trong SQL để hạn chế lỗi khi có nhiều request.

## 3.2. Ý 2b – Skill phân tích quy trình thanh toán

Theo em, skill giúp AI Agent thực hiện công việc theo các bước nhất quán.

Em có thể xây dựng hoặc sử dụng skill để phân tích luồng thanh toán:

`ClientCheckoutController → CheckoutService → OrderModel / CartModel / ProductModel → Database`

Khi thay đổi nghiệp vụ, skill có thể hướng dẫn Agent xác định:

1. Điểm bắt đầu của request.
2. Điều kiện đăng nhập và CSRF.
3. Những Model được sử dụng.
4. Những bảng dữ liệu bị thay đổi.
5. Những bước nằm trong transaction.
6. Các trường hợp có thể rollback.
7. Những test case cần thực hiện.

Với Graphify hoặc công cụ phân tích phụ thuộc, em có thể nghiên cứu cách biểu diễn các quan hệ này để dễ hiểu hơn tác động của thay đổi.

**Minh chứng cần bổ sung:** Kết quả chạy skill thực tế và nhận xét của em về tính chính xác.

## 3.3. Ý 2c – Sử dụng Git log và Git diff

Để theo dõi lịch sử nhánh:

```bash
git log feature/cart-checkout --oneline
```

Để xem các thay đổi của CartModel:

```bash
git log --oneline -- mvc/model/CartModel.php
```

Để kiểm tra một commit quan trọng:

```bash
git show 7bba01bc6
```

Commit `7bba01bc6` có nội dung kiểm tra biến thể và tồn kho khi mua hàng.

Một số commit đáng chú ý khác:

- `ea3316ec9`: Hoàn thiện model giỏ hàng.
- `6f7efd754`: Thêm trang xem giỏ hàng.
- `048297d1e`: Thêm thao tác giỏ hàng an toàn.
- `4e5960225`: Hoàn thiện luồng thanh toán và tạo đơn hàng.
- `e37251731`: Hoàn thiện giao diện giỏ hàng và thanh toán.

Qua Git log, em theo dõi được quá trình phát triển theo từng giai đoạn.

Qua Git diff, em xác định những thay đổi mã nguồn và kiểm tra liệu chúng có tác động đến nghiệp vụ đặt hàng hay không.
Để đánh giá hiệu quả làm việc, em cho rằng cần tập trung vào khả năng xử lý đúng nghiệp vụ, bảo vệ dữ liệu, kiểm thử, tự giải thích code và minh bạch lịch sử phát triển.

# IV. Ý 3 – KẾT HỢP CÔNG CỤ KIỂM THỬ 

## 4.1. Công cụ áp dụng

Em có thể kết hợp:

- PHP CLI để kiểm tra cú pháp.
- PHPUnit để kiểm thử các hàm tính toán và kiểm tra dữ liệu.
- Postman để kiểm tra request thanh toán.
- MySQL để đối chiếu kết quả đơn hàng, tồn kho và giỏ hàng.
- Git diff để xác định thay đổi.
- Trình duyệt để kiểm tra luồng thanh toán thực tế.

## 4.2. Test case đề xuất

| Mã | Trường hợp | Kết quả mong đợi |
|---|---|---|
| T01 | Giỏ hàng rỗng | Không tiếp tục thanh toán |
| T02 | Chưa đăng nhập | Chuyển đến trang đăng nhập |
| T03 | Thiếu họ tên | Thông báo lỗi |
| T04 | Email sai định dạng | Thông báo lỗi |
| T05 | Thiếu địa chỉ | Thông báo lỗi |
| T06 | Thiếu CSRF token | HTTP 403 |
| T07 | Đặt quá số lượng tồn kho | Không tạo đơn thành công |
| T08 | Thanh toán hợp lệ | Tạo đơn, lưu chi tiết |
| T09 | Có lỗi trong transaction | Rollback dữ liệu |
| T10 | Người dùng thao tác giỏ của người khác | Từ chối thao tác |
| T11 | Giảm số lượng khi đang bằng 1 | Không giảm xuống 0 |

## 4.3. Ví dụ kiểm thử tự động hàm tính tổng

Phương thức `CheckoutService::summary()` nhận danh sách sản phẩm và trả về tổng số lượng, tổng tiền.

Ví dụ dữ liệu:

- Sản phẩm A: Giá 100.000 đồng, số lượng 2.
- Sản phẩm B: Giá 250.000 đồng, số lượng 1.

Kết quả mong đợi:

- Tổng số lượng: 3.
- Tổng giá trị: 450.000 đồng.

Một bài kiểm thử PHPUnit có thể viết như sau:

```php
public function testCartSummary(): void
{
    $items = [
        [
            'cd_quantity' => 2,
            'sp_price' => 100000
        ],
        [
            'cd_quantity' => 1,
            'sp_price' => 250000
        ]
    ];

    $result = CheckoutService::summary($items);

    $this->assertEquals(3, $result['quantity']);
    $this->assertEquals(450000, $result['total']);
}
```

Khi triển khai bài test, em cần đặt phương thức này trong một class kế thừa `PHPUnit\Framework\TestCase`, nạp CheckoutService và cấu hình PHPUnit phù hợp.

Đây là mã kiểm thử mẫu cần được đưa vào môi trường thực tế và chạy để xác nhận kết quả.

## 4.4. Kiểm thử transaction

Em cần thiết kế hai tình huống:
**Trường hợp thành công:** Có đủ tồn kho, thông tin khách hàng hợp lệ. Sau khi thanh toán, đơn hàng và chi tiết đơn được tạo, tồn kho giảm và giỏ hàng được làm trống.

**Trường hợp thất bại:** Một sản phẩm trong giỏ không đủ tồn kho hoặc quá trình tạo đơn phát sinh lỗi. Em cần kiểm tra hệ thống không lưu dữ liệu một phần và không trừ tồn kho sai.

Để kiểm tra transaction chính xác, em cần đối chiếu trạng thái các bảng trước và sau khi thực hiện trên cơ sở dữ liệu thử nghiệm.

# V. KẾT LUẬN

Qua việc phân tích nghiệp vụ giỏ hàng và thanh toán, em hiểu rằng một hệ thống bán hàng không chỉ cần thực hiện đúng chức năng mà còn phải bảo vệ dữ liệu khi xảy ra lỗi.

AI Agent có thể hỗ trợ phân tích và đề xuất giải pháp, nhưng em cần hiểu transaction, xử lý tồn kho, xác thực người dùng và cách kiểm thử những tình huống phát sinh.

Git log và Git diff giúp em theo dõi quá trình phát triển, còn kiểm thử tự động và kiểm thử tích hợp giúp xác minh tính đúng đắn của chương trình.

Trong thời gian tiếp theo, em cần bổ sung những bài kiểm thử transaction và sử dụng skill thực tế để nâng cao khả năng tự kiểm soát hệ thống.