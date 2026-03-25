## 1. Tổng quan chức năng

Hệ thống quản lý đào tạo trực tuyến cho phép:

- **Khóa học** mặc định là online.
- **Giảng viên** tạo lớp học từ khóa học, quản lý học viên và theo dõi tiến độ.
- **Học viên** đăng ký lớp, học tập (hoàn thành từng bài) và theo dõi tiến độ của bản thân.
- Tự động phát sinh **chứng chỉ PDF** khi học viên hoàn thành 100% khóa học.
- **Backend**: Spring Boot, JPA/Hibernate, MySQL.
- **Tạo PDF**: sử dụng thư viện **OpenPDF** (hoặc iText) để sinh file và trả về cho client.

---

## 2. Thiết kế cơ sở dữ liệu

### 2.1. Bảng `users`

| Column        | Type         | Description                     |
|---------------|--------------|---------------------------------|
| id            | BIGINT       | PK, auto increment              |
| email         | VARCHAR(255) | unique, đăng nhập               |
| password      | VARCHAR(255) | mã hóa BCrypt                   |
| full_name     | VARCHAR(255) |                                 |
| role          | VARCHAR(50)  | `ADMIN`, `INSTRUCTOR`, `STUDENT`|
| created_at    | DATETIME     |                                 |
| updated_at    | DATETIME     |                                 |

### 2.2. Bảng `courses`

| Column         | Type          | Description                     |
|----------------|---------------|---------------------------------|
| id             | BIGINT        | PK                              |
| title          | VARCHAR(255)  |                                 |
| description    | TEXT          |                                 |
| is_online      | BOOLEAN       | DEFAULT TRUE                    |
| created_by     | BIGINT        | FK → users.id (admin/instructor)|
| created_at     | DATETIME      |                                 |
| updated_at     | DATETIME      |                                 |

### 2.3. Bảng `lessons`

| Column      | Type          | Description                     |
|-------------|---------------|---------------------------------|
| id          | BIGINT        | PK                              |
| course_id   | BIGINT        | FK → courses.id                 |
| title       | VARCHAR(255)  |                                 |
| content     | TEXT          | nội dung bài học (text, link, ...) |
| order_index | INT           | thứ tự hiển thị trong khóa học  |
| type        | VARCHAR(50)   | VIDEO, DOCUMENT, QUIZ, ...      |
| duration    | INT           | thời gian dự kiến (phút)        |
| created_at  | DATETIME      |                                 |
| updated_at  | DATETIME      |                                 |

### 2.4. Bảng `classes`

| Column         | Type          | Description                     |
|----------------|---------------|---------------------------------|
| id             | BIGINT        | PK                              |
| course_id      | BIGINT        | FK → courses.id                 |
| instructor_id  | BIGINT        | FK → users.id (giảng viên phụ trách) |
| name           | VARCHAR(255)  | tên lớp (có thể tự động tạo)    |
| start_date     | DATE          |                                 |
| end_date       | DATE          |                                 |
| status         | VARCHAR(50)   | `OPEN`, `IN_PROGRESS`, `COMPLETED` |
| created_at     | DATETIME      |                                 |
| updated_at     | DATETIME      |                                 |

### 2.5. Bảng `enrollments`

| Column       | Type          | Description                         |
|--------------|---------------|-------------------------------------|
| id           | BIGINT        | PK                                  |
| class_id     | BIGINT        | FK → classes.id                     |
| student_id   | BIGINT        | FK → users.id                       |
| enrolled_at  | DATETIME      |                                     |
| progress     | DECIMAL(5,2)  | phần trăm hoàn thành (0–100)        |
| status       | VARCHAR(50)   | `IN_PROGRESS`, `COMPLETED`          |
| completed_at | DATETIME      | khi progress = 100                  |

### 2.6. Bảng `lesson_progress`

| Column         | Type          | Description                     |
|----------------|---------------|---------------------------------|
| id             | BIGINT        | PK                              |
| enrollment_id  | BIGINT        | FK → enrollments.id             |
| lesson_id      | BIGINT        | FK → lessons.id                 |
| completed      | BOOLEAN       | DEFAULT FALSE                   |
| completed_at   | DATETIME      |                                 |
| last_updated   | DATETIME      |                                 |

