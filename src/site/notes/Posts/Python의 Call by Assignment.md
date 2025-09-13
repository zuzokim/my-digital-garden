---
{"dg-publish":true,"permalink":"/posts/python-call-by-assignment/","tags":["Python"],"created":"2025-08-17","updated":"2025-08-17T17:00:00"}
---

Python을 자료구조, 알고리즘 기초 지식과 함께 배워보고 있다. Javascript, Typescript 외에 처음 배워보는 언어라서 작성 방식도 낯설고, 개념도 조금씩 다르다. 익숙해져보기를 목표로 배워보고 있다.

## Call by Value (값에 의한 호출)

- **함수에 인자를 넘길 때, 값 자체를 복사해서 전달**
- 함수 안에서 인자를 바꿔도 **원본 변수에는 영향 없음**

예) C / Java

```c
void foo(int x) {
    x = 10;  // 복사본만 바뀜
}

int main() {
    int a = 5;
    foo(a);
    printf("%d", a); // 5
}
```
여기서 함수 안 `x`는 `a`의 **복사본** → 원본은 그대로다.

## Call by Reference (참조에 의한 호출)

- **함수에 인자를 넘길 때, 원본 객체를 참조해서 전달**
- 함수 안에서 값 변경하면 **원본에도 영향 있음**

예) C++

```c++
void foo(int &x) {  // & 사용: 참조
    x = 10;
}

int main() {
    int a = 5;
    foo(a);
    cout << a; // 10
}

```

## Python의 Call by Assignment (혹은 Call by Object Reference)

- Python에서는 **값 자체를 복사하지 않고, 변수에 바인딩된 객체 참조를 전달**
- 그래서 “Call by Value”도 아니고, “Call by Reference”도 아님
- 흔히 **Call by Object Reference / Call by Assignment**라고 부름
### 동작 방식
1. **변수는 객체에 대한 이름표**
2. **함수 호출 시, 이름표를 복사** → 함수 안 변수는 원본 객체를 가리킴
3. **mutable 객체를 수정하면 원본 변경**, immutable 객체는 변경 불가

예) mutable 객체 (리스트)
```python
def foo(lst):
    lst.append(4)  # 원본 리스트 수정

my_list = [1,2,3]
foo(my_list)
print(my_list)  # [1,2,3,4] ✅
```

예) immutable 객체 (int)
```python
def bar(x):
    x = x + 1  # 새로운 int 객체 할당

a = 5
bar(a)
print(a)  # 5 ✅ (원본 변경 없음)
```
- `x = x + 1` → **새로운 int 객체**를 `x`가 가리키도록 변경
- 원래 `a`는 여전히 5를 가리킴

## JS 예제 (primitive or reference type 케이스와 Python 비교)

예) primitive (immutable)
```js
let a = 5;
function foo(x) { x = x + 1; }
foo(a);
console.log(a); // 5
```
- 동작 원리는 Python int와 동일
- 값 자체가 복사됨 → 원본 영향 없음

예) Object (mutable)
```js
let obj = { val: 1 };
function bar(o) { o.val = 2; }
bar(obj);
console.log(obj.val); // 2
```
- 동작 원리는 Python list와 동일
- 객체 참조가 전달 → 원본 변경 가능

- JS 는 Call by Value로 모든 인자를 값으로 넘기지만 값이 primitive타입인지 reference타입인지에 따라 다르게 동작한다.
- Python은 **모든 값이 객체**이고, Call by Assignment에 따라 mutable객체는 원본을 변경하고, immutable 객체(int, float, str 등)는 변경 불가한 방식으로 동작한다.


| 언어     |     | Call 방식            | 값 타입      | 동작                          |
| ------ | --- | ------------------ | --------- | --------------------------- |
| JS     |     | Call by Value      | primitive | 값 복사 → 함수 안 변경 불가           |
| JS     |     | Call by Value      | object    | 참조값 복사 → 함수 안에서 객체 내부 변경 가능 |
| Python |     | Call by Assignment | immutable | 함수 안에서 새 객체 할당 → 원본 변경 불가   |
| Python |     | Call by Assignment | mutable   | 함수 안에서 내부 값 변경 → 원본 변경 가능   |
