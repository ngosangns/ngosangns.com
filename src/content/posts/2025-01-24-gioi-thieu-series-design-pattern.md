---
published: 2025-01-24
title: Giới thiệu series về Design Pattern của mình
tags: ["Design Pattern", "TypeScript", "Software Architecture", "OOP", "Programming", "Gang of Four", "Best Practices", "Series"]
category: "Software Engineering"
---
Tham khảo từ nhiều nguồn. Chủ yếu là các bài viết trên Viblo.

Ngôn ngữ diễn tả: Typescript

Design Pattern ban đầu đơn giản là một khái niệm kiến trúc do Christopher Alexander gây dựng. Lần đầu tiên được ứng dụng vào phần mềm vào năm 1987 bởi Kent Beck and Ward Cunningham. Hai ông trình bày ý tưởng của mình trong một hội nghị. Sau đó Design Pattern trở thành khái niệm phổ biến và tiếp tuc phát triển cho đến ngày nay. Design Pattern lần đầu tiên được tổng hợp thành cuốn sách hoàn chỉnh vào năm 1995 trong cuốn `Design Patterns: Elements of Reusable Object-Oriented Software`.

Trong phát triển phần mềm, Design Pattern là giải pháp thiết kế mã để tái sử dụng chúng. Design Pattern được sử dụng mạnh mẽ nhất trong OOP qua Object và Class. Có rất nhiều Design Pattern, theo tổng kết của Gang of Four, hiện tại có hơn 250 mẫu đang được sử dụng trong phát triển phần mềm, tuy nhiên bạn không cần sử dụng hết 250 mẫu này, một lập trình viên giỏi Desgin Pattern chỉ cần sử dụng thành thạo khoảng 26 mẫu.

Bản chất Design Pattern là tách các thành phần khó mở rộng ra thành các đối tượng khác nhau cùng với việc cấu trúc lại một cách hợp lí để dễ dàng mở rộng trong tương lai.

*Lưu ý: Design Pattern không phải thuật toán, không phải một component.*

| [Creational](./2025-01-24-creational-design-pattern) | [Structural](./2025-01-24-structural-design-pattern) | [Behavioral](./2025-01-24-behavioral-design-pattern) |
| --- | --- | --- |
| Singleton | Adapter/ Wrapper | Chain of Responsibility |
| Factory | Bridge | Command |
| Method Factory | Composite | Mediator |
| Abstract Factory | Decorator | Memento |
| Builder | Facade | Observer |
| Object Pool | Proxy | Strategy |
| Prototype | Flyweight | Visitor |
| Dependency Injection | Delegation | State |
| | Entity-Attribute-Value (EAV) | Repository |

Một số Pattern mình đã bỏ đi:

- **Template Method**: Đơn giản là mở rộng theo cách kế thừa khi đã clean code
- **Null Object**: Do class con triển khai từ class cha không chỉ triển khai khung xương mà còn mở rộng thêm nhiều phương thức khác, do đó null object bắt buộc phải tạo ra cho riêng từ loại class, gây khó mở rộng hơn
