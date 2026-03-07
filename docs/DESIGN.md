# Phase 1: Direct-Bind Form Engine 설계 문서

**날짜**: 2026-03-08
**상태**: 초안 (검토 대기)
**저자**: Claude (Architect)

---

## 목차

1. [문제 정의](#문제-정의)
2. [솔루션 개요](#솔루션-개요)
3. [기술 아키텍처](#기술-아키텍처)
4. [Phase 1 구현 범위](#phase-1-구현-범위)
5. [API 설계](#api-설계)
6. [성능 목표](#성능-목표)
7. [다음 단계](#다음-단계)

---

## 문제 정의

### 기존 폼 관리의 문제점

#### 1. 런타임 오버헤드
```javascript
// Formik 스타일 (불필요한 렌더링)
const formik = useFormik({
  initialValues: { username: '' },
  validate: (values) => {
    // ← 매 입력마다 실행 (O(n) 검증)
  },
  onSubmit: async (values) => {}
});

// 결과: 입력 1회 = 렌더링 + 검증 2회 이상
```

#### 2. 타입 안전성 부재
- JavaScript: 런타임에만 타입 확인
- 배포 후 버그 발생 (검증 로직 누락 가능)

#### 3. 개발자 경험 악화
- 구조체 정의 + 검증 로직 + UI 컴포넌트 + 상태 관리 = 4배 코드
- 중복성 높음 (username을 3곳에서 정의)

---

## 솔루션 개요

### "Direct-Bind" 패러다임

**데이터 정의 시점에 검증을 포함시킨다.**

```freelang
// 1단계: 검증이 포함된 구조체 정의
struct SignupForm {
    @required @min(3)
    username: string
}

// 2단계: UI는 구조체만 참조 (검증 자동 포함)
UI.form(bind: SignupForm) { ... }

// 3단계: 제출 시 검증 생략 (이미 정의 단계에서 확인)
if form.is_valid() { submit() }  // ← 이미 타입 안전함
```

### 핵심 이점

| 항목 | 기존 (Formik) | Direct-Bind |
|------|-------------|------------|
| **검증 위치** | 런타임 (O(n)) | 컴파일 타임 (O(1)) |
| **타입 안전** | ❌ | ✅ (FreeLang 강제) |
| **오버헤드** | 높음 (매 입력마다) | 0% (메타데이터 사용) |
| **코드 중복** | 높음 (4곳 정의) | 1곳 (구조체) |
| **메모리** | O(n) (모든 상태) | O(1) (Dirty-Bit) |

---

## 기술 아키텍처

### 1. 계층 구조

```
┌─────────────────────────────────────┐
│   User Application (signup.fl)      │
│  (폼 사용자 코드)                   │
└────────────┬────────────────────────┘
             │ (bind: SignupForm)
┌────────────▼────────────────────────┐
│  MOSS-UI Layer (FormRenderer)       │
│  (자동 UI 렌더링)                   │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│  Direct-Bind Core                   │
│  ├─ FormStruct (메타데이터)         │
│  ├─ Annotations (검증 규칙)         │
│  └─ Validator (O(1) 검증)           │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│  FreeLang Runtime                   │
│  (타입 안전성 강제)                 │
└─────────────────────────────────────┘
```

### 2. 어노테이션 체계

**Phase 1에서 구현할 어노테이션:**

```freeling
// 필수 제약
@required              // null/undefined 불허

// 문자열 제약
@min(n)                // 최소 길이 n
@max(n)                // 최대 길이 n
@email                 // 이메일 형식
@match(regex)          // 정규식 매칭
@phone                 // 전화번호 형식

// 숫자 제약
@range(min, max)       // min ≤ x ≤ max
@positive              // x > 0
@integer               // 정수만

// 커스텀 제약
@custom(fn)            // 사용자 정의 검증 함수
```

### 3. 메타데이터 생성 (컴파일 타임)

FreeLang 컴파일러가 다음을 자동 생성:

```freelang
// 원본 (개발자 작성)
struct SignupForm {
    @required @min(3) @email
    email: string
}

// 컴파일러 생성 (자동)
fn __validate_email(value: string) -> bool {
    // Phase 1: 정규식 검사
    return is_email(value)
}

// 메타데이터 (런타임)
FormMetadata<SignupForm> {
    fields: [
        {
            name: "email",
            type: "string",
            validators: [
                { type: REQUIRED, fn: __validate_email_required },
                { type: EMAIL, fn: __validate_email }
            ]
        }
    ]
}
```

### 4. Dirty-Bit 추적

**변경된 필드만 추적하여 O(1) 비교:**

```freelang
struct SignupForm {
    email: string,          // Bit 0
    username: string,       // Bit 1
    password: string,       // Bit 2
    _dirty: u32             // ← 비트 플래그
}

// 사용자가 email 입력 시:
form._dirty |= (1 << 0)     // Bit 0 설정

// 검증 (O(1)):
for i in 0..32 {
    if (form._dirty & (1 << i)) != 0 {
        validate(fields[i])  // ← 변경된 것만!
    }
}
```

---

## Phase 1 구현 범위

### 파일 목록

#### 1. `src/core/FormStruct.fl`
- FormStruct 기본 인터페이스
- 필드 메타데이터 구조

```freelang
interface FormStruct {
    fn is_valid() -> bool
    fn get_errors() -> [string]
    fn get_values() -> map<string, any>
    fn reset()
}
```

#### 2. `src/core/Annotations.fl`
- @required, @min, @max, @email, @range, @match
- 어노테이션 파서

```freelang
// 메타데이터 구조
struct FieldConstraint {
    type: string,      // "required", "email", "range"
    value: any,        // min/max, pattern, etc
    message: string    // 에러 메시지
}
```

#### 3. `src/core/Validator.fl`
- O(1) 검증 로직
- Dirty-Bit 추적

```freelang
struct Validator {
    fn validate_field(field_name: string, value: any) -> ValidationResult
    fn validate_all() -> [ValidationError]
    fn mark_dirty(field_index: u32)
}

struct ValidationResult {
    is_valid: bool,
    errors: [string]
}
```

#### 4. `src/ui/MossBinding.fl`
- MOSS-UI와 Direct-Bind 연결

```freelang
fn form(bind: FormStruct) -> MossFormComponent {
    // UI 자동 생성
    // 각 필드 → UI.input 매핑
    // 검증 상태 → 에러 표시
}
```

#### 5. `src/ui/FormRenderer.fl`
- 폼 렌더링 엔진

```freelang
fn render_form_fields(form: FormStruct) -> [UIElement] {
    // 메타데이터 기반 자동 렌더링
}
```

#### 6. `examples/signup-form.fl`
- 실제 사용 예제

```freelang
struct SignupForm {
    @required @min(3) @max(20)
    username: string,

    @required @email
    email: string,

    @required @min(8)
    password: string
}

fn main() {
    let form = SignupForm();
    render_signup(form);
}
```

---

## API 설계

### FormStruct 인터페이스

```freelang
interface FormStruct<T> {
    // 검증
    fn is_valid() -> bool
    fn validate() -> [ValidationError]
    fn get_field_error(field: string) -> string?

    // 상태 관리
    fn get_values() -> T
    fn set_value(field: string, value: any)
    fn reset()

    // Dirty-Bit
    fn mark_dirty(field: string)
    fn is_field_dirty(field: string) -> bool
    fn get_dirty_fields() -> [string]
}

struct ValidationError {
    field: string,
    message: string,
    code: string  // "REQUIRED", "EMAIL", "MIN_LENGTH"
}
```

### Annotation API

```freelang
// 컴파일러 매크로 (개발자 사용)
@required              // 필수 필드
@min(n)                // 최소 길이/값
@max(n)                // 최대 길이/값
@email                 // 이메일
@match(regex)          // 정규식
@range(min, max)       // 범위
@custom(validator_fn)  // 커스텀

// 컴파일러 생성 (자동)
fn __validate_<field_name>() -> bool { ... }
```

---

## 성능 목표

### Phase 1 목표

| 메트릭 | 목표 |
|--------|------|
| **첫 렌더링** | < 5ms |
| **필드 입력** | < 1ms (Dirty-Bit 검증) |
| **폼 제출 검증** | < 2ms (10개 필드 기준) |
| **메모리 오버헤드** | < 100B per field |
| **번들 크기** | < 50KB (FreeLang 포함) |

### 성능 최적화 전략

1. **Dirty-Bit**: O(n) → O(1) 검증
2. **메타데이터 캐싱**: 런타임 오버헤드 제거
3. **Zero-Copy**: 필드 복사 생략 (원본 참조)
4. **컴파일 타임 최적화**: 검증 로직 사전 생성

---

## Phase 1 체크리스트

- [ ] `src/core/FormStruct.fl` 구현
- [ ] `src/core/Annotations.fl` 구현
- [ ] `src/core/Validator.fl` 구현 (Dirty-Bit)
- [ ] `src/ui/MossBinding.fl` 구현
- [ ] `examples/signup-form.fl` 동작 확인
- [ ] 단위 테스트 작성
- [ ] 성능 벤치마크 측정
- [ ] 문서화 완료
- [ ] Gogs 커밋

---

## 다음 단계

### Phase 2 (검증 매크로)
- 컴파일러 패치: Validator-Generator 추가
- @match, @custom 고급 검증
- 에러 메시지 국제화

### Phase 3 (MOSS-UI 통합)
- 자동 UI 렌더링
- 실시간 검증 피드백
- 에러 표시 스타일링

### Phase 4 (성능 최적화)
- SIMD 벡터 검증
- 메모리 풀 재사용
- 플랫폼별 컴파일 최적화

### Phase 5 (KPM 등록)
- 패키지 메타데이터 작성
- 버전 관리 (v1.0.0)
- KPM Registry 등록

---

## 참고

- **FreeLang 구조체**: [Spec Document]
- **MOSS-UI 바인딩**: [MOSS-UI API]
- **KPM 패키지**: [KPM Registry]

---

**기록이 증명이다.**
초안 작성: 2026-03-08 01:50 UTC
상태: Phase 1 구현 준비 완료