### 2.7. Bảng `certificates`

| Column            | Type          | Description                         |
|-------------------|---------------|-------------------------------------|
| id                | BIGINT        | PK                                  |
| enrollment_id     | BIGINT        | FK → enrollments.id, unique         |
| certificate_number| VARCHAR(100)  | mã chứng chỉ duy nhất                |
| issued_at         | DATETIME      |                                     |
| file_path         | VARCHAR(500)  | đường dẫn lưu file PDF (nếu lưu)    |
| pdf_data          | LONGBLOB      | nếu lưu trực tiếp dưới dạng binary  |

> **Ghi chú**: Có thể lưu file PDF lên hệ thống file và lưu đường dẫn, hoặc lưu BLOB. Với yêu cầu trả về PDF, nên lưu file vật lý để tiện tải về.

---

## 3. Mô hình quan hệ

- **User** (1) — (n) **Course** (người tạo khóa học)
- **Course** (1) — (n) **Lesson**
- **Course** (1) — (n) **Class**
- **Class** (n) — (1) **User** (giảng viên)
- **Class** (1) — (n) **Enrollment**
- **Enrollment** (n) — (1) **User** (học viên)
- **Enrollment** (1) — (n) **LessonProgress**
- **LessonProgress** (n) — (1) **Lesson**
- **Enrollment** (1) — (1) **Certificate** (optional)

---

## 4. Công nghệ sử dụng

| Thành phần           | Công nghệ                               |
|----------------------|-----------------------------------------|
| Framework            | Spring Boot 3.x                         |
| ORM                  | Spring Data JPA (Hibernate)             |
| Database             | MySQL 8.x                               |
| Security             | Spring Security + JWT (tuỳ chọn)        |
| Tạo PDF              | OpenPDF (hoặc iText 7)                  |
| API Documentation    | Springdoc OpenAPI (Swagger)             |
| Build Tool           | Maven / Gradle                          |

---

## 5. Cấu trúc package

```
com.example.training
├── TrainingApplication.java
├── config
│   ├── SecurityConfig.java
│   ├── JwtAuthenticationFilter.java (nếu có JWT)
│   └── OpenApiConfig.java
├── controller
│   ├── CourseController.java
│   ├── ClassController.java
│   ├── EnrollmentController.java
│   ├── LessonProgressController.java
│   └── CertificateController.java
├── dto
│   ├── request
│   │   ├── CourseRequest.java
│   │   ├── ClassRequest.java
│   │   ├── LessonRequest.java
│   │   └── CompleteLessonRequest.java
│   └── response
│       ├── CourseResponse.java
│       ├── ClassDetailResponse.java
│       ├── EnrollmentResponse.java
│       └── CertificateResponse.java
├── entity
│   ├── User.java
│   ├── Course.java
│   ├── Lesson.java
│   ├── Class.java
│   ├── Enrollment.java
│   ├── LessonProgress.java
│   └── Certificate.java
├── repository
│   ├── UserRepository.java
│   ├── CourseRepository.java
│   ├── LessonRepository.java
│   ├── ClassRepository.java
│   ├── EnrollmentRepository.java
│   ├── LessonProgressRepository.java
│   └── CertificateRepository.java
├── service
│   ├── CourseService.java
│   ├── ClassService.java
│   ├── EnrollmentService.java
│   ├── LessonProgressService.java
│   └── CertificateService.java
├── util
│   └── PdfGenerator.java
└── exception
    ├── ResourceNotFoundException.java
    └── GlobalExceptionHandler.java
```

---

## 6. Mô tả chi tiết các thành phần chính

### 6.1. Entity

#### User
```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String email;
    private String password;
    private String fullName;
    @Enumerated(EnumType.STRING)
    private Role role;
    // timestamps
}
```

#### Course
```java
@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String description;
    private boolean isOnline = true; // mặc định online
    @ManyToOne
    @JoinColumn(name = "created_by")
    private User createdBy;
    // timestamps
}
```

#### Lesson
```java
@Entity
public class Lesson {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String content;
    private Integer orderIndex;
    private String type; // VIDEO, DOCUMENT, QUIZ
    private Integer duration; // minutes
    @ManyToOne
    @JoinColumn(name = "course_id")
    private Course course;
}
```

