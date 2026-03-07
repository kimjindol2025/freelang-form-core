# Phase 2: 컴파일러 매크로 + 고급 검증

**날짜**: 2026-03-08
**상태**: 설계 (구현 준비)
**이전**: Phase 1 완료 (f3c408b)

---

## 목표

Phase 1에서 기본 검증을 구현했으므로, Phase 2는 다음을 추가합니다:

1. **Validator-Generator 매크로**: 컴파일러가 검증 함수 자동 생성
2. **@match(regex)**: 정규식 패턴 검증
3. **@custom(fn)**: 사용자 정의 + 비동기 검증
4. **i18n**: 다국어 에러 메시지 (한글/영어/일본어)
5. **테스트 스위트**: 모든 검증 시나리오 커버

---

## 1️⃣ Validator-Generator 매크로

### 현재 (Phase 1) vs 목표 (Phase 2)

**Phase 1**: 수동으로 검증 함수 작성
```freelang
struct SignupForm {
    @required @email
    email: string
}

// 개발자가 매번 작성
fn validate_email_required() { ... }
fn validate_email() { ... }
```

**Phase 2**: 컴파일러가 자동 생성
```freelang
struct SignupForm {
    @required @email
    email: string
}

// 컴파일러 자동 생성:
// __validate_email_required()
// __validate_email()
// __register_email_validators()
```

### 구현 전략

**FreeLang 컴파일러에 추가할 패치**:
```
1. 어노테이션 파서 (이미 Phase 1에서 설계)
2. Validator-Generator (Phase 2 - 신규)
3. 메타데이터 등록 (Phase 2 - 신규)
```

**컴파일 파이프라인**:
```
파싱(Parsing)
  ↓
어노테이션 추출 (@required, @email, @match, @custom)
  ↓
Validator-Generator 실행 ← Phase 2 신규
  ├─ 각 어노테이션별 검증 함수 생성
  ├─ 메타데이터 구조 생성
  └─ 런타임 레지스트리 등록
  ↓
IR 생성
  ↓
최적화 & 컴파일
```

### Generator 코드 예시

**입력**:
```freelang
struct User {
    @required @email @match("^[a-z]+@[a-z]+\\.[a-z]{2,}$")
    email: string,

    @required @custom(is_unique_username)
    username: string
}
```

**컴파일러가 생성하는 코드**:
```freelang
// 생성된 검증 함수들 (자동)
fn __validate_User_email_required(value: string) -> bool {
    return value != null && value != ""
}

fn __validate_User_email_email(value: string) -> bool {
    return value.contains("@") && value.contains(".")
}

fn __validate_User_email_match(value: string) -> bool {
    return regex_match(value, "^[a-z]+@[a-z]+\\.[a-z]{2,}$")
}

fn __validate_User_username_required(value: string) -> bool {
    return value != null && value != ""
}

// 메타데이터 구조 (자동)
let User_validators = [
    FieldValidator{
        field_name: "email",
        validators: [
            __validate_User_email_required,
            __validate_User_email_email,
            __validate_User_email_match
        ]
    },
    FieldValidator{
        field_name: "username",
        validators: [
            __validate_User_username_required,
            is_unique_username
        ]
    }
]

// 런타임 등록 (자동)
fn __register_User_validators() {
    ValidatorRegistry.register("User", User_validators)
}
```

---

## 2️⃣ @match (정규식) 검증

### 구현

**단계 1**: 정규식 엔진 추가
```freelang
// src/core/Regex.fl (신규)
struct RegexPattern {
    pattern: string,
    compiled: any  // 컴파일된 정규식 (런타임 최적화)
}

fn regex_compile(pattern: string) -> RegexPattern { ... }
fn regex_match(pattern: RegexPattern, value: string) -> bool { ... }
fn regex_replace(pattern: RegexPattern, value: string, replacement: string) -> string { ... }
```

**단계 2**: @match 검증 구현
```freelang
// src/core/Validator.fl 확장

fn validate_match_pattern(value: string, pattern: string) -> ValidationError? {
    let regex = regex_compile(pattern)
    if !regex_match(regex, value) {
        return ValidationError{
            field: field_name,
            code: "PATTERN",
            message: "올바른 형식이 아닙니다"
        }
    }
    return null
}
```

### 사용 예시

**전화번호**:
```freelang
struct PhoneForm {
    @required @match("^01[0-9]-[0-9]{3,4}-[0-9]{4}$")
    phone: string  // "010-1234-5678"
}
```

