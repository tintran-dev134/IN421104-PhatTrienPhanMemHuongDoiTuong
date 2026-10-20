# Báo cáo phân tích vi phạm SOLID trong ClubManager

- **Môn học:** Phát triển phần mềm hướng đối tượng (IN4211)
- **Học phần:** Chương 2 — Refactoring EduHub với SOLID và Design Patterns
- **Nhiệm vụ 1:** Phân tích ít nhất 4 vi phạm SOLID trong `ClubManager`
- **Đối tượng phân tích:** Lớp `ClubManager.cs` (God class trong thư mục starter)

---

## 1. Vi phạm Single Responsibility Principle (SRP — Đơn trách nhiệm)

* **Vị trí dòng:** Toàn bộ class `ClubManager` (dòng 28–117), tiêu biểu nhất tại method `CreateClub` (dòng 35–64) và `AddMemberToClub` (dòng 67–84).
* **Mô tả vi phạm:** 
  Class `ClubManager` và method `CreateClub` cùng lúc gánh vác tới 4 trách nhiệm hoàn toàn khác nhau:
  1. Kiểm tra tính hợp lệ của dữ liệu (Validation).
  2. Tạo đối tượng và lưu trữ dữ liệu vào danh sách nội bộ (`_clubs.Add(club)`).
  3. Ghi log trực tiếp ra file văn bản trên ổ đĩa (`File.AppendAllText`).
  4. Lựa chọn kênh và gửi thông báo cho người dùng (`Console.WriteLine`).
* **Hậu quả:**
  - Vi phạm quy tắc *"Một lớp chỉ nên có một lý do duy nhất để thay đổi"*.
  - Bất kỳ thay đổi nào về cơ chế lưu trữ (chuyển sang Database), thay đổi cách ghi nhật ký, hay đổi dịch vụ gửi mail đều buộc phải sửa vào `ClubManager`.
  - Class trở thành "God Class" cồng kềnh, khó bảo trì, dễ sinh lỗi dây chuyền và không thể viết unit test riêng cho từng chức năng.
* **Đề xuất cách sửa (Refactor):**
  - Tách các trách nhiệm thành các thành phần độc lập:
    + Nghiệp vụ CLB chuyển sang `ClubService`.
    + Việc lưu trữ đưa vào Repository (`IClubRepository`).
    + Việc ghi nhật ký chuyển cho `IAppLogger`.
    + Việc gửi thông báo chuyển cho `INotificationService`.

---

## 2. Vi phạm Open/Closed Principle (OCP — Đóng/Mở)

* **Vị trí dòng:** Dòng 52–63 trong `CreateClub` và dòng 82–83 trong `AddMemberToClub`.
* **Đoạn code vi phạm:**
  ```csharp
  if (presidentEmail.Contains("@gmail.com"))
      Console.WriteLine($"Sending Gmail notification to {presidentEmail}");
  else if (presidentEmail.Contains("@yahoo.com"))
      Console.WriteLine($"Sending Yahoo notification to {presidentEmail}");
  else
      Console.WriteLine($"Sending generic email to {presidentEmail}");
  ```
* **Mô tả vi phạm:**
  Việc gửi thông báo bị "buộc cứng" bằng chuỗi `if / else if / else` dựa theo chuỗi đuôi email. Trong `AddMemberToClub` thì cố định gửi cả Email lẫn Slack.
* **Hậu quả:**
  - Vi phạm quy tắc *"Mở cho việc mở rộng, đóng cho việc sửa đổi"*.
  - Mỗi khi hệ thống muốn hỗ trợ thêm kênh thông báo mới (như SMS, Slack, Telegram, Zalo), lập trình viên buộc phải mở file `ClubManager.cs` ra để viết thêm nhánh `else if`.
  - Việc sửa đổi trực tiếp vào code đang hoạt động rất dễ gây lỗi (bug) ngoài ý muốn cho các kênh cũ.
* **Đề xuất cách sửa (Refactor):**
  - Áp dụng **Factory Pattern**:
    + Tạo interface chung `INotificationService` với các class cài đặt riêng (`EmailNotificationService`, `SmsNotificationService`, `SlackNotificationService`).
    + Dùng `NotificationServiceFactory` để khởi tạo kênh thông báo theo `NotificationChannel`. Khi thêm kênh mới chỉ cần tạo class mới và thêm nhánh trong Factory, không đụng đến service nghiệp vụ.

---

