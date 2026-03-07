# Phase 3: MOSS-UI 완전 통합

**날짜**: 2026-03-08
**상태**: 설계 (구현 준비)
**이전**: Phase 2 완료 (cbf0147)

---

## 목표

Phase 2에서 검증을 완성했으므로, Phase 3은 **UI 렌더링 자동화**에 집중합니다:

1. **자동 UI 생성**: 필드 타입 → 적절한 UI 컴포넌트 매핑
2. **실시간 피드백**: 입력 시 즉시 검증 + 에러 표시
3. **애니메이션**: 부드러운 에러/성공 효과
4. **반응형 디자인**: 모바일/데스크톱 최적화
5. **접근성**: ARIA 레이블, 키보드 네비게이션

---

## 1️⃣ 자동 UI 생성 (Type → Component 매핑)

### 현황 (Phase 2)

```freelang
let form = SignupForm()

// 수동으로 각 필드마다 입력 타입 지정
UI.form(bind: form)
    .input("username", "text", "사용자명")
    .input("email", "email", "이메일")
    .input("password", "password", "비밀번호")
    .input("age", "number", "나이")
    .checkbox("agree_terms", "약관 동의")
```

### 목표 (Phase 3)

```freelang
let form = SignupForm()

// Type을 보고 자동으로 UI 생성
UI.auto_form(bind: form)  // ← 타입별 자동 변환
    // username: string → <input type="text">
    // email: string (@email) → <input type="email">
    // password: string (@min(8)) → <input type="password">
    // age: int (@range(18, 99)) → <input type="number" min="18" max="99">
    // agree_terms: bool → <input type="checkbox">
```

### 매핑 규칙

**타입별 자동 매핑**:

| FreeLang 타입 | UI 컴포넌트 | 설명 |
|--------|---------|------|
| `string` | `<input type="text">` | 기본 텍스트 |
| `string` (@email) | `<input type="email">` | 이메일 검증 |
| `string` (@min(8)) | `<input type="password">` | 비밀번호 (8자+) |
| `string` (@match(regex)) | `<input type="text" pattern="...">` | 패턴 입력 |
| `int` | `<input type="number">` | 숫자 |
| `int` (@range(min, max)) | `<input type="range">` | 슬라이더 |
| `bool` | `<input type="checkbox">` | 체크박스 |
| `[string]` | `<select multiple>` | 다중 선택 |
| `string` (긴 텍스트) | `<textarea>` | 여러 줄 입력 |

### 구현 전략

```freelang
// src/ui/AutoFormRenderer.fl (신규)

struct FieldTypeDetector {
    fn detect_field_type(field: FieldMetadata) -> string {
        // 타입 + 어노테이션 기반 판별
        match (field.type, field.constraints) {
            ("string", has @email) => "email",
            ("string", has @password) => "password",
            ("int", has @range) => "range",
            ("bool", _) => "checkbox",
            _ => default_type
        }
    }
}

struct AutoUIGenerator {
    fn generate_input(field: FieldMetadata) -> string {
        let input_type = detect_field_type(field)

        let html = "<input type=\"" + input_type + "\""
        html += " id=\"" + field.name + "\""
        html += " name=\"" + field.name + "\""
        html += " placeholder=\"" + field.label + "\""

        // 어노테이션 기반 속성 추가
        if has @min {
            html += " minlength=\"" + min_value + "\""
        }
        if has @max {
            html += " maxlength=\"" + max_value + "\""
        }
        if has @required {
            html += " required"
        }

        html += " />"
        return html
    }
}
```

---

## 2️⃣ 실시간 검증 피드백

### 구현 흐름

```
사용자 입력
  ↓
onChange 이벤트
  ↓
mark_dirty(field)
  ↓
validate_field (async if needed)
  ↓
에러 표시 + 성공 표시
  ↓
UI 갱신 (실시간)
```

### 코드 예시

