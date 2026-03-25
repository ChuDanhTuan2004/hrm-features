Dưới đây là thiết kế chi tiết cho **chức năng Quản lý tuyển dụng** trong hệ thống HRMS, sử dụng Spring Boot (Backend) và MySQL (Database). Thiết kế bao gồm: cấu trúc bảng, quan hệ giữa các bảng, các endpoint RESTful, và luồng xử lý chính.

---

## 1. Mô hình cơ sở dữ liệu

### 1.1. Bảng `recruitment_post` – Tin tuyển dụng

| Tên cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---------|--------------|-----------|-------|
| id | BIGINT | PK, AUTO_INCREMENT | Mã tin tuyển dụng |
| title | VARCHAR(255) | NOT NULL | Tiêu đề công việc |
| department_id | BIGINT | FK → department(id) | Phòng ban cần tuyển |
| position | VARCHAR(100) | NOT NULL | Vị trí (Nhân viên, Trưởng phòng, ...) |
| quantity | INT | DEFAULT 1 | Số lượng cần tuyển |
| description | TEXT | | Mô tả công việc |
| requirements | TEXT | | Yêu cầu ứng viên |
| benefits | TEXT | | Phúc lợi |
| salary_min | DECIMAL(15,2) | | Mức lương tối thiểu |
| salary_max | DECIMAL(15,2) | | Mức lương tối đa |
| location | VARCHAR(255) | | Địa điểm làm việc |
| employment_type | ENUM('FULL_TIME','PART_TIME','INTERNSHIP','CONTRACT') | | Loại hình công việc |
| status | ENUM('DRAFT','PUBLISHED','CLOSED','CANCELLED') | NOT NULL | Trạng thái tin |
| published_at | DATETIME | | Ngày đăng |
| closed_at | DATETIME | | Ngày đóng |
| created_by | BIGINT | FK → user(id) | Người tạo tin |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | |
| updated_at | DATETIME | ON UPDATE CURRENT_TIMESTAMP | |

### 1.2. Bảng `job_application` – Hồ sơ ứng viên

| Tên cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---------|--------------|-----------|-------|
| id | BIGINT | PK, AUTO_INCREMENT | Mã hồ sơ |
| recruitment_post_id | BIGINT | FK → recruitment_post(id) | Tin tuyển dụng ứng tuyển |
| full_name | VARCHAR(100) | NOT NULL | Họ tên ứng viên |
| email | VARCHAR(100) | NOT NULL | Email |
| phone | VARCHAR(20) | NOT NULL | Số điện thoại |
| resume_url | VARCHAR(500) | | Đường dẫn file CV |
| cover_letter | TEXT | | Thư giới thiệu |
| source | VARCHAR(50) | | Nguồn ứng viên (Website, Referral, ...) |
| status | ENUM('PENDING','SCREENED','INTERVIEW_SCHEDULED','INTERVIEWED','OFFERED','HIRED','REJECTED') | NOT NULL | Trạng thái xử lý |
| applied_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | Ngày ứng tuyển |
| updated_at | DATETIME | ON UPDATE CURRENT_TIMESTAMP | |

### 1.3. Bảng `interview_schedule` – Lịch phỏng vấn

| Tên cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---------|--------------|-----------|-------|
| id | BIGINT | PK, AUTO_INCREMENT | Mã lịch phỏng vấn |
| job_application_id | BIGINT | FK → job_application(id) | Hồ sơ ứng viên |
| interviewer_id | BIGINT | FK → user(id) | Người phỏng vấn (nhân viên) |
| interview_date | DATETIME | NOT NULL | Thời gian phỏng vấn |
| duration_minutes | INT | DEFAULT 30 | Thời lượng (phút) |
| location | VARCHAR(255) | | Địa điểm (online/offline) |
| meeting_link | VARCHAR(500) | | Link nếu online |
| type | ENUM('TECHNICAL','HR','FINAL') | NOT NULL | Vòng phỏng vấn |
| status | ENUM('SCHEDULED','COMPLETED','CANCELLED','NO_SHOW') | NOT NULL | Trạng thái buổi phỏng vấn |
| feedback | TEXT | | Nhận xét của người phỏng vấn |
| result | ENUM('PASS','FAIL','PENDING') | | Kết quả |
| created_at | DATETIME | DEFAULT CURRENT_TIMESTAMP | |

### 1.4. Bảng `offer_letter` – Thư mời nhận việc (nếu cần)

| Tên cột | Kiểu dữ liệu | Ràng buộc | Mô tả |
|---------|--------------|-----------|-------|
| id | BIGINT | PK, AUTO_INCREMENT | Mã offer |
| job_application_id | BIGINT | FK → job_application(id) | Hồ sơ ứng viên |
| offered_salary | DECIMAL(15,2) | | Mức lương đề xuất |
| joining_date | DATE | | Ngày dự kiến nhận việc |
| content | TEXT | | Nội dung thư |
| status | ENUM('DRAFT','SENT','ACCEPTED','DECLINED','EXPIRED') | | Trạng thái |
| sent_at | DATETIME | | Ngày gửi |
| responded_at | DATETIME | | Ngày phản hồi |