## 3. Vi phạm Dependency Inversion Principle (DIP — Đảo ngược phụ thuộc)

* **Vị trí dòng:** Dòng 48, dòng 79, dòng 94, dòng 111 (`File.AppendAllText`), kết hợp dòng 44, 75, 93 (`new Club`, `new Member`, `new Event`).
* **Đoạn code vi phạm:**
  ```csharp
  File.AppendAllText(_logFilePath, $"[{DateTime.Now}] Club created: {name}\n");
  ```
* **Mô tả vi phạm:**
  Module cấp cao chứa logic nghiệp vụ (`ClubManager`) lại phụ thuộc trực tiếp vào chi tiết cấp thấp của hệ điều hành: gọi thẳng lớp tĩnh `System.IO.File` và file cố định `"club-log.txt"`.
* **Hậu quả:**
  - Vi phạm quy tắc *"Module cấp cao không được phụ thuộc vào module cấp thấp; cả hai phải phụ thuộc vào abstraction (interface)"*.
  - **Không thể viết Unit Test tự động:** Mỗi lần chạy test sẽ tạo ra file vật lý thật trên ổ cứng, làm chậm quá trình test và gây lỗi nếu máy chạy test không có quyền ghi ổ đĩa.
  - Không thể linh hoạt thay đổi đích ghi log (ví dụ ghi ra Console, đẩy lên Cloud, hay tắt log) mà không sửa code nghiệp vụ.
* **Đề xuất cách sửa (Refactor):**
  - Định nghĩa interface trừu tượng `IAppLogger` (với các hàm `Info`, `Error`).
  - Cung cấp các hiện thực: `ConsoleLogger`, `FileLogger`.
  - Truyền `IAppLogger` vào qua Constructor Injection. Lớp nghiệp vụ chỉ gọi `_logger.Info(...)`.

---

## 4. Vi phạm OCP & SRP trong Xuất báo cáo (Report)

* **Vị trí dòng:** Dòng 103–113 trong method `PrintClubReport`.
* **Đoạn code vi phạm:**
  ```csharp
  public void PrintClubReport()
  {
      foreach (var club in _clubs)
      {
          var memberCount = _members.Count(m => m.ClubId == club.Id);
          Console.WriteLine($"Club: {club.Name} | Members: {memberCount}");
          File.AppendAllText(_logFilePath, $"Report generated for {club.Name}\n");
      }
  }
  ```
* **Mô tả vi phạm:**
  Method `PrintClubReport` vừa làm nhiệm vụ tổng hợp thống kê số liệu, vừa tự format in trực tiếp ra màn hình Console (`Console.WriteLine`).
* **Hậu quả:**
  - Format báo cáo bị trói chặt với màn hình Console. Khi cần xuất báo cáo ra file CSV, file JSON, HTML hoặc PDF để nộp, bắt buộc phải viết thêm nhiều method mới hoặc can thiệp sửa code cũ.
* **Đề xuất cách sửa (Refactor):**
  - Áp dụng **Strategy Pattern**:
    + Tách logic xuất báo cáo thành `ReportService`.
    + Định nghĩa interface `IReportStrategy` với phương thức `Render(stats)`.
    + Tạo các chiến lược định dạng riêng biệt: `ConsoleReportStrategy`, `CsvReportStrategy`. Khi cần đổi định dạng báo cáo chỉ cần truyền Strategy tương ứng lúc chạy.

---

## 5. Vi phạm OCP trong Thông báo sự kiện (Event Notification)

* **Vị trí dòng:** Dòng 88–99 trong method `CreateEvent`.
* **Mô tả vi phạm:**
  Method `CreateEvent` sau khi tạo sự kiện thì tự duyệt vòng lặp `foreach` gửi email cho toàn bộ thành viên.
* **Hậu quả:**
  - Logic tạo sự kiện bị gắn chặt với hành vi gửi email. Nếu sau này có thêm các nhu cầu: ghi log khi tạo sự kiện, gửi SMS, đẩy thông báo mobile app thì đều phải sửa `CreateEvent`.
* **Đề xuất cách sửa (Refactor):**
  - Áp dụng **Observer Pattern**:
    + Định nghĩa interface `IEventObserver` với phương thức `OnEventCreated(ev, club)`.
    + `EventService` quản lý danh sách các observer (`Subscribe`). Khi tạo sự kiện thành công, nó phát tín hiệu thông báo cho tất cả observer (`LogNotifier`, `EmailNotifier`) mà không cần biết observer làm việc gì.
