# Session05 - Bài 1: API danh sách khóa học với Pageable và Sort

Project Java 21, Spring Boot 4.1.1, Spring Data JPA và PostgreSQL. Kế thừa entity, quan hệ và các API tạo dữ liệu của Session04; bổ sung API **GET /courses** trả `ApiResponse<Page<CourseResponse>>`.

Đã kiểm tra: **52/52 JUnit tests pass**, collection Postman chạy qua Newman đạt **13 request/36 assertions, không lỗi**. Xem `docs/KET_QUA_KIEM_TRA.md` và JSON thật tại `docs/courses-page-example.json`.

## Chạy trên máy hiện tại

1. Mở `D:\Code\Springboot\baitapSession05` bằng IntelliJ IDEA, Load Gradle Project.
2. Chọn Project SDK và Gradle JVM là **JDK 21**.
3. Chạy `BaitapSession05Application` với working directory là thư mục gốc project.
4. Gọi `http://localhost:8080/courses?page=0&size=2&sortBy=title&direction=ASC`.

PostgreSQL cần đang chạy tại localhost:5432. Database của ứng dụng là `course_management_session05`. Thông tin đăng nhập của máy nằm trong `config/application-local.properties`, được loại khỏi Git và ZIP.

Chạy bằng PowerShell tại thư mục project:

```powershell
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-21'
.\gradlew.bat bootRun
```

## Tham số API

| Query parameter | Mặc định | Xử lý |
| --- | --- | --- |
| `page` | `0` | Đánh số từ 0; số âm được tự chỉnh thành 0 trong service |
| `size` | `10` | Phải lớn hơn 0; giá trị không hợp lệ trả HTTP 400 |
| `sortBy` | `id` | Có thể bỏ qua hoặc để rỗng; hỗ trợ `id`, `title`, `status`, `instructorId` |
| `direction` | `DESC` | Kiểu `Sort.Direction`; nhận `ASC` hoặc `DESC` |

`instructorId` được ánh xạ thành thuộc tính `instructor.id` của entity. Nếu nhiều bản ghi trùng giá trị sắp xếp, service sắp tiếp theo ID cùng chiều để giữ ranh giới các trang ổn định.

Tên sort không được hỗ trợ, sai kiểu số, direction không hợp lệ hoặc offset vượt giới hạn phân trang của JPA đều nhận **400** với `success=false`, `message` và `data=null`. Trang vượt quá trang cuối vẫn nhận **200** với `content=[]`.

## Các file chính của bài

```text
src/main/java/com/example/baitapsession05/
    BaitapSession05Application.java
    controller/CourseController.java
    service/CourseService.java
    dto/CourseResponse.java
    repository/CourseRepository.java
    response/ApiResponse.java
    exception/BadRequestException.java
    exception/GlobalExceptionHandler.java
src/test/java/com/example/baitapsession05/CoursePaginationTests.java
postman/Session05-Bai1.postman_collection.json
```

### Controller

`@RequestParam` tiếp nhận 4 query parameter với các giá trị mặc định theo đề. Controller gọi service và trả:

```java
ResponseEntity<ApiResponse<Page<CourseResponse>>>
```

### Service

Chữ ký phương thức theo yêu cầu:

```java
Page<CourseResponse> getPagedCourses(int page, int size, String sortBy, Sort.Direction direction)
```

Luồng xử lý:

1. `Math.max(page, 0)` để chuẩn hóa số trang âm.
2. Kiểm tra size, offset và tên trường được phép sắp xếp.
3. `Sort.by(direction, sortField)` mô tả thứ tự.
4. `PageRequest.of(safePage, size, sort)` tạo đối tượng `Pageable`.
5. `courseRepository.findAll(pageable)` phân trang tại database và trả `Page<Course>`.
6. `courses.map(CourseResponse::fromEntity)` chuyển thành `Page<CourseResponse>`, giữ thông tin tổng số bản ghi, tổng số trang và vị trí trang.

