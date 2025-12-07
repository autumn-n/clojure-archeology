# Clojure 아키텍처 개요

이 문서는 커밋을 탐색하면서 발견한 Clojure의 전체 설계 구조를 정리합니다.

> 마지막 업데이트: 커밋 #12 (2006-03-28)

---

## 클래스 계층 구조

```
IFn (interface) ─────────────────────────┐
  │                                       │
  ▼                                       │
AFn (abstract class)                      │
  │  - implements IFn                     │
  │  - extends AMap                       │
  │  - 기본 invoke() → throwArity()       │
  │  - applyTo() → invoke() 디스패치      │
  │                                       │
  │                                       │
  ├──▶ Symbol                             │
  │      - 심볼도 함수! (커밋 #6)          │
  │      - invoke(obj) → obj.get(this)    │
  │      - invoke(obj,val) → obj.put()    │
  │                                       │
  ├──▶ RestFn0~5 (커밋 #8)                │
  │      - 가변 인자 함수 (& rest)         │
  │      - doInvoke(tld, args..., rest)   │
  │                                       │
  ▼                                       │
[사용자 정의 함수들]                       │
                                          │
AMap ─────────────────────────────────────┘
  │  - Symbol → Object 매핑
  │  - IdentityHashMap 기반
  │  - 함수의 메타데이터 저장용

Cons
  │  - first: Object
  │  - rest: Cons
  │  - Lisp의 기본 리스트 구조

ThreadLocalData
  │  - 스레드별 동적 바인딩 관리
  │  - InheritableThreadLocal 사용

RT (Runtime)
  │  - 정적 유틸리티 메서드들
  │  - cons, list, listStar, length 등

Namespace (커밋 #7)
  │  - 심볼들의 컨테이너
  │  - static table: String → Namespace (전역)
  │  - symbols: String → Symbol (로컬)
  │  - intern(): 심볼 인터닝

Num (커밋 #12) ─────────────────────────────
  │  - 숫자 시스템의 루트 (extends Number)
  │  - 자동 타입 승급, 정확한 산술
  │
  ├── IntegerNum
  │     ├── FixNum (int)
  │     └── BigNum (BigInteger)
  ├── RealNum
  │     ├── FloatNum
  │     └── DoubleNum
  ├── Rational
  └── RatioNum (분수: n/d)
```

---

## 핵심 개념

### 1. Cons (커밋 #2)
Lisp의 가장 기본적인 데이터 구조. 두 개의 포인터로 구성:
- `first` (car): 현재 요소
- `rest` (cdr): 나머지 리스트

```
(1 2 3) = Cons(1, Cons(2, Cons(3, null)))

┌───┬───┐   ┌───┬───┐   ┌───┬───┐
│ 1 │ ●─┼──▶│ 2 │ ●─┼──▶│ 3 │nil│
└───┴───┘   └───┴───┘   └───┴───┘
```

### 2. IFn - 함수 인터페이스 (커밋 #3)
Clojure에서 **모든 함수**가 구현하는 인터페이스.

```java
public interface IFn {
    Object invoke(ThreadLocalData tld) throws Exception;
    Object invoke(ThreadLocalData tld, Object arg1) throws Exception;
    Object invoke(ThreadLocalData tld, Object arg1, Object arg2) throws Exception;
    // ... 최대 5개 + 가변인자
    Object applyTo(ThreadLocalData tld, Cons arglist) throws Exception;
}
```

**왜 오버로드?**
- 성능: 박싱/언박싱 오버헤드 최소화
- 대부분의 함수 호출은 인자가 5개 이하

### 3. 동적 바인딩 (Dynamic Binding) (커밋 #2)
Common Lisp의 special variable과 유사. 스레드별로 다른 값을 가질 수 있음.

```java
// ThreadLocalData에서 관리
IdentityHashMap dynamicBindings;  // Symbol → Cons (스택)

pushDynamicBinding(sym, val)  // 바인딩 추가
popDynamicBinding(sym)        // 바인딩 제거
getDynamicBinding(sym)        // 현재 값 조회
```