#### Class (Lớp học)
```java
@Entity
@Table(name = "classes")
public class Class {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private LocalDate startDate;
    private LocalDate endDate;
    @Enumerated(EnumType.STRING)
    private ClassStatus status; // OPEN, IN_PROGRESS, COMPLETED
    @ManyToOne
    @JoinColumn(name = "course_id")
    private Course course;
    @ManyToOne
    @JoinColumn(name = "instructor_id")
    private User instructor;
}
```

#### Enrollment
```java
@Entity
public class Enrollment {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private LocalDateTime enrolledAt;
    private BigDecimal progress = BigDecimal.ZERO;
    @Enumerated(EnumType.STRING)
    private EnrollmentStatus status; // IN_PROGRESS, COMPLETED
    private LocalDateTime completedAt;

    @ManyToOne
    @JoinColumn(name = "class_id")
    private Class clazz;
    @ManyToOne
    @JoinColumn(name = "student_id")
    private User student;

    @OneToMany(mappedBy = "enrollment", cascade = CascadeType.ALL)
    private List<LessonProgress> lessonProgresses;

    @OneToOne(mappedBy = "enrollment", cascade = CascadeType.ALL)
    private Certificate certificate;
}
```

#### LessonProgress
```java
@Entity
public class LessonProgress {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private boolean completed = false;
    private LocalDateTime completedAt;
    private LocalDateTime lastUpdated;

    @ManyToOne
    @JoinColumn(name = "enrollment_id")
    private Enrollment enrollment;
    @ManyToOne
    @JoinColumn(name = "lesson_id")
    private Lesson lesson;
}
```

#### Certificate
```java
@Entity
public class Certificate {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String certificateNumber;
    private LocalDateTime issuedAt;
    private String filePath; // đường dẫn lưu file PDF

    @OneToOne
    @JoinColumn(name = "enrollment_id", unique = true)
    private Enrollment enrollment;
}
```

### 6.2. Service logic

#### 6.2.1. Xử lý tiến độ học

Khi học viên đánh dấu hoàn thành một bài học:

1. Kiểm tra `LessonProgress` của `enrollment` đó với `lesson` đã tồn tại chưa; nếu chưa thì tạo mới và set `completed = true`, ghi nhận thời gian.
2. Tính lại tổng số bài học đã hoàn thành của `enrollment` so với tổng số bài học của khóa học.
3. Cập nhật `progress = (completedLessons / totalLessons) * 100`.
4. Nếu `progress == 100` và `status` chưa là `COMPLETED`, chuyển `status = COMPLETED`, ghi nhận `completedAt`, và gọi `CertificateService.generateCertificate()`.

#### 6.2.2. Tạo chứng chỉ PDF

- Sử dụng `PdfGenerator` để tạo file PDF với nội dung: tên học viên, tên khóa học, tên lớp, ngày hoàn thành, mã chứng chỉ.
- Lưu file vào thư mục cấu hình (ví dụ: `uploads/certificates/`) và ghi đường dẫn vào bảng `certificates`.
- Mã chứng chỉ có thể tạo bằng `UUID` hoặc công thức như `CERT-{enrollmentId}-{timestamp}`.

#### 6.2.3. Phân quyền cơ bản (không bắt buộc nhưng nên có)

- **ADMIN**: toàn quyền CRUD khóa học, lớp, người dùng.
- **INSTRUCTOR**: chỉ được quản lý khóa học và lớp mình tạo, xem tiến độ học viên trong lớp mình dạy.
- **STUDENT**: chỉ xem lớp đã đăng ký, cập nhật tiến độ học, tải chứng chỉ của mình.

### 6.3. API endpoints

#### Khóa học
| Method | Endpoint                   | Mô tả                       |
|--------|----------------------------|-----------------------------|
| GET    | `/api/courses`             | Danh sách khóa học          |
| POST   | `/api/courses`             | Tạo khóa học mới            |
| GET    | `/api/courses/{id}`        | Chi tiết khóa học + danh sách bài học |
| PUT    | `/api/courses/{id}`        | Cập nhật                    |
| DELETE | `/api/courses/{id}`        | Xóa (nếu chưa có lớp)       |

