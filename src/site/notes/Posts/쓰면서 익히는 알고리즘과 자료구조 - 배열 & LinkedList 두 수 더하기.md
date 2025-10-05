---
{"dg-publish":true,"permalink":"/posts/and-linked-list/","created":"2025-10-05","updated":"2025-10-05"}
---

```js
class ListNode {
  constructor(val, next = null) {
    this.val = val;
    this.next = next;
  }
}

function addTwoNumbers(l1, l2) {
  let dummy = new ListNode(0);
  let curr = dummy;
  let carry = 0; //자릿수 합이 10 이상이면 `carry`를 다음 자리로 넘긴다.

  while (l1 || l2 || carry) {
    const val1 = l1 ? l1.val : 0;
    const val2 = l2 ? l2.val : 0;
    const sum = val1 + val2 + carry;

    carry = Math.floor(sum / 10);
    curr.next = new ListNode(sum % 10);
    curr = curr.next;

    if (l1) l1 = l1.next;
    if (l2) l2 = l2.next;
  }

  return dummy.next;
}


// 1 → 2 → 3
const l1 = new ListNode(1, new ListNode(2, new ListNode(3)));
// 4 → 5 → 6
const l2 = new ListNode(4, new ListNode(5, new ListNode(6)));

let result = addTwoNumbers(l1, l2);

// 결과 출력
let out = [];
while (result) {
  out.push(result.val);
  result = result.next;
}
console.log(out.join(" → ")); // 5 → 7 → 9

```

LinkedList 구현인 ListNode가 아닌, 배열 리터럴로 주어졌을 때 구현해보기.

```js
function addLinkedLists(l1, l2) {
  const maxLen = Math.max(l1.length, l2.length);
  const result = new Array(maxLen).fill(0);
  let carry = 0;

  for (let i = maxLen - 1; i >= 0; i--) {
    const a = l1[i - (maxLen - l1.length)] || 0;
    const b = l2[i - (maxLen - l2.length)] || 0;
    const sum = a + b + carry;

    result[i] = sum % 10;
    carry = Math.floor(sum / 10);
  }

  if (carry > 0) result.unshift(carry);

  return result;
}

// 예시
console.log(addLinkedLists([1, 2, 3], [4, 5, 6])); // [5, 7, 9]
```

만약 배열 길이가 다른 케이스가 있다면, 단순 자리별 합으로 처리하도록 해보자.

```js
function addLinkedListsElementwise(l1, l2) {
  const maxLen = Math.max(l1.length, l2.length);
  const result = [];

  for (let i = 0; i < maxLen; i++) {
    const a = l1[i] ?? 0;
    const b = l2[i] ?? 0;
    result.push(a + b);
  }

  return result;
}

// ✅ 예시
console.log(addLinkedListsElementwise([1, 2, 3], [4, 5])); // [5, 7, 3]
console.log(addLinkedListsElementwise([1, 2], [9, 9, 9])); // [10, 11, 9]

```