```freelang
struct RealtimeFormValidator {
    form: FormStruct,
    on_field_change: fn(field: string, value: any),
    debounce_ms: int = 300,  // 입력 300ms 후 검증

    async fn bind_field_realtime(field_name: string, value: any) {
        // 1. 값 업데이트
        this.form.set_value(field_name, value)
        this.form.mark_dirty(field_name)

        // 2. 디바운스 (입력 중에는 검증 안 함)
        wait(this.debounce_ms)

        // 3. 비동기 검증
        let result = await this.form.validate_field_async(field_name)

        // 4. UI 업데이트
        if result.is_valid {
            show_success(field_name)        // ✅ 초록색
        } else {
            show_error(field_name, result)  // ❌ 빨간색 + 메시지
        }
    }
}
```

### 성능 최적화

**디바운싱** (300ms):
```
사용자: a-b-c 입력 (3글자, 100ms 간격)
검증 실행 횟수: 1회 (마지막 입력 후 300ms)

vs Formik (매 글자마다):
검증 실행 횟수: 3회 ❌ (오버헤드)
```

---

## 3️⃣ 애니메이션 & 시각적 피드백

### CSS 애니메이션

```css
/* 에러 표시 - 슬라이드인 + 페이드인 */
@keyframes slideInError {
    from {
        opacity: 0;
        transform: translateY(-10px);
    }
    to {
        opacity: 1;
        transform: translateY(0);
    }
}

.form-field.error .error-message {
    animation: slideInError 0.3s ease-out;
}

/* 성공 표시 - 체크마크 애니메이션 */
@keyframes checkmark {
    from {
        transform: scale(0);
        opacity: 0;
    }
    to {
        transform: scale(1);
        opacity: 1;
    }
}

.form-field.success::after {
    content: "✓";
    animation: checkmark 0.3s ease-out;
    color: #27ae60;
}
```

### 상태 변화

```
[초기]
<input placeholder="이메일" />

[입력 중] (검증 대기)
<input value="user@" class="validating" />
 → 로딩 스피너

[검증 완료 - 실패]
<input value="user@" class="error" />
<span class="error-message">올바른 이메일이 아닙니다</span>

[검증 완료 - 성공]
<input value="user@example.com" class="success" />
<span class="success-message">✓</span>
```

---

## 4️⃣ 반응형 디자인

### 모바일 최적화

```css
/* 모바일: 단일 열 */
@media (max-width: 768px) {
    .form-field {
        width: 100%;
        margin-bottom: 20px;
    }

    .form-submit {
        font-size: 16px;  /* iOS 줌 방지 */
    }
}

/* 데스크톱: 2열 (선택사항) */
@media (min-width: 1024px) {
    .form-row {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 20px;
    }
}
```

### Touch 이벤트

```freelang
// 모바일에서는 onChange 대신 onBlur 사용
// (더 효율적인 검증)

fn bind_field_mobile(field_name: string) {
    // onBlur: 입력 완료 후 검증
    on_blur: fn() {
        validate_field(field_name)
    }
}
```

---

## 5️⃣ 접근성 (ARIA)

### ARIA 레이블

```html
<!-- 스크린 리더 지원 -->
<label for="email" id="email-label">
    이메일 <span class="required">*</span>
</label>
<input
    id="email"
    aria-labelledby="email-label"
    aria-required="true"
    aria-invalid="false"
    aria-describedby="email-error"
/>
<span id="email-error" role="alert"></span>
```

### 키보드 네비게이션

```
Tab: 다음 필드로 이동
Shift+Tab: 이전 필드로 이동
Enter: 폼 제출 (마지막 필드에서)
Escape: 폼 리셋
```

---

## 📋 Phase 3 구현 파일

### 1. `src/ui/AutoFormRenderer.fl` (신규)

```freelang
struct FieldTypeDetector { ... }
struct AutoUIGenerator { ... }

fn detect_field_type() -> string
fn generate_input() -> string
fn generate_select() -> string
fn generate_textarea() -> string
```

### 2. `src/ui/RealtimeValidator.fl` (신규)

```freelang
struct RealtimeFormValidator { ... }
struct DebounceManager { ... }

async fn validate_field_with_debounce()
fn show_error_animation()
fn show_success_animation()
```

### 3. `src/ui/Animations.fl` (신규)

```css
// 애니메이션 정의
// - slideInError
// - checkmark
// - fadeOut
// - shake (invalid)
```

### 4. `src/ui/FormStyles.fl` (신규)

