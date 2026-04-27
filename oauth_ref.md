# 소상공인(카카오 기반) OAuth 설계 정리

## 개요

1. **현재 상황**
    - 카카오 콜백 → 전화번호 인증된 사용자에게만 토큰 발급, 그 외엔 null + 메시지

2. **구조적 문제**
    - 가입은 시작했으나 끝낼 방법이 없어 보인다
    - 상태 표현 부재

3. **설계 방향**
    - AuthStatusEvaluator로 발급 조건 평가 책임 집중
    - TempToken 신규 추가 
    - 인증 완료 시점에서 조건 충족 여부 확인 후 토큰 발급
    - `authStatus`로 상태만 표현

4. **이점**
    - 가입 경로 모호 해소
    - 책임 경계 정리
    - 프론트 경량화

---

## 현재 상황

사용자가 카카오 로그인 버튼을 누르면, 서버는 카카오로부터 받은 코드로 사용자 정보를 조회하고 우리 DB에서 해당 유저를 찾거나 새로 만든다.

문제는 그 다음이다. **전화번호 인증이 완료된 사용자에게만 토큰을 발급**하고, 아직 인증이 안 된 신규 유저에게는 `accessToken=null`, `refreshToken=null`에 메시지만 돌려준다.

---

## 구조적 문제

지금 구조에서 신규 유저가 가입을 끝낼 방법이 없다.

카카오 로그인을 통과해도 서버는 토큰을 주지 않는다. 그런데 전화번호 인증 API나 약관 동의 API를 호출하려면 토큰이 있어야 한다. 결국 **가입은 시작됐지만 끝낼 수 없는 상태**가 된다.

여기에 추가로 엮인 문제들이 있다.

### 상태를 알 수 없다  
 - 지금 응답만 봐서는 "현재 어떤 상태인지"를 프론트가 파악하기 어렵다. Flutter 앱 쪽에서 직접 분기 로직을 만들어야 하는 상황이다.
 - 현재 `LoginUsecase`는 내부적으로 `message` 필드를 통해 "왜 토큰이 발급되지 않았는지" 이유를 담아서 반환한다. 
 - 실제 응답 DTO인 `LoginApiResponse`에는 `accessToken`, `refreshToken` 두 필드만 있고 `message`는 아예 매핑이 안 되어 있다.


### 약관 재동의가 고려되지 않았다  
 - 현재는 "전화번호 인증 완료"만 조건으로 보고 있다. 나중에 약관이 업데이트되면 기존 회원도 다시 동의해야 하는데, 이걸 처리할 구조가 없다.

---

## 설계 방향

### AuthFacade - 인증 처리 퍼사드 `▶ 신규`

- 위치: `cases/auth/`
- 책임: `AuthStatusEvaluator`(상태 판정)와 `JwtService`(토큰 생성)를 조합해 인증 처리를 한 번에 수행한다.
- 반환: `AuthResult` (토큰과 `authStatus` 포함)
- 호출자: Usecase가 각자 작업을 처리한 직후, `AuthFacade`로 발급 조건을 평가하고 토큰을 발급하도록 위임한다.
- UseCase는 `AuthFacade` 하나만 호출하면 상태와 토큰을 동시에 받을 수 있다.
- `AuthStatusEvaluator`와 `JwtService`는 각자의 역할만 유지한 채 오염되지 않는다.

### AuthStatusEvaluator - 인증 상태 평가
- 위치: services/auth/ (or cases/auth/ 공통)
- 입력: User
- 반환: AuthStatus enum (CONSENT_REQUIRED, PHONE_VERIFICATION_REQUIRED, COMPLETED)
- 책임: "토큰 발급 가능 여부"를 한 곳에서 판단한다. 발급 조건이 변경되더라도 이 클래스만 수정하면 된다
- 토큰 생성에는 관여하지 않는다.
- 상태 판정만 책임진다.

### TempToken - 임시 토큰

- 카카오 인증을 통과한 신규 유저나 재동의가 필요한 기존 유저에게 **15분짜리 임시 토큰(TempToken)** 을 먼저 발급한다.
- 카카오 로그인 직후 LoginUseCase도 AuthFacade를 호출한다. 
- AuthFacade는 평가 결과에 따라 TempToken 또는 정식 토큰을 발급한다.
- 그 외에는 정식 토큰을 발급한다.