#### Lớp học
| Method | Endpoint                   | Mô tả                       |
|--------|----------------------------|-----------------------------|
| GET    | `/api/classes`             | Danh sách lớp (filter theo giảng viên, khóa học) |
| POST   | `/api/courses/{courseId}/classes` | Tạo lớp cho khóa học       |
| GET    | `/api/classes/{id}`        | Chi tiết lớp + danh sách học viên (giảng viên xem) |
| POST   | `/api/classes/{id}/enroll` | Học viên đăng ký vào lớp    |

#### Bài học
| Method | Endpoint                               | Mô tả                       |
|--------|----------------------------------------|-----------------------------|
| POST   | `/api/courses/{courseId}/lessons`      | Thêm bài học vào khóa học   |
| PUT    | `/api/lessons/{id}`                    | Cập nhật bài học            |
| DELETE | `/api/lessons/{id}`                    | Xóa bài học                 |

#### Tiến độ học
| Method | Endpoint                                           | Mô tả                             |
|--------|----------------------------------------------------|-----------------------------------|
| POST   | `/api/enrollments/{enrollmentId}/lessons/{lessonId}/complete` | Đánh dấu hoàn thành bài học |
| GET    | `/api/enrollments/{enrollmentId}/progress`         | Xem tiến độ của học viên          |
| GET    | `/api/classes/{classId}/students`                  | Giảng viên xem tiến độ tất cả học viên |

#### Chứng chỉ
| Method | Endpoint                                         | Mô tả                        |
|--------|--------------------------------------------------|------------------------------|
| GET    | `/api/certificates/enrollment/{enrollmentId}`    | Lấy thông tin chứng chỉ      |
| GET    | `/api/certificates/enrollment/{enrollmentId}/download` | Tải file PDF chứng chỉ |

### 6.4. Tạo PDF mẫu

Sử dụng `OpenPDF` (hoặc iText) để tạo:

```java
public byte[] generateCertificatePdf(String studentName, String courseName, String className, String issueDate, String certNumber) throws IOException, DocumentException {
    Document document = new Document();
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    PdfWriter.getInstance(document, baos);
    document.open();

    // Font và nội dung
    Paragraph title = new Paragraph("CHỨNG CHỈ HOÀN THÀNH KHÓA HỌC", FontFactory.getFont(FontFactory.HELVETICA_BOLD, 18));
    title.setAlignment(Element.ALIGN_CENTER);
    document.add(title);
    document.add(Chunk.NEWLINE);

    Paragraph content = new Paragraph("Chứng nhận: " + studentName + " đã hoàn thành khóa học " + courseName + " (lớp " + className + ")", FontFactory.getFont(FontFactory.HELVETICA, 12));
    document.add(content);
    document.add(Chunk.NEWLINE);

    Paragraph info = new Paragraph("Ngày cấp: " + issueDate + "\nMã số: " + certNumber);
    document.add(info);

    document.close();
    return baos.toByteArray();
}
```

Lưu file:
```java
String filePath = "uploads/certificates/cert_" + enrollmentId + ".pdf";
Files.write(Paths.get(filePath), pdfBytes);
```

---

## Bổ sung chức năng bài kiểm tra cuối khóa

Sau khi học viên hoàn thành 100% nội dung bài học, hệ thống sẽ yêu cầu làm một bài kiểm tra. Chỉ khi vượt qua bài kiểm tra (đạt điểm tối thiểu) thì mới được cấp chứng chỉ.

---

## 1. Thay đổi thiết kế cơ sở dữ liệu

### 1.1. Bảng `exams` – bài kiểm tra cuối khóa

| Column       | Type          | Description                           |
|--------------|---------------|---------------------------------------|
| id           | BIGINT        | PK                                    |
| course_id    | BIGINT        | FK → courses.id                       |
| title        | VARCHAR(255)  | Ví dụ: "Kiểm tra cuối khóa"           |
| description  | TEXT          | Hướng dẫn                             |
| pass_score   | DECIMAL(5,2)  | Điểm tối thiểu để đạt (ví dụ 5.0)     |
| time_limit   | INT           | Thời gian làm bài (phút), NULL nếu không giới hạn |
| max_attempts | INT           | Số lần làm bài tối đa, mặc định 1     |
| created_at   | DATETIME      |                                       |
| updated_at   | DATETIME      |                                       |