---

## 2. Thiết kế API (RESTful)

### 2.1. Quản lý tin tuyển dụng

| Method | Endpoint | Mô tả |
|--------|----------|-------|
| POST | `/api/recruitment/posts` | Tạo tin tuyển dụng mới (status = DRAFT) |
| GET | `/api/recruitment/posts` | Danh sách tin tuyển dụng (có phân trang, lọc theo status, department) |
| GET | `/api/recruitment/posts/{id}` | Chi tiết một tin |
| PUT | `/api/recruitment/posts/{id}` | Cập nhật tin |
| PATCH | `/api/recruitment/posts/{id}/publish` | Đăng tin (chuyển status từ DRAFT → PUBLISHED) |
| PATCH | `/api/recruitment/posts/{id}/close` | Đóng tin (PUBLISHED → CLOSED) |
| DELETE | `/api/recruitment/posts/{id}` | Xóa tin (chỉ khi status = DRAFT) |

**Ví dụ Request Body (POST):**
```json
{
  "title": "Java Developer",
  "departmentId": 5,
  "position": "Senior",
  "quantity": 2,
  "description": "...",
  "requirements": "...",
  "benefits": "...",
  "salaryMin": 15000000,
  "salaryMax": 25000000,
  "location": "Hà Nội",
  "employmentType": "FULL_TIME"
}
```

### 2.2. Quản lý hồ sơ ứng viên

| Method | Endpoint | Mô tả |
|--------|----------|-------|
| POST | `/api/recruitment/applications` | Ứng viên nộp hồ sơ (public) |
| GET | `/api/recruitment/posts/{postId}/applications` | DS hồ sơ của một tin (HR) |
| GET | `/api/recruitment/applications/{id}` | Chi tiết hồ sơ |
| PUT | `/api/recruitment/applications/{id}` | Cập nhật thông tin hồ sơ (HR) |
| PATCH | `/api/recruitment/applications/{id}/status` | Cập nhật trạng thái hồ sơ (SCREENED, REJECTED, v.v.) |
| DELETE | `/api/recruitment/applications/{id}` | Xóa hồ sơ (chưa có lịch phỏng vấn) |

**Ví dụ Request Body (POST ứng viên):**
```json
{
  "recruitmentPostId": 10,
  "fullName": "Nguyễn Văn A",
  "email": "vana@example.com",
  "phone": "0912345678",
  "resumeUrl": "https://...",
  "coverLetter": "...",
  "source": "WEBSITE"
}
```

### 2.3. Quản lý lịch phỏng vấn

| Method | Endpoint | Mô tả |
|--------|----------|-------|
| POST | `/api/recruitment/interviews` | Tạo lịch phỏng vấn (tự động cập nhật status ứng viên thành INTERVIEW_SCHEDULED) |
| GET | `/api/recruitment/applications/{applicationId}/interviews` | DS lịch phỏng vấn của một hồ sơ |
| GET | `/api/recruitment/interviews/{id}` | Chi tiết buổi phỏng vấn |
| PUT | `/api/recruitment/interviews/{id}` | Cập nhật lịch (thời gian, địa điểm) |
| PATCH | `/api/recruitment/interviews/{id}/complete` | Hoàn thành phỏng vấn (cập nhật feedback, result, status) |
| DELETE | `/api/recruitment/interviews/{id}` | Hủy lịch (cập nhật status CANCELLED) |

**Ví dụ Request Body (POST):**
```json
{
  "jobApplicationId": 123,
  "interviewerId": 45,
  "interviewDate": "2026-04-10T09:00:00",
  "durationMinutes": 60,
  "location": "Phòng họp A",
  "type": "TECHNICAL"
}
```

### 2.4. Quản lý offer

| Method | Endpoint | Mô tả |
|--------|----------|-------|
| POST | `/api/recruitment/offers` | Tạo offer (status DRAFT) |
| PUT | `/api/recruitment/offers/{id}` | Cập nhật offer |
| PATCH | `/api/recruitment/offers/{id}/send` | Gửi offer (chuyển status SENT, ghi nhận sent_at) |
| PATCH | `/api/recruitment/offers/{id}/response` | Ứng viên phản hồi (ACCEPTED/DECLINED) |
| GET | `/api/recruitment/applications/{applicationId}/offer` | Xem offer của hồ sơ |

---

## 3. Xử lý nghiệp vụ trong Service Layer (Spring Boot)

### 3.1. Đăng tin tuyển dụng

- Khi tạo mới, tin có `status = DRAFT`.
- Chỉ khi gọi endpoint `publish`, hệ thống mới kiểm tra các trường bắt buộc (title, department, position, etc.) và chuyển `status = PUBLISHED`, đồng thời gán `published_at = now()`.
- Khi đóng tin, nếu đang có ứng viên chưa xử lý hết có thể hiển thị cảnh báo (nhưng vẫn cho phép đóng).