Clojure의 `^:dynamic` 변수, `binding` 매크로의 기초.

### 4. 함수도 맵이다 (커밋 #4)
`AFn extends AMap` - 함수도 속성(메타데이터)을 가질 수 있음.

```clojure
;; Clojure에서 이렇게 사용
(def ^{:doc "덧셈 함수"} my-add +)
(meta my-add)  ; => {:doc "덧셈 함수"}
```

### 5. 심볼/키워드도 함수다 (커밋 #6)
`Symbol extends AFn` - 심볼을 함수처럼 호출하면 맵에서 값을 꺼냄.

```clojure
;; Clojure에서 이렇게 사용
(:name person)     ; 심볼/키워드를 함수로 호출
;; 내부적으로: symbol.invoke(person) → person.get(symbol)

;; 아래 두 표현은 동일
(:name person)
(get person :name)
```

**"모든 것이 함수"** 철학의 구현. 데이터 접근도 함수 호출로 통일.

### 6. Namespace와 Symbol 인터닝 (커밋 #7)
Namespace는 심볼들을 그룹화하는 컨테이너.

```
┌─────────────────────────────────────────────┐
│            Namespace.table (전역)            │
│  "user" → Namespace ─┬─ symbols ─┬─ "x" → Symbol(x)
│  "core" → Namespace  │           └─ "y" → Symbol(y)
│                      │
└─────────────────────────────────────────────┘
```

**인터닝 (Interning)**: 같은 이름 = 같은 객체
```java
// Namespace.intern()
Symbol sym = symbols.get(name);
if(sym == null)
    symbols.put(name, sym = new Symbol(name, this));
return sym;  // 항상 같은 객체 반환
```

**Symbol의 값 검색 순서:**
```
1. ThreadLocalData.getDynamicBinding(sym)  → 동적 바인딩
2. symbol.val                               → 전역 값
3. UNBOUND → Exception
```

이것이 나중에 **Var** 시스템으로 발전.

---

## 설계 원칙 (지금까지 관찰)

1. **JVM 위의 Lisp**: Java 클래스로 Lisp 기본 구조 구현
2. **성능 고려**: 인자 개수별 오버로드로 박싱 최소화
3. **스레드 안전성**: 처음부터 멀티스레드 고려 (ThreadLocalData)
4. **메타데이터**: 모든 것이 메타데이터를 가질 수 있는 구조
5. **모든 것이 함수**: 심볼, 키워드 등도 IFn 구현 → 데이터 접근 문법 통일

---

## 아직 등장하지 않은 것들

- [ ] Reader (코드 파싱)
- [ ] Compiler
- [ ] Var (전역 변수) - Symbol.val이 초기 형태
- [x] Namespace (커밋 #7)
- [ ] 컬렉션들 (Vector, Map, Set)
- [ ] Seq 추상화
- [ ] 매크로 시스템
- [ ] Java interop

---

## 용어집

| 용어 | 설명 |
|------|------|
| **Cons** | Lisp의 기본 페어 구조 (first + rest) |
| **Symbol** | 이름을 나타내는 식별자 |
| **IFn** | Invokable Function - 호출 가능한 것의 인터페이스 |
| **AFn** | Abstract Function - IFn의 기본 구현 |
| **Arity** | 함수의 인자 개수 |
| **Dynamic Binding** | 스레드/스코프별로 다른 값을 가지는 바인딩 |
| **RT** | Runtime - 런타임 유틸리티 클래스 |
| **Namespace** | 심볼들을 그룹화하는 컨테이너 |
| **Interning** | 같은 이름의 심볼이 항상 같은 객체가 되도록 하는 기법 |
| **UNBOUND** | 심볼에 아직 값이 바인딩되지 않은 상태 |
| **RestFn** | 가변 인자(& rest)를 지원하는 함수의 베이스 클래스 |
| **Num** | 숫자 타입의 추상 베이스 클래스 |
| **RatioNum** | 정확한 유리수 (분자/분모) |
| **Double Dispatch** | 두 인자의 타입에 따라 메서드를 선택하는 패턴 |
