---
title: "Go 값 리시버 vs 포인터 리시버 — Java 개발자를 위한 설명"
date: 2026-06-04 18:00:00 +0900
categories: [Programming]
tags: [go, receiver, pointer]
---

Java만 쓰다가 Go를 처음 접하면 메서드 선언이 낯설게 느껴진다.

```go
func (r *PostgresUserRepository) FindByKakaoID(ctx context.Context, kakaoID string) (*domain.User, error) {
```

`(r *PostgresUserRepository)` 이건 뭔지, 왜 리턴값이 두 개인지, `*`는 왜 붙이는지. Java 배경으로 Go를 시작할 때 가장 먼저 막히는 부분들이다.

---

## 리시버(Receiver) — Go의 메서드 선언 방식

### Java와의 비교

Java는 클래스 블록 안에 메서드를 넣어서 소속을 표현한다.

```java
public class PostgresUserRepository {
    public User findByKakaoId(String kakaoId) {
        // this.pool 사용 가능
    }
}
```

Go에는 클래스가 없다. 대신 함수 앞에 **"이 타입에 붙인다"**고 선언한다.

```go
func (r *PostgresUserRepository) FindByKakaoID(ctx context.Context, kakaoID string) (*domain.User, error) {
//    ↑ 리시버
//    Java의 "this"에 해당, 이름을 r로 지은 것
}
```

`(r *PostgresUserRepository)` 부분을 **리시버(receiver)**라고 부른다. 의미는 "`PostgresUserRepository` 타입에 `FindByKakaoID` 메서드를 붙인다. 내부에서 `r`로 인스턴스에 접근한다"이다.

### 호출 방법은 Java와 동일

```java
// Java
PostgresUserRepository repo = new PostgresUserRepository(pool);
repo.findByKakaoId("12345");
```

```go
// Go
repo := NewPostgresUserRepository(pool)
repo.FindByKakaoID(ctx, "12345")
```

호출 방법은 완전히 같다. 차이는 **선언 위치**뿐이다.

```
Java:  클래스 블록 안에 넣는다
Go:    함수 앞에 (r *타입) 을 붙인다
```

### 리시버 없이도 가능하지만

```go
// 리시버 없는 일반 함수로 만들면
func FindByKakaoID(r *PostgresUserRepository, ctx context.Context, kakaoID string) (*domain.User, error)

// 호출할 때 직접 넘겨야 함
FindByKakaoID(repo, ctx, "12345")
```

동작은 같지만 두 가지 문제가 있다.

1. `repo.FindByKakaoID()` 형태로 못 쓴다
2. **인터페이스를 구현할 수 없다** — 이게 핵심

Clean Architecture에서 `service → repository 인터페이스`에 의존하는 구조라면 리시버는 필수다.

---

## 다중 반환값 — Go에 예외가 없는 이유

Go는 예외(Exception)가 없다. 대신 **값과 에러를 같이 반환**하는 것이 관례다.

```go
func (r *PostgresUserRepository) FindByKakaoID(...) (*domain.User, error)
//                                                   ↑ 결과값      ↑ 에러
```

Java와 위치가 다르다. Go는 리턴형이 파라미터 뒤에 오고, 여러 개를 반환할 때 `()`로 묶는다.

```java
// Java — 리턴형이 앞에, 에러는 throw
public User findByKakaoId(String kakaoId) throws SQLException
```

```go
// Go — 리턴형이 뒤에, 에러도 반환값
func FindByKakaoID(kakaoID string) (*domain.User, error)
```

### 반환 경우의 수

```go
return user, nil   // 정상: 유저 찾음, 에러 없음
return nil, nil    // 정상: 유저 없음(미가입), 에러도 없음
return nil, err    // 비정상: DB 오류 발생
```

`nil, nil`이 어색해 보이지만, "유저 없음"이 에러가 아니라 정상 분기일 때 이 패턴을 쓴다. service 레이어에서 `user == nil`이면 신규 사용자로 처리하는 식이다.

---

## 값 리시버 vs 포인터 리시버

```go
func (r PostgresUserRepository) FindByKakaoID(...)  // 값 리시버
func (r *PostgresUserRepository) FindByKakaoID(...) // 포인터 리시버
```

### 메모리 수준에서의 차이

**값 리시버**: 호출 시 구조체를 복사해서 `r`에 넘긴다.

```
메모리:  주소 0x01 → [Repo{pool: ...}]
                         ↓ 역참조 후 복사
         주소 0x99 → [Repo{pool: ...}]  ← r (새로 생긴 복사본)
```

**포인터 리시버**: 주소만 넘긴다. 복사 없음.

```
메모리:  주소 0x01 → [Repo{pool: ...}]
                         ↑
         r = 0x01  (주소만 들고 있음)
```

### 호출 시 자동 변환

```go
repo  := Repo{}   // 값
prepo := &Repo{}  // 포인터
```

| 호출 | 값 리시버 `(r Repo)` | 포인터 리시버 `(r *Repo)` |
|------|---------------------|--------------------------|
| `repo.Method()` | 그대로 복사 | Go가 `(&repo).Method()`로 변환 |
| `prepo.Method()` | `(*prepo)` 역참조 후 복사 | 그대로 주소 전달 |