**Ràng buộc**: mỗi khóa học có thể có 0 hoặc 1 bài kiểm tra cuối khóa.

### 1.2. Bảng `exam_questions`

| Column        | Type          | Description                        |
|---------------|---------------|------------------------------------|
| id            | BIGINT        | PK                                 |
| exam_id       | BIGINT        | FK → exams.id                      |
| question_text | TEXT          | Nội dung câu hỏi                   |
| question_type | VARCHAR(50)   | `SINGLE_CHOICE`, `MULTIPLE_CHOICE`, `ESSAY` |
| points        | DECIMAL(5,2)  | Điểm tối đa cho câu hỏi            |
| order_index   | INT           | Thứ tự hiển thị                    |
| created_at    | DATETIME      |                                    |

### 1.3. Bảng `exam_options` – lựa chọn cho câu hỏi trắc nghiệm

| Column        | Type          | Description                        |
|---------------|---------------|------------------------------------|
| id            | BIGINT        | PK                                 |
| question_id   | BIGINT        | FK → exam_questions.id             |
| option_text   | TEXT          | Nội dung lựa chọn                  |
| is_correct    | BOOLEAN       | Đúng/sai                           |
| order_index   | INT           |                                    |

### 1.4. Bảng `exam_attempts` – lần làm bài của học viên

| Column         | Type          | Description                           |
|----------------|---------------|---------------------------------------|
| id             | BIGINT        | PK                                    |
| enrollment_id  | BIGINT        | FK → enrollments.id                   |
| attempt_number | INT           | Lần thứ mấy (1,2,3...)                |
| started_at     | DATETIME      | Bắt đầu làm bài                       |
| submitted_at   | DATETIME      | Nộp bài                               |
| score          | DECIMAL(5,2)  | Tổng điểm đạt được                    |
| passed         | BOOLEAN       | Có đạt yêu cầu hay không              |
| status         | VARCHAR(50)   | `IN_PROGRESS`, `SUBMITTED`            |

### 1.5. Bảng `exam_answers` – câu trả lời chi tiết cho từng câu hỏi

| Column            | Type          | Description                        |
|-------------------|---------------|------------------------------------|
| id                | BIGINT        | PK                                 |
| attempt_id        | BIGINT        | FK → exam_attempts.id              |
| question_id       | BIGINT        | FK → exam_questions.id             |
| selected_option_id| BIGINT        | FK → exam_options.id (cho trắc nghiệm) |
| essay_answer      | TEXT          | Câu trả lời tự luận                |
| is_correct        | BOOLEAN       | Đúng/sai (dùng cho tự động chấm)   |
| points_earned      | DECIMAL(5,2)  | Điểm đạt được cho câu hỏi này      |

**Ghi chú**: Với câu hỏi tự luận, giảng viên sẽ chấm thủ công và cập nhật `is_correct`, `points_earned`. Có thể tích hợp cơ chế chấm tự động cho trắc nghiệm.

---

## 2. Điều chỉnh bảng `enrollments`

Thêm trạng thái mới để phản ánh giai đoạn chờ kiểm tra:

| Column       | Type          | Description                                     |
|--------------|---------------|-------------------------------------------------|
| status       | VARCHAR(50)   | `IN_PROGRESS`, `PENDING_EXAM`, `COMPLETED`      |

- **IN_PROGRESS**: đang học bài học.
- **PENDING_EXAM**: đã học xong bài học (progress = 100%) nhưng chưa làm bài kiểm tra hoặc chưa đạt.
- **COMPLETED**: đã hoàn thành cả bài học và bài kiểm tra, đã nhận chứng chỉ.

Sau khi học viên hoàn thành 100% bài học, hệ thống chuyển `status` thành `PENDING_EXAM`. Chỉ khi làm bài kiểm tra và đạt yêu cầu, mới chuyển thành `COMPLETED` và tạo chứng chỉ.

---

## 3. Logic nghiệp vụ bổ sung

### 3.1. Sau khi hoàn thành tất cả bài học

Khi `LessonProgressService` phát hiện `progress = 100`:

