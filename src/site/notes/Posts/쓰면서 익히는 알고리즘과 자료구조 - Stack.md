---
{"dg-publish":true,"permalink":"/posts/stack/","created":"2025-10-12","updated":"2025-10-12"}
---

스택을 활용하는 기초 문제 valid parentheses 풀어보기
https://leetcode.com/problems/valid-parentheses/

```ts
function isValid(string: string): boolean {
	const stack = [];
	const starts = ['(', '[', '{'];
	const pairs = {
		'(': ')',
		'[': ']',
		'{': '}'
	};
	
	for(const char of string){
		if(starts.includes(char)){
			stack.push(char);
		}else {
			if(stack.pop() !== pairs[char]){
			return false;
			}
		}
	}
	return stack.length === 0;
};
```

```ts
function isValid(string: string): boolean {
  const stack = [];
  const pairs = {
		'(': ')',
		'[': ']',
		'{': '}'
	};

  for (const char of string) {
    switch (char) {
      case '(':
      case '{':
      case '[':
        stack.push(char);
        break;
      case ')':
      case '}':
      case ']':
        if (stack.pop() !== pairs[char]) return false;
        break;
      default:
        break;
    }
  }

  return stack.length === 0;
}
```