**비밀번호 (숫자+문자+특수문자)**:
```freelang
struct PasswordForm {
    @required @min(8) @match("^(?=.*[A-Za-z])(?=.*[0-9])(?=.*[@$!%*#?&])[A-Za-z0-9@$!%*#?&]{8,}$")
    password: string
}
```

**IPv4 주소**:
```freelang
struct NetworkForm {
    @match("^((25[0-5]|(2[0-4]|1[0-9])?[0-9])\\.?\\b){4}$")
    ip_address: string
}
```

---

## 3️⃣ @custom (사용자 정의) 검증

### 비동기 검증 지원

**Phase 1**: 동기만 지원
```freelang
@custom(is_username_unique)  // 즉시 실행
```

**Phase 2**: 비동기 지원
```freelang
@custom(async_check_username)  // 비동기 실행 (DB 조회)
@custom(check_email_domain)    // 외부 API 호출
```

### 구현

```freelang
// src/core/AsyncValidator.fl (신규)

// 비동기 검증 결과
struct AsyncValidationResult {
    is_valid: bool,
    error: ValidationError?,
    pending: bool  // 검증 진행 중
}

// 비동기 검증 함수 타입
type AsyncValidator<T> = fn(value: T) -> AsyncValidationResult

fn validate_async(validators: [AsyncValidator], value: any) -> AsyncValidationResult {
    // 병렬 실행 가능
    // Promise.all 스타일
}
```

### 사용 예시

**DB에서 중복 확인**:
```freelang
// 비동기 검증 함수 정의
fn async_check_username(username: string) -> bool {
    // 비동기: DB 조회
    let exists = await db.user_exists(username)
    return !exists
}

struct SignupForm {
    @required @min(3) @custom(async_check_username)
    username: string
}

// UI에서:
form.on_field_change("username", async fn(value) {
    let result = await form.validate_field_async("username")
    if !result.is_valid {
        show_error("이미 사용 중인 사용자명입니다")
    }
})
```

**이메일 도메인 확인**:
```freelang
fn check_email_domain(email: string) -> bool {
    let domain = email.split("@")[1]
    // DNS 조회
    let mx_records = await dns.lookup_mx(domain)
    return mx_records.length > 0
}

struct ContactForm {
    @required @email @custom(check_email_domain)
    email: string
}
```

---

## 4️⃣ i18n (다국어 지원)

### 에러 메시지 로컬라이제이션

**현재 (Phase 1)**:
```freelang
ERROR_MESSAGES = {
    "required": {
        "ko": "필수 필드입니다",
        "en": "This field is required"
    }
}
```

**Phase 2 목표**: 완전한 i18n 지원

**구조**:
```freelang
// src/i18n/messages.fl (신규)

struct I18nMessage {
    code: string,      // "REQUIRED", "EMAIL", "MIN_LENGTH"
    locale: string,    // "ko", "en", "ja"
    message: string,   // 메시지
    params: map<string, any>?  // {min: 3, max: 20}
}

map<string, map<string, string>> MESSAGES = {
    "ko": {
        "required": "필수 필드입니다",
        "email": "올바른 이메일이 아닙니다",
        "min_length": "최소 {min}자 이상",
        "max_length": "최대 {max}자 이하",
        "pattern": "올바른 형식이 아닙니다",
        "range": "{min}에서 {max} 사이"
    },
    "en": {
        "required": "This field is required",
        "email": "Invalid email address",
        "min_length": "Minimum {min} characters",
        "max_length": "Maximum {max} characters",
        "pattern": "Invalid format",
        "range": "Must be between {min} and {max}"
    },
    "ja": {
        "required": "必須フィールドです",
        "email": "有効なメールアドレスではありません",
        "min_length": "最小{min}文字以上",
        "max_length": "最大{max}文字以下",
        "pattern": "無効な形式です",
        "range": "{min}から{max}の間"
    }
}

// 포매팅
fn format_message(code: string, locale: string, params: map<string, any>?) -> string {
    let template = MESSAGES[locale][code]
    if params != null {
        for (key, value) in params {
            template = template.replace("{" + key + "}", value as string)
        }
    }
    return template
}
```

### 사용 방법

```freelang
fn render_form(locale: string = "ko") {
    let form = SignupForm()
    let form_ui = UI.form(bind: form, locale: locale)
        .input("username")
        .email("email")
        .on_submit(fn() {
            let result = form.validate()
            if !result.is_valid {
                for error in result.errors {
                    let message = format_message(error.code, locale)
                    show_error(message)
                }
            }
        })
        .build()
}
```