Service dùng `@Transactional(readOnly = true)`. Repository vẫn là interface trống kế thừa `JpaRepository<Course, Long>`; không cần viết thêm phương thức `findAll(Pageable)`.

### DTO và response

`CourseResponse` chỉ có `id`, `title`, `status`, `instructorId`. Không đưa entity Instructor hoặc danh sách enrollments vào response, nên JSON không có chu trình quan hệ hai chiều.

Ví dụ rút gọn khi database có 3 khóa học Alpha, Beta, Gamma:

```json
{
  "success": true,
  "message": "Lấy danh sách khóa học thành công",
  "data": {
    "content": [
      { "id": 2, "title": "Alpha", "status": "ACTIVE", "instructorId": 1 },
      { "id": 3, "title": "Beta", "status": "ACTIVE", "instructorId": 1 }
    ],
    "number": 0,
    "size": 2,
    "numberOfElements": 2,
    "totalElements": 3,
    "totalPages": 2
  }
}
```

Response thật có thêm metadata do Spring Data cung cấp như sort, pageable, first, last, empty. ID trong ví dụ chỉ minh họa; ID thực tế phụ thuộc database.

Các API POST/PUT từ Session04 vẫn trả thông điệp với `data=null`; API GET mới đưa Page DTO vào `data`.

## Thử bằng Postman

1. Chạy ứng dụng.
2. Import `postman/Session05-Bai1.postman_collection.json`.
3. Để collection variable `baseUrl=http://localhost:8080`.
4. Chạy collection theo thứ tự: nhóm đầu tạo một giảng viên và ba khóa học Gamma, Alpha, Beta; nhóm sau kiểm tra phân trang/sắp xếp và lỗi đầu vào.

Collection có 13 request và tự lấy ID giảng viên từ header Location. Mỗi lần chạy thêm một giảng viên và ba khóa học, không xóa dữ liệu đang có. Sau lần kiểm tra trên máy hiện tại, database đã có dữ liệu demo để gọi GET trực tiếp.

Một số URL để thử:

```text
GET /courses
GET /courses?page=0&size=2&sortBy=title&direction=ASC
GET /courses?page=1&size=2&sortBy=title&direction=ASC
GET /courses?page=-5&size=2
GET /courses?size=0
GET /courses?sortBy=unknown
GET /courses?direction=SIDEWAYS
```

## Kiểm tra tự động

```powershell
.\gradlew.bat test bootJar
```

Các test dùng profile `test`, kết nối database riêng **course_management_session05_test** và rollback dữ liệu sau mỗi test. Database này đã được tạo trên máy hiện tại. Trên máy khác, chạy `sql/create-test-database.sql` trước khi chạy test; có thể dùng biến `TEST_DB_URL` để thay đổi kết nối test.

`CoursePaginationTests` kiểm tra giá trị mặc định, title ASC/DESC, trang kế tiếp, page âm, sortBy rỗng, trang vượt phạm vi, size/sort/direction sai, offset quá lớn, ánh xạ instructorId, DTO và phân trang khi title trùng nhau. Các test entity, repository, service và API Session04 cũng được giữ lại để kiểm tra hồi quy.

## Dùng bản ZIP trên máy khác

1. Cài Java 21 và PostgreSQL.
2. Trong database `postgres`, chạy `sql/create-database.sql` để tạo database ứng dụng. Nếu muốn chạy test, tạo thêm database bằng `sql/create-test-database.sql`.
3. Sao chép `config/application-local.properties.example` thành `config/application-local.properties`, điền tài khoản PostgreSQL của máy đó.
4. Mở project rồi chạy như hướng dẫn trên. Hibernate tự tạo bảng bằng `ddl-auto=update`.

Có thể cấu hình `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` bằng biến môi trường.

Tài liệu tham khảo: [Spring Data Page.map](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Page.html) và [Spring Data web support](https://docs.spring.io/spring-data/commons/reference/repositories/core-extensions.html).