포인터로 값 리시버 메서드를 호출하면 `(*prepo)`로 역참조한 뒤 복사가 일어난다. 반대로 값으로 포인터 리시버를 호출하면 `&repo`로 주소를 만들어 넘긴다.

### 수정 가능 여부

```go
type Counter struct{ count int }

// 값 리시버 — 복사본을 수정, 원본 불변
func (c Counter) Increment() {
    c.count++  // 복사본만 증가
}

// 포인터 리시버 — 원본을 수정
func (c *Counter) Increment() {
    c.count++  // 원본이 증가
}

c := Counter{count: 0}
c.Increment()
fmt.Println(c.count)  // 값 리시버면 0, 포인터 리시버면 1
```

Java는 항상 참조이므로 이런 구분이 없다. Go는 명시적으로 선택해야 한다.

---

## pool/client 같은 연결 객체에 포인터 리시버를 써야 하는 이유

`pgxpool.Pool`은 내부적으로 이런 구조다.

```
Pool
├── 연결 1 (실제 DB 소켓)
├── 연결 2
├── 연결 3
└── 뮤텍스 (동시 접근 제어)
```

값 리시버로 쓰면 메서드 호출마다 이 구조체 전체가 복사된다.

**문제 1: 복사 비용** — Pool이 크면 매 호출마다 무거운 복사가 일어난다.

**문제 2: 동시성 버그** — Pool 안의 뮤텍스가 복사되면 의도한 대로 동작하지 않는다.

```
원본 Pool.mutex: 잠금 상태
복사본 Pool.mutex: 잠금 해제 상태  ← 복사 시점의 상태가 굳어버림
```

두 뮤텍스가 서로 다른 상태로 따로 놀게 된다. 포인터 리시버를 쓰면 모든 메서드 호출이 같은 pool을 공유한다.

---

## 역참조(Dereference)란

포인터는 값의 **주소**를 들고 있다. 역참조는 그 주소를 따라가서 **실제 값을 꺼내는 것**이다.

```go
repo := Repo{name: "postgres"}  // 값
p    := &repo                   // p는 repo의 주소 (예: 0x01)

// 역참조: 주소를 따라가서 값을 꺼냄
fmt.Println(*p)       // Repo{name: "postgres"}
fmt.Println(*p == repo) // true — 같은 값
```

`*p`는 "p가 가리키는 곳의 값"이다. Java에는 이 개념이 없다. Java는 객체를 항상 참조로만 다루기 때문에 주소를 직접 조작할 일이 없다.

### 역참조가 쓰이는 상황

**포인터로 값 리시버 메서드를 호출할 때:**

```go
func (r Repo) Find() {}  // 값 리시버 — Repo를 받아야 함

prepo := &Repo{}         // 포인터
prepo.Find()             // Go가 내부적으로 (*prepo).Find() 로 변환
//                               ↑ 역참조: 주소 → 값
//                                         값을 복사해서 r에 넘김
```

**포인터 리시버 메서드 내부에서 필드에 접근할 때:**

```go
func (r *Repo) SetName(name string) {
    r.name = name  // Go가 내부적으로 (*r).name = name 으로 처리
    //              ↑ 역참조: r(주소)을 따라가서 실제 필드를 수정
}
```

Go에서는 `r.name`이라고 쓰면 알아서 역참조를 해준다. `(*r).name`이라고 명시적으로 쓸 필요가 없다.

---

## 메서드셋과 인터페이스 구현

Go 컴파일러는 타입별로 "이 타입이 가진 메서드 목록"을 관리한다.

```go
type Repo struct{}

func (r Repo) Find() {}   // "Repo에 Find를 붙인다"
func (r *Repo) Save() {}  // "*Repo에 Save를 붙인다"
```

컴파일러가 이 선언을 보고 두 개의 테이블을 만든다.

```
Repo  테이블: { Find }
*Repo 테이블: { Find, Save }
```

`*Repo` 테이블에 `Find`가 자동으로 포함되는 이유:
- `Find`는 값 리시버 → 포인터에서 호출할 때 역참조 후 복사하면 실행 가능 → 자동 포함
- `Save`는 포인터 리시버 → 값에서 호출하려면 주소가 필요한데 값은 주소가 없음 → `Repo` 테이블에 넣을 수 없음

```
*Repo에 붙인 메서드 → Repo 테이블에 줄 수 없음 (값은 주소가 없어서 실행 불가)
 Repo에 붙인 메서드 → *Repo 테이블에 자동으로 줌 (역참조하면 실행 가능)
```

### 인터페이스 구현 체크

```go
type Saver interface{ Save() }
```

`Saver`를 구현하려면 테이블에 `Save()`가 있어야 한다.

```
Repo  테이블: { Find }        → Save() 없음 → Saver 구현 못 함
*Repo 테이블: { Find, Save }  → Save() 있음 → Saver 구현함
```

```go
var s Saver = Repo{}   // ❌ 컴파일 에러: Repo 테이블에 Save() 없음
var s Saver = &Repo{}  // ✅ 가능: *Repo 테이블에 Save() 있음
```