```css
// 전체 폼 스타일
// - 모바일 반응형
// - 다크/라이트 테마
// - 상태별 색상
```

### 5. `examples/complete-form.fl` (신규)

```freelang
// 전체 통합 예제
// - 자동 UI 생성
// - 실시간 검증
// - 애니메이션
// - 접근성
```

### 6. `tests/ui-integration.test.fl` (신규)

```freelang
// UI 통합 테스트
// - 자동 타입 감지
// - 실시간 피드백
// - 애니메이션 트리거
```

---

## 🎨 UI 레이아웃 예시

### 구조

```html
<div class="form-container">
    <form class="direct-bind-form">
        <!-- 필드 1 -->
        <div class="form-field validating">
            <label for="username">
                사용자명 <span class="required">*</span>
            </label>
            <input
                id="username"
                type="text"
                placeholder="3-20자"
                class="form-input"
            />
            <span class="loading-spinner"></span>
        </div>

        <!-- 필드 2 -->
        <div class="form-field error">
            <label for="email">
                이메일 <span class="required">*</span>
            </label>
            <input
                id="email"
                type="email"
                value="invalid"
                class="form-input error"
                aria-invalid="true"
                aria-describedby="email-error"
            />
            <span id="email-error" class="error-message" role="alert">
                올바른 이메일이 아닙니다
            </span>
        </div>

        <!-- 필드 3 -->
        <div class="form-field success">
            <label for="password">
                비밀번호 <span class="required">*</span>
            </label>
            <input
                id="password"
                type="password"
                value="SecurePass123!"
                class="form-input success"
                aria-invalid="false"
            />
            <span class="success-message">✓</span>
        </div>

        <!-- 제출 -->
        <button type="submit" class="form-submit">
            가입하기
        </button>

        <!-- 전체 에러 -->
        <div class="form-errors" role="alert"></div>
    </form>
</div>
```

---

## 📊 성능 목표 (Phase 3)

| 메트릭 | Phase 2 | Phase 3 목표 |
|--------|--------|------------|
| 자동 UI 생성 | - | <50ms |
| 실시간 검증 (디바운스) | - | 300ms |
| 에러 애니메이션 | - | 300ms |
| 필드 렌더링 | - | <100ms |
| 전체 폼 렌더링 | - | <500ms (10필드) |
| 메모리 (UI 상태) | - | <500B/필드 |

---

## 📝 Phase 3 체크리스트

### 1. 자동 UI 생성
- [ ] FieldTypeDetector 구현
- [ ] AutoUIGenerator 구현
- [ ] 타입별 속성 매핑 (min, max, required, pattern)
- [ ] 동적 placeholder 생성

### 2. 실시간 검증
- [ ] RealtimeFormValidator 구현
- [ ] DebounceManager 구현
- [ ] 에러 표시 함수
- [ ] 성공 표시 함수

### 3. 애니메이션
- [ ] slideInError 애니메이션
- [ ] checkmark 애니메이션
- [ ] shake (invalid) 효과
- [ ] fadeOut 애니메이션
- [ ] 로딩 스피너

### 4. 반응형 디자인
- [ ] 모바일 CSS (최대 768px)
- [ ] 데스크톱 CSS (1024px+)
- [ ] Touch 이벤트 처리
- [ ] 키보드 네비게이션

### 5. 접근성
- [ ] ARIA 레이블 추가
- [ ] aria-invalid 상태
- [ ] aria-describedby 링크
- [ ] role="alert" 에러 메시지
- [ ] 스크린 리더 테스트

### 6. 예제 & 테스트
- [ ] complete-form.fl 전체 통합 예제
- [ ] ui-integration.test.fl 테스트
- [ ] 성능 벤치마크
- [ ] 크로스 브라우저 테스트

---

## 🎯 예상 결과

```freelang
// Phase 3 후: 한 줄로 완성!
let form = SignupForm()
UI.auto_form(bind: form).build()

// 자동으로:
// ✅ 타입별 UI 생성
// ✅ 실시간 검증
// ✅ 에러 애니메이션
// ✅ 모바일 반응형
// ✅ 접근성 완비
```

---

**기록이 증명이다.**