- Cập nhật `Enrollment.progress = 100`.
- Nếu khóa học không có bài kiểm tra cuối khóa (hoặc `exams` không tồn tại) thì chuyển `status = COMPLETED`, tạo chứng chỉ ngay.
- Nếu có bài kiểm tra, chuyển `status = PENDING_EXAM`, không tạo chứng chỉ.

### 3.2. Học viên làm bài kiểm tra

- API `POST /api/enrollments/{enrollmentId}/exam/start` – khởi tạo một `ExamAttempt` mới, trả về danh sách câu hỏi (ẩn đáp án).
- API `POST /api/enrollments/{enrollmentId}/exam/submit` – nhận câu trả lời, tính điểm (tự động chấm trắc nghiệm), lưu vào `exam_answers`, cập nhật `score` và `passed`.
- Nếu `passed = true`:
    - Chuyển `Enrollment.status = COMPLETED`
    - Tạo chứng chỉ (`Certificate`)
- Nếu `passed = false`:
    - Nếu số lần làm chưa vượt quá `max_attempts`, vẫn giữ `PENDING_EXAM`, cho phép làm lại.
    - Nếu hết số lần, có thể giữ trạng thái `PENDING_EXAM` và không cho làm tiếp, hoặc chuyển sang `FAILED` (tuỳ nghiệp vụ).

### 3.3. Giảng viên xem kết quả bài kiểm tra

- API `GET /api/classes/{classId}/exam-results` – xem danh sách học viên và điểm bài kiểm tra của lớp mình dạy.
- Đối với câu hỏi tự luận, giảng viên có API `PUT /api/exam-answers/{answerId}/grade` để chấm điểm và cập nhật `points_earned`. Sau đó, hệ thống tính lại tổng điểm của lần làm bài và cập nhật `score` và `passed`.

---

## 4. Bổ sung các entity Java

### Exam.java
```java
@Entity
public class Exam {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String description;
    private BigDecimal passScore;
    private Integer timeLimit;
    private Integer maxAttempts = 1;

    @ManyToOne
    @JoinColumn(name = "course_id")
    private Course course;

    @OneToMany(mappedBy = "exam", cascade = CascadeType.ALL)
    private List<ExamQuestion> questions;
}
```

### ExamQuestion.java
```java
@Entity
public class ExamQuestion {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String questionText;
    private String questionType; // SINGLE_CHOICE, MULTIPLE_CHOICE, ESSAY
    private BigDecimal points;
    private Integer orderIndex;

    @ManyToOne
    @JoinColumn(name = "exam_id")
    private Exam exam;

    @OneToMany(mappedBy = "question", cascade = CascadeType.ALL)
    private List<ExamOption> options;
}
```

### ExamAttempt.java
```java
@Entity
public class ExamAttempt {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private Integer attemptNumber;
    private LocalDateTime startedAt;
    private LocalDateTime submittedAt;
    private BigDecimal score;
    private Boolean passed;
    private String status; // IN_PROGRESS, SUBMITTED

    @ManyToOne
    @JoinColumn(name = "enrollment_id")
    private Enrollment enrollment;
}
```

### ExamAnswer.java
```java
@Entity
public class ExamAnswer {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private BigDecimal pointsEarned;
    private Boolean isCorrect;
    private String essayAnswer;

    @ManyToOne
    @JoinColumn(name = "attempt_id")
    private ExamAttempt attempt;

    @ManyToOne
    @JoinColumn(name = "question_id")
    private ExamQuestion question;

    @ManyToOne
    @JoinColumn(name = "selected_option_id")
    private ExamOption selectedOption;
}
```

---

## 5. Điều chỉnh service

### 5.1. LessonProgressService – sau khi hoàn thành 100%
```java
public void completeLesson(...) {
    // ... tính progress
    if (newProgress.compareTo(BigDecimal.valueOf(100)) == 0) {
        Enrollment enrollment = enrollmentRepository.findById(enrollmentId);
        Course course = enrollment.getClazz().getCourse();
        Optional<Exam> exam = examRepository.findByCourseId(course.getId());
        if (exam.isPresent()) {
            enrollment.setStatus(EnrollmentStatus.PENDING_EXAM);
        } else {
            enrollment.setStatus(EnrollmentStatus.COMPLETED);
            certificateService.generateCertificate(enrollment);
        }
        enrollmentRepository.save(enrollment);
    }
}
```

