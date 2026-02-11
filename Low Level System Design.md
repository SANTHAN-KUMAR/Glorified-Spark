# SecureExam Platform — Low-Level System Design (LLD)

## 1. Database Schema

### Entity-Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ EXAM_SESSIONS : takes
    USERS ||--o| FACE_ENROLLMENTS : has
    EXAMS ||--o{ QUESTIONS : contains
    EXAMS ||--o{ EXAM_SESSIONS : has
    EXAM_SESSIONS ||--o{ ANSWERS : includes
    EXAM_SESSIONS ||--o{ CODE_SUBMISSIONS : includes
    EXAM_SESSIONS ||--o| PROCTORING_SESSIONS : monitored_by
    PROCTORING_SESSIONS ||--o{ PROCTORING_FRAMES : captures
    PROCTORING_SESSIONS ||--o{ VIOLATIONS : flags
    EXAMS ||--o{ PLAGIARISM_JOBS : triggers
    PLAGIARISM_JOBS ||--o{ PLAGIARISM_RESULTS : produces
    
    USERS {
        uuid id PK
        string email UK
        string password_hash
        string full_name
        enum role "student|admin|superadmin"
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }
    
    FACE_ENROLLMENTS {
        uuid id PK
        uuid user_id FK
        string photo_url "Encrypted S3 path"
        jsonb embedding "512-dim ArcFace vector"
        timestamp enrolled_at
        timestamp expires_at
    }
    
    EXAMS {
        uuid id PK
        uuid created_by FK
        string title
        text description
        int duration_minutes
        timestamp start_window
        timestamp end_window
        enum status "draft|published|active|closed"
        jsonb seb_config "SEB settings override"
        boolean shuffle_questions
        boolean shuffle_options
        float passing_score
        timestamp created_at
    }
    
    QUESTIONS {
        uuid id PK
        uuid exam_id FK
        enum type "mcq|coding"
        text body "Markdown content"
        jsonb options "MCQ: [{id, text, is_correct}]"
        jsonb test_cases "Coding: [{input, expected, hidden}]"
        float points
        int time_limit_seconds "For coding: execution limit"
        int memory_limit_mb
        int order_index
    }
    
    EXAM_SESSIONS {
        uuid id PK
        uuid user_id FK
        uuid exam_id FK
        enum status "started|in_progress|submitted|timed_out|flagged"
        timestamp started_at
        timestamp submitted_at
        timestamp deadline "Server-calculated"
        float total_score
        string seb_config_key "BEK verification"
        string ip_address
        string user_agent
    }
    
    ANSWERS {
        uuid id PK
        uuid session_id FK
        uuid question_id FK
        jsonb response "MCQ: selected_option_id | Coding: null"
        int time_spent_ms
        boolean is_correct
        float score
        timestamp answered_at
        timestamp updated_at
    }
    
    CODE_SUBMISSIONS {
        uuid id PK
        uuid session_id FK
        uuid question_id FK
        text source_code
        string language "python|java|cpp|etc"
        enum status "queued|running|completed|error"
        jsonb results "[{test_case_id, passed, stdout, stderr, time_ms, memory_kb}]"
        int passed_count
        int total_count
        float score
        timestamp submitted_at
    }
    
    PROCTORING_SESSIONS {
        uuid id PK
        uuid session_id FK
        enum status "active|ended|error"
        int total_frames
        int total_violations
        float avg_face_confidence
        timestamp started_at
        timestamp ended_at
    }
    
    PROCTORING_FRAMES {
        bigint id PK "Auto-increment for speed"
        uuid proctoring_session_id FK
        string snapshot_url "S3 path, null if not stored"
        float face_confidence
        int face_count
        float gaze_yaw
        float gaze_pitch
        jsonb detected_objects "[{class, confidence, bbox}]"
        boolean speech_detected
        int speaker_count
        timestamp captured_at
    }
    
    VIOLATIONS {
        uuid id PK
        uuid proctoring_session_id FK
        enum type "no_face|multi_face|face_mismatch|gaze_deviation|phone_detected|multi_speaker|tab_switch"
        enum severity "warning|alert|critical"
        string snapshot_url
        text details
        float confidence
        int duration_seconds
        enum review_status "pending|confirmed|dismissed"
        uuid reviewed_by FK
        timestamp occurred_at
        timestamp reviewed_at
    }
    
    PLAGIARISM_JOBS {
        uuid id PK
        uuid exam_id FK
        uuid question_id FK
        enum status "queued|processing|completed|failed"
        enum engine "dolos|moss|both"
        timestamp started_at
        timestamp completed_at
    }
    
    PLAGIARISM_RESULTS {
        uuid id PK
        uuid job_id FK
        uuid submission_a FK
        uuid submission_b FK
        float similarity_score "0.0 - 1.0"
        jsonb details "matched regions, token overlap"
        enum status "flagged|cleared"
    }
    
    AUDIT_LOG {
        bigint id PK
        uuid user_id FK
        string action
        string resource_type
        uuid resource_id
        jsonb metadata
        string ip_address
        timestamp created_at
    }
    
    SEB_CONFIGS {
        uuid id PK
        uuid exam_id FK
        text config_blob "Encrypted .seb XML"
        string config_key_hash
        timestamp generated_at
    }
```

### Key Indexes

```sql
-- Performance-critical queries
CREATE INDEX idx_exam_sessions_user     ON exam_sessions(user_id, exam_id);
CREATE INDEX idx_exam_sessions_status   ON exam_sessions(status) WHERE status = 'started';
CREATE INDEX idx_answers_session        ON answers(session_id);
CREATE INDEX idx_code_subs_session      ON code_submissions(session_id, question_id);
CREATE INDEX idx_proctor_frames_session ON proctoring_frames(proctoring_session_id, captured_at);
CREATE INDEX idx_violations_session     ON violations(proctoring_session_id, severity);
CREATE INDEX idx_violations_pending     ON violations(review_status) WHERE review_status = 'pending';
CREATE INDEX idx_plagiarism_score       ON plagiarism_results(similarity_score DESC);
CREATE INDEX idx_audit_user_time        ON audit_log(user_id, created_at DESC);
```

---

## 2. API Specification

### 2.1 Authentication

```
POST /api/v1/auth/register
  Body: { email, password, full_name }
  Response: { user_id, message }

POST /api/v1/auth/login
  Body: { email, password }
  Response: { access_token, refresh_token, expires_in }

POST /api/v1/auth/refresh
  Body: { refresh_token }
  Response: { access_token, expires_in }

POST /api/v1/auth/enroll-face
  Body: multipart/form-data { photo: File }
  Headers: Authorization: Bearer <token>
  Response: { enrollment_id, status: "enrolled" }
```

### 2.2 Exam Management (Admin)

```
POST /api/v1/exams
  Body: { title, description, duration_minutes, start_window, end_window,
          shuffle_questions, shuffle_options, passing_score }
  Response: { exam_id }

POST /api/v1/exams/{exam_id}/questions
  Body: { type: "mcq"|"coding", body, options?, test_cases?, points, 
          time_limit_seconds?, memory_limit_mb? }
  Response: { question_id }

POST /api/v1/exams/{exam_id}/publish
  Response: { status: "published", seb_config_download_url }

GET  /api/v1/exams/{exam_id}/sessions
  Query: ?status=submitted&page=1&limit=20
  Response: { sessions: [...], total, page }

GET  /api/v1/exams/{exam_id}/results
  Response: { results: [{ user, score, violations_count, plagiarism_flags }] }
```

### 2.3 Exam Taking (Student)

```
GET  /api/v1/student/exams
  Response: { exams: [{ id, title, start_window, end_window, duration }] }

POST /api/v1/student/exams/{exam_id}/start
  Response: { session_id, questions: [...], deadline, server_time }

POST /api/v1/student/sessions/{session_id}/answer
  Body: { question_id, response: "option_id" | null }
  Response: { saved: true }

POST /api/v1/student/sessions/{session_id}/submit
  Response: { status: "submitted", score? }
```

### 2.4 Code Execution

```
POST /api/v1/code/run
  Body: { source_code, language, stdin?, session_id, question_id }
  Response: { submission_id, status: "queued" }

GET  /api/v1/code/result/{submission_id}
  Response: { status, results: [{ test_case, passed, stdout, stderr, 
              time_ms, memory_kb }], score }

POST /api/v1/code/submit  (final submission)
  Body: { source_code, language, session_id, question_id }
  Response: { submission_id, status: "queued", is_final: true }
```

### 2.5 Proctoring

```
WebSocket: wss://api.your-platform.com/api/v1/proctoring/stream
  Auth: ?token=<jwt>
  → See Section 3 for WebSocket protocol

GET  /api/v1/admin/proctoring/live
  Response: { active_sessions: [{ user, violation_count, last_frame_url }] }

GET  /api/v1/admin/violations
  Query: ?severity=critical&status=pending&exam_id=...
  Response: { violations: [...], total }

PATCH /api/v1/admin/violations/{id}
  Body: { review_status: "confirmed"|"dismissed" }
  Response: { updated: true }
```

### 2.6 Plagiarism

```
POST /api/v1/admin/plagiarism/check
  Body: { exam_id, question_id?, engine: "dolos"|"moss"|"both" }
  Response: { job_id, status: "queued" }

GET  /api/v1/admin/plagiarism/jobs/{job_id}
  Response: { status, results_count, flagged_count }

GET  /api/v1/admin/plagiarism/results
  Query: ?job_id=...&min_score=0.7
  Response: { pairs: [{ student_a, student_b, similarity, details }] }
```

---

## 3. WebSocket Proctoring Protocol

### Connection Handshake

```
Client → Server: wss://api/v1/proctoring/stream?token=<jwt>&session_id=<uuid>
Server → Client: { type: "CONNECTED", proctoring_session_id: "<uuid>" }
```

### Message Types

```typescript
// Client → Server Messages
type ClientMessage =
  | { type: "FRAME",   data: string,  ts: number }  // base64 JPEG (320×240)
  | { type: "AUDIO",   data: string,  ts: number }  // base64 Opus chunk (5s)
  | { type: "EVENT",   event: BrowserEvent }
  | { type: "HEARTBEAT" }

type BrowserEvent = 
  | { name: "focus_lost",   ts: number }
  | { name: "focus_gained", ts: number }
  | { name: "resize",       width: number, height: number }
  | { name: "paste_attempt", ts: number }

// Server → Client Messages  
type ServerMessage =
  | { type: "WARNING",    message: string, violation_type: string }
  | { type: "ALERT",      message: string, violation_type: string }
  | { type: "CRITICAL",   message: string, action: "pause"|"terminate" }
  | { type: "ACK",        frame_id: number }
  | { type: "TIME_SYNC",  server_time: number }
  | { type: "HEARTBEAT_ACK" }
```

### Frame Processing Rate

| Check | Frequency | Server Load |
|-------|:---------:|-------------|
| Client-side face count | Every frame (3s) | None (runs in browser) |
| Server face verification | Every 30s | 1 inference per 30s per user |
| Object detection (YOLOv8) | Every 10s | 1 inference per 10s per user |
| Audio VAD | Continuous (5s chunks) | Lightweight, CPU only |
| Speaker diarization | Every 60s | 1 inference per 60s per user |

**At 500 users:**
- Face verification: ~17 inferences/sec → 1 GPU handles this easily
- Object detection: ~50 inferences/sec → YOLOv8-nano at ~100 FPS on T4, 1 GPU sufficient
- Audio: ~100 chunks/sec → CPU-only (Silero VAD is very lightweight)

---

## 4. AI Proctoring Pipeline — Detailed Processing

```mermaid
graph TB
    subgraph Input["Frame Arrival"]
        WS["WebSocket Server"]
    end
    
    subgraph Queue["RabbitMQ Routing"]
        EX["proctoring Exchange<br/>(topic)"]
        Q1["queue.frames.face<br/>(every 30s)"]
        Q2["queue.frames.objects<br/>(every 10s)"]
        Q3["queue.audio.vad<br/>(continuous)"]
        Q4["queue.audio.speakers<br/>(every 60s)"]
        Q5["queue.frames.store<br/>(every 30s + violations)"]
    end
    
    subgraph Workers["Processing Workers"]
        FW["Face Worker<br/>DeepFace ArcFace"]
        OW["Object Worker<br/>YOLOv8-nano"]
        AW["Audio VAD Worker<br/>Silero"]
        SW["Speaker Worker<br/>pyannote"]
        StW["Storage Worker"]
    end
    
    subgraph Output["Results"]
        VE["Violation Engine<br/>(aggregation + severity)"]
        DB["PostgreSQL"]
        S3["MinIO/S3"]
    end
    
    WS --> EX
    EX --> Q1 & Q2 & Q3 & Q4 & Q5
    Q1 --> FW
    Q2 --> OW
    Q3 --> AW
    Q4 --> SW
    Q5 --> StW --> S3
    
    FW & OW & AW & SW --> VE --> DB
```

### Violation Engine Logic (Pseudocode)

```python
class ViolationEngine:
    # Thresholds
    FACE_MATCH_THRESHOLD = 0.6          # DeepFace similarity
    GAZE_DEVIATION_THRESHOLD = 30       # degrees from center
    GAZE_SUSTAINED_SECONDS = 15         # how long before flagging
    PHONE_CONFIDENCE_THRESHOLD = 0.7    # YOLOv8 confidence
    MULTI_SPEAKER_THRESHOLD = 2         # speaker count

    def process_face_result(self, session_id, similarity, face_count):
        if face_count == 0:
            if self.duration_no_face(session_id) > 5:
                self.flag(session_id, "no_face", "warning")
            if self.duration_no_face(session_id) > 15:
                self.flag(session_id, "no_face", "alert")
        
        elif face_count > 1:
            self.flag(session_id, "multi_face", "alert")
        
        elif similarity < FACE_MATCH_THRESHOLD:
            self.flag(session_id, "face_mismatch", "critical")

    def process_object_result(self, session_id, detections):
        for obj in detections:
            if obj.class_name == "cell phone" and obj.confidence > 0.7:
                self.flag(session_id, "phone_detected", "critical")
            elif obj.class_name == "book" and obj.confidence > 0.7:
                self.flag(session_id, "book_detected", "alert")

    def process_audio_result(self, session_id, speech, speaker_count):
        if speaker_count >= 2:
            self.flag(session_id, "multi_speaker", "critical")

    def flag(self, session_id, type, severity):
        # Deduplication: don't flag same type within 30s window
        if self.recent_violation(session_id, type, window=30):
            return
        
        violation = Violation(session_id, type, severity, snapshot_url, timestamp)
        db.insert(violation)
        notify_admin(violation)  # Push via WebSocket to admin dashboard
        
        if severity == "critical":
            notify_student(session_id, "warning_displayed")
```

---

## 5. Key Sequence Diagrams

### 5.1 Exam Start (Full Flow)

```mermaid
sequenceDiagram
    participant S as Student (SEB)
    participant F as Frontend
    participant A as API Server
    participant R as Redis
    participant D as PostgreSQL
    participant W as WS Server
    
    S->>F: Open exam URL in SEB
    F->>A: POST /auth/login {email, password}
    A->>D: SELECT user WHERE email = ...
    A->>A: Verify password hash (bcrypt)
    A-->>F: {access_token, refresh_token}
    
    F->>A: POST /student/exams/{id}/start
    A->>D: Verify exam is within time window
    A->>D: Check no existing active session
    A->>D: INSERT exam_session (status=started)
    A->>D: SELECT questions WHERE exam_id (randomize order)
    A->>R: SET session:{id}:deadline = now + duration
    A->>R: HMSET session:{id}:meta {exam_id, user_id, ...}
    A-->>F: {session_id, questions, deadline, server_time}
    
    F->>F: Initialize timer (deadline - server_time)
    F->>F: getUserMedia() → start webcam + mic
    F->>W: Connect WebSocket (session_id, token)
    W->>D: INSERT proctoring_session
    W-->>F: {type: "CONNECTED", proctoring_session_id}
    
    Note over F: Exam is now live with proctoring
```

### 5.2 Face Enrollment

```mermaid
sequenceDiagram
    participant S as Student
    participant F as Frontend
    participant A as API
    participant AI as DeepFace
    participant S3 as MinIO
    participant D as PostgreSQL
    
    S->>F: Navigate to enrollment page
    F->>F: getUserMedia() → webcam
    F->>F: face-api.js → detect single face
    F->>F: Capture high-quality frame
    F->>A: POST /auth/enroll-face {photo: base64}
    
    A->>AI: Extract face embedding (ArcFace, 512-dim)
    AI-->>A: embedding vector
    A->>A: Validate: exactly 1 face, sufficient quality
    A->>S3: Store encrypted photo
    S3-->>A: photo_url
    A->>D: INSERT face_enrollment {user_id, photo_url, embedding}
    A-->>F: {status: "enrolled"}
```

### 5.3 Code Submission Flow

```mermaid
sequenceDiagram
    participant F as Frontend
    participant A as API
    participant Q as RabbitMQ
    participant J as Judge0
    participant D as PostgreSQL
    
    F->>A: POST /code/run {source, language, session_id, question_id}
    A->>D: INSERT code_submission (status=queued)
    A->>Q: Publish to "code.execute" queue
    A-->>F: {submission_id, status: "queued"}
    
    Q->>J: Submit to Judge0 API
    Note over J: Runs in sandboxed container<br/>Time limit: 5s, Memory: 256MB
    
    loop Each test case
        J->>J: Execute with test input
        J->>J: Compare output with expected
    end
    
    J-->>Q: Results
    Q->>D: UPDATE code_submission SET results, status=completed
    Q->>D: Calculate score (passed/total × points)
    
    F->>A: GET /code/result/{submission_id} (polling every 2s)
    A->>D: SELECT code_submission
    A-->>F: {status: "completed", results, score}
```

---

## 6. Caching Strategy (Redis)

| Key Pattern | TTL | Purpose |
|---|---|---|
| `session:{id}:deadline` | exam duration | Server-side timer (source of truth) |
| `session:{id}:meta` | exam duration + 1hr | Session metadata cache |
| `session:{id}:answers` | exam duration | Write-behind buffer for answers |
| `user:{id}:token` | 15min | JWT validation cache |
| `exam:{id}:questions` | until exam closes | Question cache (avoid DB reads) |
| `proctor:{session_id}:state` | exam duration | Face state (last seen, no-face duration) |
| `proctor:{session_id}:violations:recent` | 30s (sliding) | Deduplication window |
| `ratelimit:{user_id}:{endpoint}` | 1min | Rate limiting counters |
| `ws:connections` | – (SET) | Track active WebSocket connections per server |

### Write-Behind Pattern for Answers
```
Student answers → Redis HSET session:{id}:answers → 
  Flush to PostgreSQL every 10s OR on exam submit (whichever first)
```

---

## 7. Queue Topology (RabbitMQ)

```
Exchange: proctoring (topic)
├── Routing: frame.face      → queue.frames.face     (3 consumers)
├── Routing: frame.objects    → queue.frames.objects   (2 consumers)
├── Routing: audio.vad        → queue.audio.vad        (2 consumers)
├── Routing: audio.speakers   → queue.audio.speakers   (1 consumer)
└── Routing: frame.store      → queue.frames.store     (2 consumers)

Exchange: code (direct)
└── Routing: execute          → queue.code.execute     (4 consumers)

Exchange: plagiarism (direct)
└── Routing: check            → queue.plagiarism.check (1 consumer)

Exchange: notifications (fanout)
├── → queue.notifications.email
└── → queue.notifications.websocket
```

---

## 8. SEB Configuration Spec

Generated per exam as a `.seb` file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <!-- Exam URL -->
    <key>startURL</key>
    <string>https://exam.your-platform.com/session/{exam_id}</string>

    <!-- Security -->
    <key>allowQuit</key>        <false/>
    <key>allowSpellCheck</key>  <false/>
    <key>enableClipboard</key>  <false/>
    
    <!-- Browser -->    
    <key>enableJavaScript</key>       <true/>
    <key>allowMediaAutoCapture</key>  <true/>
    <key>mediaAutoCaptureMicrophone</key> <true/>
    <key>mediaAutoCaptureCamera</key> <true/>
    
    <!-- URL Filter -->
    <key>URLFilterEnable</key>  <true/>
    <key>URLFilterRules</key>
    <array>
        <dict>
            <key>action</key>   <integer>1</integer>  <!-- allow -->
            <key>expression</key>
            <string>exam.your-platform.com/*</string>
        </dict>
    </array>
    
    <!-- Quit -->
    <key>quitURL</key>
    <string>https://exam.your-platform.com/submitted</string>
</dict>
</plist>
```

---

## 9. Error Handling Matrix

| Error Code | HTTP Status | Scenario | Client Action |
|:----------:|:----------:|----------|---------------|
| `AUTH_001` | 401 | Invalid credentials | Show error, retry |
| `AUTH_002` | 403 | Account disabled | Contact admin |
| `EXAM_001` | 409 | Session already exists | Resume existing session |
| `EXAM_002` | 403 | Exam not in time window | Show "exam not available" |
| `EXAM_003` | 410 | Deadline passed | Auto-submit, show results |
| `CODE_001` | 408 | Execution timeout | Show TLE message |
| `CODE_002` | 500 | Judge0 unavailable | Retry queue, show "processing" |
| `PROC_001` | – | WebSocket disconnect | Auto-reconnect (exp. backoff, max 5) |
| `PROC_002` | – | Camera permission denied | Block exam start, show instructions |
| `PLAG_001` | 503 | MOSS/Dolos unavailable | Retry later, notify admin |
