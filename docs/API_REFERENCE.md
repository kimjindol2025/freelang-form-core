# API Reference - Direct-Bind Form Engine v1.0.0

> Complete API documentation for direct-bind-ui package

---

## Table of Contents

1. [Core APIs](#core-apis)
2. [Validation APIs](#validation-apis)
3. [Async APIs](#async-apis)
4. [UI APIs](#ui-apis)
5. [Performance APIs](#performance-apis)
6. [i18n APIs](#i18n-apis)

---

## Core APIs

### FormStruct<T> Interface

Base interface for all form structures.

```freelang
interface FormStruct<T> {
    // 필드 메타데이터 반환
    fn get_field_metadata() -> [FieldMetadata],

    // 더티 비트 초기화
    fn reset_dirty_bits(),

    // 변경된 필드 조회
    fn get_modified_fields() -> [string],

    // 필드 값 설정 (더티 비트 자동 갱신)
    fn set_field(field_name: string, value: any)
}
```

### FieldMetadata

필드의 메타데이터.

```freelang
struct FieldMetadata {
    name: string,                          // 필드명
    type: string,                          // 타입 (string/int/bool/...)
    required: bool = false,                // 필수 여부
    constraints: [FieldConstraint] = [],   // 제약 조건 배열
    display_name: string? = null           // UI 표시명
}
```

### FieldConstraint

제약 조건 정의.

```freelang
struct FieldConstraint {
    type: string,                  // "required", "min", "max", "email", "match", "range", "custom"
    value: any? = null            // 제약 값 (min=5, pattern="...", etc)
}
```

### ValidationResult

검증 결과.

```freelang
struct ValidationResult {
    is_valid: bool,                       // 성공 여부
    errors: [ValidationError] = [],       // 오류 배열
    field_count: int? = null,            // 검증한 필드 수
    execution_time_ms: i64? = null       // 실행 시간
}
```

### ValidationError

검증 오류.

```freelang
struct ValidationError {
    field: string,                  // 필드명
    code: string,                   // 오류 코드 (REQUIRED, MIN_LENGTH, EMAIL, ...)
    message: string,                // 오류 메시지
    value: any? = null             // 잘못된 값
}
```

---

## Validation APIs

### FormValidator<T>

동기 폼 검증.

```freelang
struct FormValidator<T> {
    // 폼 검증 (동기)
    fn validate(form: T) -> ValidationResult

    // 단일 필드 검증
    fn validate_field(field_name: string, value: any) -> ValidationResult

    // 언어 설정 (ko/en/ja/zh/es)
    fn set_language(lang: string)

    // 커스텀 검증 함수 등록
    fn register_validator(field: string, validator: fn(any) -> bool)

    // 캐시 설정
    fn set_cache_ttl(ms: i64)

    // 캐시 초기화
    fn clear_cache()
}
```

#### 사용 예

```freelang
struct LoginForm {
    @required @email
    email: string,
    @required @min(6)
    password: string
}

let form = LoginForm{ email: "test@example.com", password: "pass123" }
let validator = FormValidator<LoginForm>{}
let result = validator.validate(form)

if !result.is_valid {
    for error in result.errors {
        print("❌ " + error.field + ": " + error.message)
    }
}
```

### FieldValidator

단일 필드 검증.

```freelang
struct FieldValidator {
    field_name: string,
    field_index: int,
    type: string,
    constraints: [FieldConstraint] = [],
    current_value: any? = null,
    last_error: ValidationError? = null,

    // 필드 검증 실행
    fn validate() -> ValidationResult,

    // 마지막 오류 조회
    fn get_last_error() -> ValidationError?
}
```

---

## Async APIs

### AsyncFormValidator<T>

비동기 폼 검증 (DB/API 호출).

```freelang
struct AsyncFormValidator<T> {
    // 비동기 검증 실행
    fn validate_async(form: T) -> ValidationResult,

    // 단일 필드 비동기 검증
    fn validate_field_async(field: string, value: any) -> ValidationResult,

    // 캐시 설정 (기본: 5초)
    fn set_cache_ttl(ms: i64),

    // 캐시 상태 조회
    fn get_cache_stats() -> map<string, any>,

    // 진행 중 콜백 설정
    fn on_validating(callback: fn(field_name: string)),

    // 완료 콜백 설정
    fn on_validated(callback: fn(result: ValidationResult))
}
```

#### 사용 예

```freelang
let form = SignupForm{
    username: "john_doe",
    email: "john@example.com"
}

let async_validator = AsyncFormValidator<SignupForm>{}

async_validator.on_validating(fn(field_name) {
    print("🔄 검증 중: " + field_name)
})

async_validator.on_validated(fn(result) {
    if result.is_valid {
        print("✅ 모든 검증 통과")
    } else {
        print("❌ 검증 실패: " + result.errors[0].message)
    }
})

let result = async_validator.validate_async(form)
```

### CachedAsyncValidator

캐싱된 비동기 검증.

```freelang
struct CachedAsyncValidator {
    // 캐시 정보 조회
    fn get_cache_info() -> map<string, any>,

    // 특정 필드 캐시 삭제
    fn invalidate_field(field: string),

    // 전체 캐시 삭제
    fn clear_all()
}
```

---

## UI APIs

### AutoFormBuilder<T>

자동 폼 UI 생성.

```freelang
struct AutoFormBuilder<T> {
    // 제목 설정
    fn with_title(title: string) -> AutoFormBuilder<T>,

    // 제출 버튼 텍스트 설정
    fn with_submit_text(text: string) -> AutoFormBuilder<T>,

    // 언어 설정 (ko/en/ja/zh/es)
    fn with_language(lang: string) -> AutoFormBuilder<T>,

    // 테마 설정 (light/dark)
    fn with_theme(theme: string) -> AutoFormBuilder<T>,

    // 반응형 활성화
    fn with_responsive(enabled: bool) -> AutoFormBuilder<T>,

    // CSS 클래스 추가
    fn with_custom_css(css: string) -> AutoFormBuilder<T>,

    // 폼 빌드
    fn build() -> string
}
```

#### 사용 예

```freelang
let form_html = AutoFormBuilder<UserForm>{}
    .with_title("사용자 등록")
    .with_submit_text("가입하기")
    .with_language("ko")
    .with_theme("dark")
    .with_responsive(true)
    .build()

// HTML 렌더링
print(form_html)
```

### RealtimeFormValidator<T>

실시간 폼 검증 (UI 피드백).

```freelang
struct RealtimeFormValidator<T> {
    // 필드 변경 감지
    fn on_field_change(
        field: string,
        value: any,
        callbacks: RealtimeCallbacks
    ),

    // 모든 필드 변경 감지
    fn on_form_change(
        form: T,
        callbacks: RealtimeCallbacks
    ),

    // 검증 딜레이 설정 (기본: 300ms)
    fn set_debounce_delay(ms: i64),

    // 로딩 상태 표시
    fn show_loading(field: string),

    // 오류 표시
    fn show_error(field: string, message: string),

    // 성공 표시
    fn show_success(field: string)
}
```

```freelang
struct RealtimeCallbacks {
    on_validate: fn(result: ValidationResult)?,
    on_loading: fn()?,
    on_error: fn(error: ValidationError)?,
    on_success: fn()?
}
```

#### 사용 예

```freelang
let realtime = RealtimeFormValidator<SignupForm>{}

realtime.on_field_change("email", "test@example.com", {
    on_validate: fn(result) {
        if result.is_valid {
            realtime.show_success("email")
        } else {
            realtime.show_error("email", result.errors[0].message)
        }
    }
})
```

---

## Performance APIs

### SIMDBatchValidator

SIMD 벡터 최적화 검증 (4x 병렬).

```freelang
struct SIMDBatchValidator {
    // 배치 검증 (SIMD 4x)
    fn validate_batch(
        fields: [FieldMetadata],
        values: [any]
    ) -> [ValidationResult],

    // 단일 필드 검증
    fn validate_single(
        field: FieldMetadata,
        value: any
    ) -> ValidationResult
}
```

#### 사용 예

```freelang
let fields = [...]  // 100+ 필드
let values = [...]

let simd = SIMDBatchValidator{}
let start = __current_time_ms()

let results = simd.validate_batch(fields, values)

let elapsed = __current_time_ms() - start
print("SIMD 완료: " + elapsed + "ms")  // ~7ms (vs 순차 25ms)
```

### MemoryPool

메모리 풀 (GC 제거).

```freelang
struct MemoryPool {
    // ValidationResult 할당
    fn acquire_result() -> ValidationResult,

    // ValidationResult 반환
    fn release_result(result: ValidationResult),

    // ValidationError 할당
    fn acquire_error() -> ValidationError,

    // ValidationError 반환
    fn release_error(error: ValidationError),

    // 풀 통계 조회
    fn get_stats() -> map<string, any>
}
```

#### 사용 예

```freelang
let pool = MemoryPool{}

for i in 0..1000 {
    let result = pool.acquire_result()
    // 사용...
    pool.release_result(result)
}

let stats = pool.get_stats()
print("재사용된 객체: " + stats["total_reused"])  // 1000
```

### JITCompiler

JIT 런타임 최적화.

```freelang
struct JITCompiler {
    // 검증 기록
    fn record_validation(
        field_name: string,
        field_type: string,
        time_ms: i64
    ),

    // Hotspot 필드 조회
    fn get_hotspot_fields() -> [string],

    // 컴파일 통계
    fn get_stats() -> map<string, any>,

    // 통계 출력
    fn print_stats()
}
```

#### 사용 예

```freelang
let jit = JITCompiler{}

// 첫 9회: 인터프리터 (2ms/회)
for i in 0..9 {
    jit.record_validation("username", "string", 2)
}

// 10회: JIT 자동 컴파일
jit.record_validation("username", "string", 2)

// 11회+: 컴파일된 코드 (0.2ms/회)
for i in 11..20 {
    jit.record_validation("username", "string", 0.2)
}

jit.print_stats()
```

---

## i18n APIs

### I18nFormatter

국제화 메시지 포매팅.

```freelang
struct I18nFormatter {
    // 메시지 설정
    fn set_language(lang: string),  // ko/en/ja/zh/es

    // 메시지 포매팅
    fn format(key: string, params: map<string, any>?) -> string
}
```

#### 지원 언어

| 언어 | 코드 | 상태 |
|------|------|------|
| 한국어 | ko | ✅ |
| English | en | ✅ |
| 日本語 | ja | ✅ |
| 中文 | zh | ✅ |
| Español | es | ✅ |

#### 사용 예

```freelang
let i18n = I18nFormatter{}
i18n.set_language("ja")

let msg = i18n.format("error_required", { "field": "email" })
// → "メール フィールドは必須です"

i18n.set_language("ko")
let msg_ko = i18n.format("error_email")
// → "올바른 이메일 형식이 아닙니다"
```

---

## Type Definitions

### 제약 조건 타입

```freelang
enum ConstraintType {
    REQUIRED = "required",
    MIN = "min",
    MAX = "max",
    RANGE = "range",
    EMAIL = "email",
    MATCH = "match",
    POSITIVE = "positive",
    INTEGER = "integer",
    CUSTOM = "custom"
}
```

### 오류 코드

```freelang
enum ValidationErrorCode {
    REQUIRED = "REQUIRED",
    MIN_LENGTH = "MIN_LENGTH",
    MAX_LENGTH = "MAX_LENGTH",
    MIN_VALUE = "MIN_VALUE",
    MAX_VALUE = "MAX_VALUE",
    EMAIL = "EMAIL",
    PATTERN = "PATTERN",
    POSITIVE = "POSITIVE",
    INTEGER = "INTEGER",
    CUSTOM = "CUSTOM"
}
```

---

## Examples

### 전체 예제

```freelang
// 1. 폼 정의
struct RegistrationForm {
    @required @min(3) @max(20)
    username: string,

    @required @email
    email: string,

    @required @match("^(?=.*[A-Z])(?=.*[0-9]).{8,}$")
    password: string,

    @required @match("^(?=.*[A-Z])(?=.*[0-9]).{8,}$")
    password_confirm: string,

    @range(18, 100)
    age: int,

    @custom("validate_terms")
    terms_accepted: bool
}

// 2. 커스텀 검증
fn validate_terms(value: any) -> bool {
    return value == true
}

// 3. 폼 인스턴스
let form = RegistrationForm{
    username: "john_doe",
    email: "john@example.com",
    password: "Password123",
    password_confirm: "Password123",
    age: 25,
    terms_accepted: true
}

// 4. 검증
let validator = FormValidator<RegistrationForm>{}
let result = validator.validate(form)

// 5. 결과 처리
if result.is_valid {
    print("✅ 등록 완료!")
} else {
    for error in result.errors {
        print("❌ " + error.field + ": " + error.message)
    }
}

// 6. 자동 UI (옵션)
let ui = AutoFormBuilder<RegistrationForm>{}
    .with_title("회원가입")
    .with_language("ko")
    .build()
```

---

**최종 업데이트**: 2026-03-08
**버전**: v1.0.0
**라이선스**: MIT