### 5.2. ExamService – xử lý làm bài
```java
public ExamAttempt startExam(Long enrollmentId) {
    Enrollment enrollment = enrollmentRepository.findById(enrollmentId);
    // Kiểm tra enrollment status == PENDING_EXAM
    // Kiểm tra số lần đã làm (count attempts) < exam.maxAttempts
    ExamAttempt attempt = new ExamAttempt();
    attempt.setAttemptNumber(attemptCount + 1);
    attempt.setStartedAt(LocalDateTime.now());
    attempt.setStatus("IN_PROGRESS");
    attempt.setEnrollment(enrollment);
    return examAttemptRepository.save(attempt);
}

public void submitExam(Long attemptId, List<AnswerDto> answers) {
    ExamAttempt attempt = examAttemptRepository.findById(attemptId);
    // tính điểm tự động cho trắc nghiệm
    BigDecimal totalScore = BigDecimal.ZERO;
    for (AnswerDto ans : answers) {
        // lưu vào exam_answers, tính điểm
    }
    attempt.setScore(totalScore);
    attempt.setPassed(totalScore.compareTo(attempt.getExam().getPassScore()) >= 0);
    attempt.setSubmittedAt(LocalDateTime.now());
    attempt.setStatus("SUBMITTED");
    examAttemptRepository.save(attempt);

    if (attempt.getPassed()) {
        Enrollment enrollment = attempt.getEnrollment();
        enrollment.setStatus(EnrollmentStatus.COMPLETED);
        enrollment.setCompletedAt(LocalDateTime.now());
        enrollmentRepository.save(enrollment);
        certificateService.generateCertificate(enrollment);
    } else {
        // nếu còn lượt, vẫn giữ PENDING_EXAM
        // nếu hết lượt, có thể chuyển sang FAILED hoặc không cho làm nữa
    }
}
```

---

## 6. API bổ sung

| Method | Endpoint                                 | Mô tả                                         |
|--------|------------------------------------------|-----------------------------------------------|
| GET    | `/api/enrollments/{id}/exam`             | Lấy thông tin bài kiểm tra của khóa học (nếu có) |
| POST   | `/api/enrollments/{id}/exam/start`       | Bắt đầu làm bài kiểm tra, trả về danh sách câu hỏi |
| POST   | `/api/exam-attempts/{attemptId}/submit`  | Nộp bài kiểm tra                              |
| GET    | `/api/exam-attempts/{attemptId}/result`  | Xem kết quả chi tiết của lần làm              |
| GET    | `/api/classes/{classId}/exam-results`    | Giảng viên xem kết quả tất cả học viên trong lớp |

---

## 7. Ghi chú về tự luận

- Nếu sử dụng câu hỏi tự luận, cần có thêm API cho giảng viên chấm bài.
- Hệ thống có thể gửi thông báo cho giảng viên khi có bài tự luận cần chấm.
- Sau khi giảng viên chấm xong, tổng điểm được tính lại và nếu đạt yêu cầu, tự động chuyển trạng thái và cấp chứng chỉ.

---

## 8. Luồng tổng thể (cập nhật)

1. Học viên đăng ký lớp, bắt đầu học các bài học.
2. Khi hoàn thành 100% bài học:
   - Nếu khóa học **không** có bài kiểm tra cuối khóa → cấp chứng chỉ ngay.
   - Nếu **có** bài kiểm tra → chuyển sang trạng thái `PENDING_EXAM`.
3. Học viên làm bài kiểm tra (tối đa số lần cho phép).
4. Nếu đạt điểm yêu cầu:
   - Chuyển `COMPLETED`, tạo chứng chỉ PDF, lưu vào database/file.
5. Nếu không đạt:
   - Cho phép làm lại nếu còn lượt.
   - Nếu hết lượt, có thể đánh dấu không đạt, không cấp chứng chỉ (tuỳ nghiệp vụ).

---

Với các bổ sung này, hệ thống quản lý đào tạo đáp ứng đầy đủ yêu cầu: khóa học online, theo dõi tiến độ, cấp chứng chỉ PDF **có kèm bài kiểm tra cuối khóa**.