### 설령 컴파일이 된다면 — 복사 문제

값을 인터페이스에 담으면 Go는 그 값을 **인터페이스 내부에 복사해서 저장**한다.

```
Repo{} 원본  →  복사  →  인터페이스 s 내부 (복사본)
```

```go
var s Saver = Repo{}  // 인터페이스 안에 복사본이 들어감

s.Save()
// Save()는 포인터 리시버 — 원본 주소가 필요
// 그런데 s 안에는 복사본만 있고 원본 주소가 없음 → 버그
```

`&Repo{}`를 담으면 포인터(원본 주소)가 복사되므로 이 문제가 없다.

Go는 메서드 테이블 불일치만으로 컴파일 에러를 낸다. 복사 문제는 "왜 그런 규칙이 생겼는지"의 배경이다.

---

## 언제 뭘 쓰나

| 상황 | 선택 |
|------|------|
| 상태 변경 필요 | 포인터 `*` |
| pool/mutex 등 복사 불가 필드 보유 | 포인터 `*` |
| 연결 객체를 들고 있을 때 | 포인터 `*` |
| 불변, 복사 안전, 작은 구조체 | 값 |

실제 표준 라이브러리 사례:

- `time.Time` — 모든 메서드가 값 리시버 (불변, 복사 안전)
- `sql.DB`, `redis.Client` — 모든 메서드가 포인터 리시버 (연결 객체)
- `errors.New` 반환값 — 포인터 리시버 (동등성 보장)

### 한 타입 안에서는 통일

한 타입에 메서드가 여러 개면 **전부 포인터 또는 전부 값으로 통일**하는 것이 Go 관례다. 섞으면 인터페이스 구현 시 `*T`로만 담아야 하는지 `T`로도 되는지 혼란이 생긴다.

`pool`, `client`, `mutex`를 들고 있으면 포인터로 통일한다. 그 외에는 수정 필요 여부와 복사 비용을 기준으로 결정한다.

---

## 리시버는 반드시 구체 타입에 붙인다

Go를 처음 접하면 "인터페이스에도 리시버를 붙일 수 있지 않나?"라는 의문이 생긴다. 결론부터 말하면 **불가능하다**.

```go
type UserRepository interface {
    FindOrCreate(ctx context.Context, kakaoID string) (*User, error)
}

// 이건 불가능
func (r UserRepository) FindOrCreate(...) {
    r.pool.QueryRow(...)  // ← 인터페이스에는 필드가 없다
}
```

인터페이스는 "이 메서드들이 있어야 한다"는 **계약(contract)**만 정의한다. `pool`이나 `client` 같은 필드가 없으니 실제 동작을 구현할 수 없다.

리시버는 항상 구체 타입(구조체)에 붙인다.

```go
// 구체 타입에 붙이는 것만 가능
func (r *PostgresUserRepository) FindOrCreate(...) {
    r.pool.QueryRow(...)  // pool 필드가 있으니 실행 가능
}
```

Java와 비교하면:

```java
// Java — 인터페이스에는 구현 없음
interface UserRepository {
    User findOrCreate(String kakaoId);
}

// 클래스(구체 타입)에 구현
class PostgresUserRepository implements UserRepository {
    private Pool pool;  // 필드가 있어야 구현 가능

    public User findOrCreate(String kakaoId) { ... }
}
```

Go의 리시버 = Java의 클래스 내부 메서드 구현. 역할이 동일하다.

---

## 생성자는 인터페이스를 반환한다

구체 타입에 리시버를 붙이되, **외부에 노출할 때는 인터페이스를 반환**하는 것이 Go 관례다.

```go
// 구체 타입 반환 — 호출하는 쪽이 PostgresUserRepository에 직접 의존
func NewPostgresUserRepository(pool *pgxpool.Pool) *PostgresUserRepository {
    return &PostgresUserRepository{pool: pool}
}

// 인터페이스 반환 — 호출하는 쪽은 UserRepository만 알면 됨
func NewPostgresUserRepository(pool *pgxpool.Pool) domain.UserRepository {
    return &PostgresUserRepository{pool: pool}  // 내부에서는 구체 타입으로 생성
}
```

인터페이스를 반환하면 두 가지 이점이 있다.

**1. 구현체 교체가 자유롭다** — 나중에 MySQL로 바꾸더라도 호출하는 쪽 코드를 건드리지 않아도 된다.

**2. 테스트에서 mock으로 교체할 수 있다** — 실제 DB 없이 인터페이스를 구현한 mock 구조체로 대체 가능하다.

```go
// service는 domain.UserRepository 인터페이스에만 의존
type AuthService struct {
    userRepo domain.UserRepository  // PostgresUserRepository인지 MockUserRepository인지 모름
}

// 테스트에서 mock 주입
svc := NewAuthService(MockUserRepository{})

// 프로덕션에서 실제 구현체 주입
svc := NewAuthService(NewPostgresUserRepository(pool))
```

리시버는 구체 타입에, 반환은 인터페이스로 — 이 두 원칙이 Clean Architecture에서 의존성을 역전시키는 핵심이다.