#### 이 토큰을 활용 가능한 상황 제한
- 약관 동의 API 호출
- 전화번호 인증 API 호출


#### 클라이언트가 토큰을 받으러 API를 한 번 더 호출할 필요가 없다.
 - 정식 토큰은 각 UseCase가 완료되는 시점에 AuthStatusEvaluator로 조건을 확인하고, 충족되면 그 자리에서 바로 발급된다.

### 각 인증 완료 시점에서 즉시 발급
 - 동의 처리, 전화번호 인증 각 UseCase는 자신의 처리를 끝낸 직후 `AuthFacade`를 호출한다.
 - `AuthFacade`가 `AuthResult`(authStatus + 토큰)를 반환하므로 UseCase는 그 결과를 응답에 담기만 하면 된다.
 - UseCase가 `JwtService`를 직접 호출하지 않아도 된다.

발급 조건은 하나로 통일된다.
> 전화번호 인증 완료 **AND** 모든 필수 약관 동의

신규 가입, 약관 재동의, 정상 재로그인 — 세 가지 케이스가 전부 이 조건 하나로 판정된다.

### authStatus — 백엔드 & 프론트 간 역할 분리

모든 관련 응답에 `authStatus` 필드가 포함된다.

| 값 | 의미 |
|---|---|
| `CONSENT_REQUIRED` | 아직 동의 안 한 필수 약관이 있다 |
| `PHONE_VERIFICATION_REQUIRED` | 전화번호 인증이 필요하다 |
| `COMPLETED` | 정식 토큰 발급 조건을 모두 충족했다 |

서버는 "지금 어떤 상태인지"만 알려준다. "어떤 화면을 보여줄지"는 Flutter 앱이 결정한다.

---

## 이점

 - 신규 유저가 "가입은 시작했지만 끝낼 방법이 없는" 상태가 사라진다. 임시 토큰으로 동의와 인증을 순서대로 진행할 수 있다.
 - `AuthStatusEvaluator`는 상태 판정만, `JwtService`는 토큰 생성만, `AuthFacade`는 조합만 담당한다.
 - 각자 역할이 명확하다.
 - `authStatus` enum 덕분에 현재 가입 단계가 어디인지 서버에서 명시적으로 관리된다. 약관 업데이트로 인한 재동의 같은 케이스도 별도 처리 없이 자연스럽게 커버된다.
 - 상태 판정을 서버가 하니까 Flutter 앱은 `authStatus` 값에 따라 화면만 바꿔주면 된다.

### 단점
 - 토큰 발급 트리거가 여러 UseCase에 분산되어 있어, 새로운 인증 단계가 추가될 때 해당 UseCase에도 `AuthFacade` 호출을 추가해야 한다.

---

## 플로우 다이어그램

### 신규 가입

```
카카오 로그인
    │
    ▼
AuthFacade 호출 → 조건 미충족
    │
    ▼
임시 토큰 발급 (authStatus: CONSENT_REQUIRED)
    │
    ▼
약관 동의 × N
    │  AuthFacade 호출 → 조건 미충족
    ▼
전화번호 인증 (SMS 발송 → 코드 입력)
    │  AuthFacade 호출 → 조건 충족
    ▼
정식 토큰 즉시 발급 (authStatus: COMPLETED)
```

### 정상 재로그인

```
카카오 로그인
    │
    ▼
AuthFacade 호출 → 조건 즉시 충족
    │
    ▼
정식 토큰 발급 (authStatus: COMPLETED)
```

### 약관 재동의 필요 (기존 회원)

```
카카오 로그인
    │
    ▼
AuthFacade 호출 → 조건 미충족
    │
    ▼
임시 토큰 발급 (authStatus: CONSENT_REQUIRED)
    │
    ▼
약관 동의 × N
    │  AuthFacade 호출 → 전화번호 인증 이미 완료 → 조건 충족
    ▼
정식 토큰 즉시 발급 (authStatus: COMPLETED)
```