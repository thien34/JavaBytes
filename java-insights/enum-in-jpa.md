# Sử dụng Enum trong JPA 📚

Date: 19/09/2024
<br>
Author: Hạo Thiên

Enum luôn được khuyên dùng và được sử dụng rất phổ biến trong JPA. Enum giúp kiểm tra giá trị đầu vào của database, đồng thời thể hiện trực quan giá trị của database thông qua Enum ở tầng Java code.

Có 2 cách chính sử dụng Enum trong JPA: `@Enumerated` và `@Convert`

## @Enumerated

```
public enum Status {
    ACTIVE, DEACTIVATE;
}
```
```
@Entity
public class Admin {
  @Id
  private int id;
  private String name;

  @Enumerated
  private Status status;
}
```
Mặc định Enumerated sẽ có EnumType là ORDINAL, lưu theo thứ tự khai báo của Enum và kiểu dữ liệu là Integer (hoặc NUMBER) ACTIVE sẽ là 0, và DEACTIVATE sẽ là 1
```
var admin = new Admin();
admin.setName("John");
admin.setStatus(Status.DEACTIVATE);
```
```
insert 
into
    Admin
    (status, name, id) 
values
    (?, ?, ?)
...
binding parameter [1] as [INTEGER] - [1]
binding parameter [2] as [VARCHAR] - [John]
binding parameter [3] as [INTEGER] - [1]
```
ngược lại khi query
```
select
        admin0_.id as id1_6_,
        admin0_.name as name2_6_,
        admin0_.status as status3_6_
    from
        admin admin0_ 
...
extracted value ([id1_6_] : [INTEGER]) - [1]
extracted value ([name2_6_] : [VARCHAR]) - [John]
extracted value ([status3_6_] : [INTEGER]) - [DEACTIVATE]
```
Nếu để **EnumType** là STRING, giá trị trong database sẽ là tên của Enum và kiểu dữ liệu là text (hoặc VARCHAR)
```
@Entity
public class Admin {
  @Id
  private int id;
  private String name;

  @Enumerated(EnumType.STRING)
  private Status status;
}
```
```
insert 
into
    Admin
    (status, name, id) 
values
    (?, ?, ?)
...
binding parameter [1] as [INTEGER] - [1]
binding parameter [2] as [VARCHAR] - [John]
binding parameter [3] as [VARCHAR] - [DEACTIVATE]
```

## @Convert

Cách này yêu cầu viết thêm một **Converter class** như dưới đây

```
@Converter(autoApply = true)
public class StatusConverter implements AttributeConverter<Status, Integer> {

 public Integer convertToDatabaseColumn(Status value) {
  if (value == null) {
   return null;
  }
  return value.getValue();
 }

 public Status convertToEntityAttribute(Integer value) {
  if (value == null) {
   // Can throw an exception here
   return null;
  }
  return Status.fromValue(value);
 }
}
```

```
@Entity
public class Admin {
  @Id
  private int id;
  private String name;

  @Convert(converter = StatusConverter.class)
  private Status status;
}
```

Annotation `@Convert` ở entity có thể bỏ do ta đã khai báo `@Converter(autoApply = true)` Với cả 2 cách trên, khi thêm data vào database, chỉ thao tác với Enum thuần túy, không cần quan tâm giá trị thực trong database

```
var admin = new Admin();
...
admin.setStatus(Status.ACTIVE);
```
Khi query
```
List<Admin> findByStatus(Status status);

@Query("FROM Admin WHERE status = ?1")
List<Admin> listAllByStatus(Status status);
```
Với JPQL hay HQL, một khi đã sử dụng kiểu Enum, ta sẽ bắt buộc sử dụng Enum trong tham số, không thể dùng giá trị thực trong database. Không thể viết
```
@Query("FROM Admin WHERE status = 0")
List<Admin> listAllActiveStatus();

@Query("FROM Admin WHERE status = ?1")
List<Admin> listAllByStatus(String status); // Kiểu của status phải là Enum
```

## Đánh giá các cách dùng Enum

Sử dụng Enumerated ORDINAL có vẻ ổn, nhưng sẽ phát sinh lỗi không ngờ tới nếu bạn thay đổi thứ tự của Enum

```
public enum Status {
    ACTIVE, BLOCKED, DEACTIVATE;
}
```

Lúc này nếu status trong database là 1 sẽ trả về BLOCKED chứ không còn là DEACTIVATE như cũ

```
select
        admin0_.id as id1_6_,
        admin0_.name as name2_6_,
        admin0_.status as status3_6_
    from
        admin admin0_ 
...
extracted value ([id1_6_] : [INTEGER]) - [1]
extracted value ([name2_6_] : [VARCHAR]) - [John]
extracted value ([status3_6_] : [INTEGER]) - [BLOCKED]
```

Tương tự thì với Enumerated STRING, lỗi sẽ phát sinh nếu bạn thay đổi tên các phần tử của Enum. Dùng @Convert an toàn nhưng lại yêu cầu viết thêm class Converter

## Nên sử dụng Enum như thế nào?
Trong gần 10 năm qua, mình luôn luôn sử dụng kiểu số trong database sau đó mapping với JPA bằng @Convert, vì cho rằng điều đó sẽ giúp tối ưu lưu trữ trong database. Nhưng hóa ra, điều đó là không cần thiết và rất bất tiện.
Giả sử bạn sử dụng kiểu số. Dù bạn chính là người viết ra những Enum đó, thì cũng sẽ cũng lúc bạn tự hỏi những status 1, 2, 3, 4 trong database là gì. Kết quả là bạn lại phải quay lại code để xem trong Enum. Ngoài lập trình viên, nhiều role khác cũng sẽ có nhu cầu truy vấn database. Bạn sẽ lại phải giải thích cho họ status 1, 2, 3, 4 trong database có nghĩa là gì. Ngoài ra, dữ liệu khi được trích xuất và sử dụng ở những ứng dụng khác, Enum trở lên vô dụng ở đó và lại cần phải xây dựng một converter riêng.
Thay vào đó, nếu trong database hiển thị luôn ACTIVE, BLOCKED, DEACTIVATE, sẽ chẳng cần bạn hoặc bất kỳ ai giải thích gì thêm. Việc trao đổi giữa các hệ thống cũng trở nên vô cùng đơn giản vì nó đã quá rõ ràng rồi.
Vấn đề tối ưu dữ liệu cũng không cần quá nghiêm trọng hóa. Bạn chỉ tốn thêm vài byte mỗi record để lưu kiểu chữ thay vì số. Quá nhỏ so với những lợi ích nó mang lại.
Do đó, mình khuyến nghị sử dụng Enumerated STRING. Vừa trực quan lại dễ dàng khai báo vì không yêu cầu viết thêm bất kỳ converter nào khác.