### 3.2. Nộp hồ sơ

- Người dùng không cần đăng nhập vẫn nộp được (public).
- Có thể validate trùng email + postId (nếu muốn ngăn nộp trùng).
- Sau khi nộp, status mặc định là `PENDING`.
- Có thể lưu file CV lên server (hoặc S3) và lưu URL vào `resume_url`.

### 3.3. Xử lý hồ sơ và lịch phỏng vấn

- Khi HR cập nhật trạng thái ứng viên, có thể kích hoạt các hành động: nếu chuyển sang `REJECTED` thì không thể tạo lịch phỏng vấn mới.
- Khi tạo lịch phỏng vấn, hệ thống tự động cập nhật status ứng viên thành `INTERVIEW_SCHEDULED`.
- Khi hoàn thành phỏng vấn, dựa vào `result` để cập nhật status ứng viên (PASS → INTERVIEWED hoặc nếu vòng cuối thì có thể chuyển sang OFFERED; FAIL → REJECTED).

### 3.4. Offer và tuyển dụng

- Sau khi ứng viên vượt qua phỏng vấn, HR có thể tạo offer với status `DRAFT`.
- Khi gửi offer, status chuyển thành `SENT`, hệ thống có thể gửi email thông báo.
- Khi ứng viên chấp nhận (ACCEPTED), status ứng viên chuyển thành `HIRED`; lúc này có thể tạo bản ghi nhân viên mới trong bảng `employee` (tùy thuộc vào thiết kế tổng thể).

---

## 4. Code mẫu Spring Boot (các phần chính)

### 4.1. Entity

```java
@Entity
@Table(name = "recruitment_post")
@Data
@NoArgsConstructor
public class RecruitmentPost {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
    
    private String position;
    private Integer quantity;
    private String description;
    private String requirements;
    private String benefits;
    private BigDecimal salaryMin;
    private BigDecimal salaryMax;
    private String location;
    
    @Enumerated(EnumType.STRING)
    private EmploymentType employmentType;
    
    @Enumerated(EnumType.STRING)
    private PostStatus status;
    
    private LocalDateTime publishedAt;
    private LocalDateTime closedAt;
    
    @ManyToOne
    @JoinColumn(name = "created_by")
    private User createdBy;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // getters, setters, prePersist...
}
```

### 4.2. Repository

```java
public interface RecruitmentPostRepository extends JpaRepository<RecruitmentPost, Long> {
    Page<RecruitmentPost> findByStatus(PostStatus status, Pageable pageable);
    Page<RecruitmentPost> findByDepartmentIdAndStatus(Long deptId, PostStatus status, Pageable pageable);
}
```

### 4.3. Service

```java
@Service
@Transactional
public class RecruitmentService {
    @Autowired
    private RecruitmentPostRepository postRepository;
    
    public RecruitmentPost publishPost(Long postId) {
        RecruitmentPost post = postRepository.findById(postId)
            .orElseThrow(() -> new ResourceNotFoundException("Post not found"));
        if (post.getStatus() != PostStatus.DRAFT) {
            throw new InvalidStateException("Only draft posts can be published");
        }
        post.setStatus(PostStatus.PUBLISHED);
        post.setPublishedAt(LocalDateTime.now());
        return postRepository.save(post);
    }
    
    // các phương thức khác tương tự
}
```

### 4.4. Controller

```java
@RestController
@RequestMapping("/api/recruitment/posts")
public class RecruitmentPostController {
    @Autowired
    private RecruitmentService recruitmentService;
    
    @PostMapping
    public ResponseEntity<RecruitmentPost> createPost(@RequestBody RecruitmentPostDto dto) {
        return ResponseEntity.ok(recruitmentService.createPost(dto));
    }
    
    @PatchMapping("/{id}/publish")
    public ResponseEntity<RecruitmentPost> publishPost(@PathVariable Long id) {
        return ResponseEntity.ok(recruitmentService.publishPost(id));
    }
    
    // ... các endpoint khác
}
```

---

## 5. Lưu ý về bảo mật và phân quyền

- Sử dụng Spring Security với JWT hoặc session-based.
- Phân quyền:
  - **HR_ROLE**: có quyền tạo, sửa, đăng tin, xem danh sách ứng viên, tạo lịch phỏng vấn, tạo offer.
  - **MANAGER_ROLE**: có thể xem tin tuyển dụng của phòng ban mình quản lý, đánh giá phỏng vấn.
  - **CANDIDATE (public)**: chỉ được phép nộp hồ sơ, không cần token.
- Endpoint nộp hồ sơ công khai nên cần có cơ chế chống spam (rate limiting, captcha).

---

Nếu bạn cần tôi triển khai chi tiết một phần cụ thể (ví dụ: tạo entity chi tiết, viết service hoàn chỉnh, hoặc thiết kế DTO) hãy cho biết.
