# ABA Problem
## 선수지식
Compare and Exchange(CAS)

## 개요
[참고글](!https://blog.naver.com/jjoommnn/130040068875)
위 내용을 아주 간단하게 설명하면, 타 스레드에서 pop 두번에 pop한 메모리를 push해서 stack이 최종적으로  
A B C에서  
A C가 되었다면,  
CAS가 true를 반환해서 top이던 A를 B로 변경하게 된다. 그러나 B는 이미 타 스레드에서 pop했기 때문에 해제되었을 여지가 있고, 쓰레기값이 스택의 top에 들어가게 되는 참사가 발생한다.  

## 언제 주로 발생할 수 있는가
메모리의 재사용이 빈번한 memory pool 등의 구현에서 만약 lock free 자료구조(스택)를 사용한다면 발생할 여지가 있다. (물론 그냥 lock free를 쓰는 것 자체만으로도 발생할 여지는 있지만, 메모리의 재사용이 아주 빈번한 메모리풀에서 더 그럴 가능성이 높다)  

IOCP 33 - Memory pool #2 참고

## 어떻게 해결하는가
