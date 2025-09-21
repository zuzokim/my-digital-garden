---
{"dg-publish":true,"permalink":"/posts/linked-list-and-array/","created":"2025-09-21","updated":"2025-09-21"}
---


[쓰면서 익히는 알고리즘과 자료구조](https://product.kyobobook.co.kr/detail/S000001810374) 를 읽고 정리해보는 기초 자료구조 이해.. 챕터3의 연결리스트(Linked List) 부분정리!


- **연결 리스트(Linked List)**
    - 노드(Node) 단위로 데이터를 저장한다.
    - 각 노드는 **데이터 + 다음 노드의 포인터**로 구성된다.
    - 메모리 공간이 흩어져 있어도 상관없다.
- **배열(Array)**
    - 연속된 메모리 공간에 데이터를 저장한다.
    - 각 요소는 **index**로 접근할 수 있다.
    - 메모리 상에서 연속적이라 접근 속도가 빠르다.

- 시간복잡도 비교 : **접근은 배열이 강점**, **삽입·삭제는 연결 리스트가 강점**
	- **접근 (Access)**
	    - 연결 리스트: O(n) — 원하는 위치까지 순차적으로 탐색해야 함
	    - 배열: O(1) — 인덱스로 바로 접근 가능
	- **삽입 (Insert)**
	    - 연결 리스트: O(1) — 포인터만 바꿔주면 되지만, 삽입 위치를 찾는 데 O(n)이 걸릴 수 있음
	    - 배열: 평균 O(n) — 중간에 넣으려면 뒤 요소들을 한 칸씩 밀어야 함
	- **삭제 (Delete)**
	    - 연결 리스트: O(1) — 포인터만 끊어주면 되지만, 삭제할 노드를 찾는 데 O(n)이 걸릴 수 있음
	    - 배열: 평균 O(n) — 삭제 후 뒤 요소들을 당겨야 함
	- **검색 (Search)**
	    - 연결 리스트: O(n)
	    -  배열: O(n)


자바스크립트로 연결리스트 구현하기

```js
class Node {
  constructor(value) {
    this.value = value;
    this.next = null;
  }
}


class LinkedList {
  constructor() {
    this.head = null;
    this.size = 0;
  }

  // 맨 뒤에 노드 추가
  append(value) {
    const newNode = new Node(value);

    if (!this.head) {
      this.head = newNode;
    } else {
      let current = this.head;
      while (current.next) {
        current = current.next;
      }
      current.next = newNode;
    }

    this.size++;
  }

  // 맨 앞에 노드 추가
  prepend(value) {
    const newNode = new Node(value);
    newNode.next = this.head;
    this.head = newNode;
    this.size++;
  }

  // 특정 값 삭제
  remove(value) {
    if (!this.head) return;

    if (this.head.value === value) {
      this.head = this.head.next;
      this.size--;
      return;
    }

    let current = this.head;
    while (current.next && current.next.value !== value) {
      current = current.next;
    }

    if (current.next) {
      current.next = current.next.next;
      this.size--;
    }
  }

  // 탐색
  find(value) {
    let current = this.head;
    while (current) {
      if (current.value === value) return current;
      current = current.next;
    }
    return null;
  }

  // 리스트 출력
  print() {
    let current = this.head;
    const values = [];
    while (current) {
      values.push(current.value);
      current = current.next;
    }
    console.log(values.join(" -> "));
  }
}

const list = new LinkedList();

list.append(10);
list.append(20);
list.append(30);
list.print(); // 10 -> 20 -> 30

list.prepend(5);
list.print(); // 5 -> 10 -> 20 -> 30

list.remove(20);
list.print(); // 5 -> 10 -> 30

console.log(list.find(30)); // Node { value: 30, next: null }

```