---

## 5️⃣ 테스트 스위트

### 테스트 파일 구조

```
tests/
├── validators.test.fl       # 검증 로직 단위 테스트
├── async-validators.test.fl # 비동기 검증 테스트
├── regex.test.fl            # 정규식 테스트
├── i18n.test.fl             # 다국어 메시지 테스트
└── integration.test.fl      # 통합 테스트
```

### 테스트 예시

**유효한 이메일**:
```freelang
fn test_valid_email() {
    let validator = FieldValidator{
        field_name: "email",
        constraints: [
            FieldConstraint{ type: "required", value: null },
            FieldConstraint{ type: "email", value: null }
        ]
    }

    validator.current_value = "user@example.com"
    let result = validator.validate()
    assert result.is_valid == true
    assert result.errors.length == 0
}
```

**정규식 매칭**:
```freelang
fn test_phone_regex() {
    let validator = FieldValidator{
        field_name: "phone",
        constraints: [
            FieldConstraint{ type: "match", value: "^01[0-9]-[0-9]{3,4}-[0-9]{4}$" }
        ]
    }

    // 유효
    validator.current_value = "010-1234-5678"
    assert validator.validate().is_valid == true

    // 무효
    validator.current_value = "12345"
    assert validator.validate().is_valid == false
}
```

**비동기 검증**:
```freelang
async fn test_async_username_check() {
    let result = await async_check_username("john_doe")
    assert result == true  // 사용 가능

    let result2 = await async_check_username("admin")  // 예약어
    assert result2 == false  // 사용 불가
}
```

**i18n 메시지**:
```freelang
fn test_messages_ko() {
    let ko_msg = format_message("required", "ko")
    assert ko_msg == "필수 필드입니다"
}

fn test_messages_en() {
    let en_msg = format_message("required", "en")
    assert en_msg == "This field is required"
}

fn test_message_with_params() {
    let msg = format_message("min_length", "ko", {"min": 3})
    assert msg == "최소 3자 이상"
}
```

---

## 📋 Phase 2 구현 체크리스트

### 1. 코어 구현
- [ ] `src/core/Regex.fl` - 정규식 엔진
- [ ] `src/core/AsyncValidator.fl` - 비동기 검증
- [ ] `src/core/Validator.fl` 확장 - @match, @custom 추가
- [ ] `src/i18n/messages.fl` - 다국어 메시지

### 2. 컴파일러 매크로
- [ ] FreeLang 컴파일러에 Validator-Generator 추가
- [ ] 어노테이션 → 검증 함수 자동 생성
- [ ] 메타데이터 자동 등록

### 3. 예제
- [ ] `examples/advanced-validators.fl` - @match/@custom 예제
- [ ] `examples/async-form.fl` - 비동기 검증 예제
- [ ] `examples/multilang-form.fl` - i18n 예제

### 4. 테스트
- [ ] `tests/validators.test.fl` - 검증 단위 테스트
- [ ] `tests/async-validators.test.fl` - 비동기 테스트
- [ ] `tests/regex.test.fl` - 정규식 테스트
- [ ] `tests/i18n.test.fl` - 다국어 테스트
- [ ] `tests/integration.test.fl` - 통합 테스트

### 5. 문서
- [ ] `docs/REGEX_GUIDE.md` - 정규식 가이드
- [ ] `docs/ASYNC_VALIDATION.md` - 비동기 검증 가이드
- [ ] `docs/I18N_GUIDE.md` - 다국어 지원 가이드
- [ ] `docs/API_REFERENCE.md` - API 참조

### 6. Gogs
- [ ] 커밋: Phase 2 완료

---

## 🎯 성능 목표 (Phase 2)

| 메트릭 | Phase 1 | Phase 2 목표 |
|--------|--------|------------|
| 필드 입력 검증 | <1ms | <1ms (비정규식) |
| 정규식 검증 | - | <0.5ms |
| 비동기 검증 | - | <100ms (DB 포함) |
| i18n 포매팅 | - | <0.1ms |
| 메모리 오버헤드 | <100B/필드 | <150B/필드 |

---

## 📅 일정

| 항목 | 예정 |
|------|------|
| **설계** | ✅ 완료 |
| **구현** | 진행 중 |
| **테스트** | 3월 8일 |
| **Gogs 커밋** | 3월 8일 |
| **완료** | 3월 8일 22시 예정 |

---

**기록이 증명이다.**
