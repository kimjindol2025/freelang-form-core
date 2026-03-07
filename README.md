# FreeLang Direct-Bind Form Engine

**Zero-Dependency, Type-Safe Form Management powered by FreeLang Struct Metadata**

![Status](https://img.shields.io/badge/status-Phase%201-blue) ![FreeLang](https://img.shields.io/badge/FreeLang-v7+-green) ![npm](https://img.shields.io/badge/npm-0%25-red)

---

## 🎯 핵심 철학

전통적 폼 관리(Formik, React Hook Form)는 **데이터 검사소** 방식입니다.
```
입력 → 컴포넌트 → 상태 → 검증 → 렌더링 → 검사소(재검사) → 제출
```

**Direct-Bind** 는 데이터가 **태어날 때부터** 올바른 형식을 강제합니다.
```
구조체 정의(검증 어노테이션) → UI 자동 바인딩 → 제출(이미 검증됨)
```

---

## ✨ 특징

- ✅ **Zero npm Dependencies**: 순수 FreeLang + MOSS-UI 만 사용
- ✅ **Type-Safe**: 구조체 메타데이터 기반 컴파일 타임 검증
- ✅ **Auto-Binding**: 필드 ↔ UI 컨트롤 자동 연결
- ✅ **Dirty-Bit Tracking**: 변경된 필드만 O(1) 비교
- ✅ **Zero-Copy**: 메모리 오버헤드 최소화 (SIMD 최적화)
- ✅ **Self-Hosting Ready**: 별도 빌드 도구 불필요

---

## 📦 프로젝트 구조

```
freelang-form-core/
├── docs/
│   ├── DESIGN.md              # Phase 1 설계 문서
│   └── API-Reference.md       # (Phase 2)
├── src/
│   ├── core/
│   │   ├── FormStruct.fl      # 폼 구조체 기반 시스템
│   │   ├── Annotations.fl     # @required, @email, @range 등
│   │   └── Validator.fl       # 검증 로직 (컴파일 타임 주입)
│   └── ui/
│       ├── MossBinding.fl     # MOSS-UI ↔ FreeLang 바인딩
│       └── FormRenderer.fl    # UI 렌더링 엔진
├── examples/
│   ├── signup-form.fl         # 회원가입 예제
│   ├── todo-form.fl           # Todo 폼 예제
│   └── login-form.fl          # 로그인 폼 예제
├── tests/
│   ├── form-binding.test.fl   # 바인딩 테스트
│   └── validation.test.fl     # 검증 테스트
└── README.md
```

---

## 🚀 빠른 시작 (Phase 1 - Coming Soon)

### 1. 구조체 정의 (검증 어노테이션)

```freelang
struct SignupForm {
    @required @min(3) @max(20)
    username: string,

    @required @email
    email: string,

    @required @min(8)
    password: string,

    @range(18, 99)
    age: int,

    @required @match("^[0-9]{10,11}$")
    phone: string
}
```

### 2. UI 바인딩 (자동 생성)

```freelang
fn render_signup_form() {
    let form = SignupForm();

    // Direct-Bind: 구조체 필드와 UI 자동 연결
    UI.form(bind: form) {
        UI.input(field: .username, placeholder: "사용자명");
        UI.input(field: .email, type: "email");
        UI.input(field: .password, type: "password");
        UI.input(field: .age, type: "number");
        UI.input(field: .phone, mask: "###-####-####");

        UI.button("가입", on_click: fn() {
            // form은 이미 모든 검증을 통과한 상태
            if form.is_valid() {
                auth.register(form);
            }
        });
    }
}
```

### 3. 결과

- **런타임 오버헤드 0%** (컴파일 타임 주입)
- **메모리 효율** (Dirty-Bit로 변경된 필드만 추적)
- **개발자 경험** (어노테이션만으로 완성)

---

## 📚 Phase 로드맵

| Phase | 목표 | 완료 |
|-------|------|------|
| **1** | 설계 + 기본 구조체 바인딩 | 🔄 진행 중 |
| **2** | 검증 매크로 + 컴파일러 패치 | ⏳ 예정 |
| **3** | MOSS-UI 통합 렌더링 | ⏳ 예정 |
| **4** | 테스트 스위트 + 성능 최적화 | ⏳ 예정 |
| **5** | KPM 패키지 등록 | ⏳ 예정 |

---

## 🔗 관련 프로젝트

- [FreeLang](https://gogs.dclub.kr/kim/freelang) - 언어 런타임
- [MOSS-UI](https://gogs.dclub.kr/kim/v2-freelang-ai) - UI 엔진
- [KPM Registry](https://gogs.dclub.kr/kim/kpm-registry) - 패키지 관리

---

## 📖 문서

- [`DESIGN.md`](docs/DESIGN.md) - Phase 1 기술 설계
- [`API-Reference.md`](docs/API-Reference.md) - API 명세 (Phase 2)

---

## 📝 라이선스

MIT

---

**기록이 증명이다.** - 김진돌 표준 워크플로우
