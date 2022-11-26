# Compare And Exchange (CAS)
## 개요
의사코드로 쉽게 이해가 가능하다. 간혹 함수에 따라 value, expected, desired가 아닌 value, desired, expected 순으로 아규먼트를 받는 경우가 있으니 주의하자.  
```c++
// https://en.wikipedia.org/wiki/Compare-and-swap
// val, expected, desired 순
function cas(p: pointer to int, old: int, new: int) is
    if *p ≠ old
        return false
    *p ← new
    return true
```

lock free 자료구조들의 구현에도 사용된다.  
ABA problem이라는 문제를 안고 있으므로 관련 내용을 참고하자.  
