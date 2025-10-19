---
title: "10 Lỗi Phổ Biến Mà Lập Trình Viên JavaScript Mới Thường Mắc Phải"
meta_title: ""
description: "this is meta description"
date: 2024-12-09T05:00:00Z
image: "/images/gallery/10.jpg"
categories: ["Software", "JavaScript"]
author: "Lê Anh Tiến"
tags: ["software", "javascript" ]
draft: false
---

JavaScript là ngôn ngữ “dễ học nhưng khó giỏi”. Cứ tưởng chỉ cần `console.log("Hello World")` là đời sang trang, ai ngờ bước sang dòng thứ hai là “tội đồ” của runtime error. Dưới đây là 10 lỗi phổ biến mà lập trình viên JavaScript mới thường mắc phải — cùng cách tránh để khỏi rơi vào hố sâu `undefined`.

---

# 10 Lỗi Phổ Biến Mà Lập Trình Viên JavaScript Mới Thường Mắc Phải

---

JavaScript là ngôn ngữ “dễ học nhưng khó giỏi”. Cứ tưởng chỉ cần `console.log("Hello World")` là đời sang trang, ai ngờ bước sang dòng thứ hai là “tội đồ” của runtime error. Dưới đây là 10 lỗi phổ biến mà lập trình viên JavaScript mới thường mắc phải — cùng cách tránh để khỏi rơi vào hố sâu `undefined`.

---

## 1. Quên `let`, `const`, và để `var` tung hoành

Không gì đáng sợ hơn việc dùng `var` trong năm 2025. Nó khiến biến bị **hoisting**, nghĩa là “chưa khai báo đã tồn tại”.  
Thế là khi debug, bạn nhìn thấy biến “vô tri” nhảy múa trong phạm vi toàn cầu.  
> **Giải pháp:** Dùng `const` mặc định, `let` khi cần gán lại. Và quên `var` đi như quên người yêu cũ.

---

## 2. So sánh bằng `==` thay vì `===`

`==` sẽ tự ép kiểu (coercion), khiến `"5" == 5` trả về `true`. Rồi chị em ngồi tự hỏi “vì sao chương trình sai mà không lỗi?”.  
> **Giải pháp:** Hãy luôn dùng `===` và `!==` để so sánh giá trị lẫn kiểu dữ liệu. Đừng để JavaScript “sáng tạo giúp bạn”.

---

## 3. Không hiểu `this` là ai

`this` trong JavaScript như người yêu hay giận dỗi — lúc ở trong object, lúc lại biến mất trong callback.  
> **Giải pháp:** Dùng arrow function (`=>`) khi muốn giữ ngữ cảnh `this` của cha nó, hoặc `.bind()` nếu cần ràng buộc lại.  
Đừng giả định `this` là bạn hiểu rõ — nó thay đổi còn nhanh hơn deadline.

---

## 4. Gọi hàm bất đồng bộ mà quên `await`

Ai cũng từng làm thế này: gọi API bằng `fetch()` rồi `console.log()` ngay kết quả. Kết quả? Một `Promise { pending }` như bức thư chưa mở.  
> **Giải pháp:** Dùng `async/await`, hoặc `then()` nếu chưa quen. Và nhớ thêm `try...catch` để bắt lỗi — chứ đừng bắt đầu hoảng loạn khi API trả về lỗi 404.

---

## 5. Dùng `for` sai cách với mảng

Nhiều người vẫn dùng `for (var i in arr)` để duyệt mảng, trong khi `for...in` là để duyệt **thuộc tính** chứ không phải phần tử.  
> **Giải pháp:** Dùng `for...of`, hoặc cao cấp hơn là `arr.forEach()` hay `arr.map()`. Đừng biến mảng thành đối tượng hỗn loạn.

---

## 6. Lạm dụng `null` và `undefined`

Cứ nghĩ `undefined` và `null` là anh em song sinh, nhưng không — một thằng “chưa được gán”, một thằng “được gán là trống rỗng”.  
> **Giải pháp:** Khi không có giá trị, hãy dùng `null` có chủ đích. Và đừng quên kiểm tra với `== null` (bắt được cả hai) hoặc `=== null` (nếu bạn chắc chắn).

---

## 7. Không nắm rõ phạm vi biến (scope)

Nhiều người mới “bẻ cổ” logic của mình bằng cách khai báo biến bên trong block rồi cố dùng ở ngoài.  
> **Giải pháp:** Học kỹ khái niệm **block scope** và **function scope**. `let` và `const` chỉ tồn tại trong block `{}`.  
Đừng để bug xuất hiện chỉ vì bạn khai báo biến “ở trong nhà nhưng gọi ngoài đường”.

---

## 8. Tưởng JSON là object (và ngược lại)

`JSON` chỉ là chuỗi, không phải object. Nhưng nhiều người cố truy cập `json.key` rồi thắc mắc tại sao “nó không chịu nói chuyện với mình”.  
> **Giải pháp:** Dùng `JSON.parse()` để chuyển chuỗi JSON thành object, và `JSON.stringify()` để làm ngược lại. Rất đơn giản, nhưng ai cũng từng quên.

---

## 9. Không bắt lỗi với `try...catch`

Cứ để code tự chạy rồi cầu nguyện “mong là không lỗi” là phong cách nguy hiểm nhất.  
> **Giải pháp:** Bao quanh đoạn code nghi ngờ bằng `try...catch`, và log lỗi ra rõ ràng.  
Không ai hoàn hảo, nhưng ai biết bắt lỗi thì đỡ “đổ thừa số phận”.

---

## 10. Không đọc thông báo lỗi (hoặc lười console.log)

Có người xem `console.log` là “bùa trừ tà”, có người lại bỏ qua hoàn toàn. Cả hai đều nguy hiểm.  
> **Giải pháp:** Log đúng chỗ, đọc kỹ lỗi, và hiểu nguyên nhân. Mỗi dòng lỗi là một cơ hội để bạn hiểu JavaScript hơn — hoặc ít nhất, hiểu vì sao code của mình tan nát.

---

### Kết

JavaScript không khó, chỉ là nó “thông minh theo cách riêng”. Càng hiểu rõ các lỗi cơ bản, bạn càng đỡ “ăn hành” khi code.
Và nhớ: **một lập trình viên giỏi không phải là người không mắc lỗi, mà là người sửa lỗi nhanh hơn người khác**.

---

*Viết bởi Kaleidoscope — người từng `console.log` cả cuộc đời mình.*

---