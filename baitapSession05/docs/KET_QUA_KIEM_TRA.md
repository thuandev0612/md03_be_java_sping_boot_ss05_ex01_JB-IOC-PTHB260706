# Kết quả kiểm tra Session05 - Bài 1

Ngày kiểm tra: 25/10/2026, giờ Việt Nam.

## Gradle và PostgreSQL

`gradlew.bat test bootJar` với JDK 21: **BUILD SUCCESSFUL**.

| Bộ test | Số test | Lỗi/thất bại |
| --- | --- | --- |
| CoursePaginationTests | 20 | 0 |
| CourseEnrollmentApiTests | 14 | 0 |
| EntityMappingTests | 14 | 0 |
| RepositoryServiceTests | 4 | 0 |
| Tổng | 52 | 0 |

Test dùng database `course_management_session05_test`, rollback dữ liệu sau mỗi test.

## Kiểm tra HTTP bằng collection Postman

Đã chạy file JAR thật trên localhost và chạy collection bằng Newman 6.2.3: **13 request, 36 assertions, 0 lỗi**.

- Tạo giảng viên và ba khóa học Gamma, Alpha, Beta: HTTP 201.
- GET page=0, size=2, sortBy=title, direction=ASC: HTTP 200; content là **Alpha, Beta**; totalElements=3, totalPages=2.
- GET page=1 với cùng sort: HTTP 200, content gồm Gamma.
- Bỏ query parameter: page=0, size=10, thứ tự ID giảm dần.
- Title DESC: khóa học theo tên giảm dần.
- Page âm: HTTP 200, page được điều chỉnh về 0.
- Trang vượt phạm vi: HTTP 200, content rỗng.
- Size=0, sortBy không tồn tại, direction sai: HTTP 400, lỗi được bọc trong ApiResponse.

Đã đối chiếu thêm bằng SQL trực tiếp: `ORDER BY title ASC, id ASC LIMIT 2` trả Alpha, Beta đúng như API.

JSON thật đã lưu ở `courses-page-example.json`. Nội dung khóa học chỉ có id, title, status và instructorId, không chứa entity quan hệ hai chiều.

Database ứng dụng `course_management_session05` hiện giữ lại một giảng viên và ba khóa học demo. Ứng dụng đã được dừng sau kiểm tra; chạy lại `BaitapSession05Application` để thử Postman tại cổng mặc định 8080.
