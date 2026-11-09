# Модель данных (v0.1, черновик для Prisma)

Система не привязана к конкретному вузу: шкала оценок, баллы за достижения и пороги риска хранятся в настройках.

## Сущности

### Пользователи и структура
- **User**: id, email, passwordHash, fullName, role (STUDENT | TEACHER | ADVISOR | DEAN), locale (ru|kk|en), avatarUrl, createdAt
- **Faculty**: id, name
- **Group**: id, name (например «ИС-21-1»), facultyId, course (1–4), advisorId → User(ADVISOR)
- **StudentProfile**: userId, groupId, studentNumber, enrollmentYear
- **TeacherProfile**: userId, facultyId, position

### Учебный процесс
- **Semester**: id, name, startDate, endDate, isCurrent
- **Subject**: id, code, name, credits
- **CourseOffering** (предмет у группы в семестре): id, subjectId, groupId, teacherId, semesterId
- **Grade**: id, studentId, offeringId, type (CURRENT | MIDTERM | FINAL | TOTAL), score (0–100), date
- **Attendance**: id, studentId, offeringId, date, status (PRESENT | ABSENT | LATE | EXCUSED)
- **ScheduleSlot** (полная версия): id, offeringId, dayOfWeek, startTime, endTime, room

### Шкала и расчёты
- **GradeScale**: minScore, maxScore, letter, gpaPoints (например 95–100 → A → 4.0). Редактирует деканат.
- GPA = Σ(gpaPoints × credits) / Σ credits. Берётся по TOTAL-оценке каждого предмета. Значение вычисляется, в базе не хранится.
- **Рейтинг** (вычисляемый, внутри группы или курса), два режима фильтра:
  1. `GPA`: только GPA;
  2. `GPA_PLUS`: GPA + нормализованные баллы за подтверждённые достижения (вес задаётся в Setting).

### Достижения
- **Achievement**: id, studentId, type (HACKATHON | OLYMPIAD | CERTIFICATE | PUBLICATION | SPORT), title, date, result (например «1 место», «финалист»), description, fileUrl (сертификат), status (PENDING | APPROVED | REJECTED), reviewedById → User(ADVISOR), reviewedAt, reviewComment, points (выставляются при подтверждении по таблице AchievementPoints), eventId? → Event
- **HackathonDetails** (1:1 к Achievement с type = HACKATHON): teamName, teamMembers, projectName, projectDescription, githubUrl, demoUrl
- **AchievementPoints**: type, result-level (PARTICIPANT | FINALIST | PRIZE | WINNER), points. Редактирует деканат.

### Каталог мероприятий (ведёт деканат)
- **Event**: id, type (HACKATHON | OLYMPIAD | CONFERENCE | OTHER), title, description, organizer, startDate, endDate, registrationDeadline, url, tags[], isOnline, city

### Риск отчисления
- **RiskAssessment** (кэш расчёта): studentId, semesterId, level (LOW | MEDIUM | HIGH), score, reasons (JSON: список причин), method (RULES | ML), calculatedAt
- Правила для MVP (пороги берутся из Setting, значения ниже — предложение):
  - GPA < 2.0 → +2
  - посещаемость < 70% → +2, < 80% → +1
  - TOTAL < 50 по любому предмету (долг) → +2 за каждый
  - итог: 0–1 LOW, 2–3 MEDIUM, ≥4 HIGH

### ИИ-ассистент
- **ChatSession**: id, userId, title, createdAt
- **ChatMessage**: id, sessionId, role (user | assistant | tool), content, provider, model, createdAt
- **AiProvider** (полная версия, настройки деканата): name, baseUrl, model, isEnabled, priority

### Прочее
- **Setting**: key, value (JSON). Здесь веса рейтинга, пороги риска, провайдер ИИ по умолчанию.
- **Notification** (полная версия): userId, type, payload, readAt

## Права доступа (кратко)
| Данные | STUDENT | TEACHER | ADVISOR | DEAN |
|---|---|---|---|---|
| Свои оценки и посещаемость | ✅ | — | — | — |
| Оценки по предмету | — | свои предметы (ввод) | своя группа (просмотр) | все |
| Достижения | свои (создание) | просмотр своих студентов | своя группа (подтверждение) | все |
| Риск | свой | свои студенты | своя группа | все |
| Каталог мероприятий | просмотр | просмотр | просмотр | CRUD |
| Пользователи, группы, шкала, настройки | — | — | — | CRUD |

## Демо-данные (seed)
- 1 факультет, 3 группы, 2 курса, по ~20 студентов в группе, 6 преподавателей, 3 эдвайзера, 1 деканат.
- 2 семестра оценок и посещаемости. Распределение реалистичное: 10–15% студентов в зоне риска.
- ~40 достижений разных типов и статусов, ~10 мероприятий в каталоге.
- Демо-логины: `student@demo.kz`, `teacher@demo.kz`, `advisor@demo.kz`, `dean@demo.kz` / пароль `demo1234`.
