IOCP Reference Counting, 스마트 포인터 참고  
우선 직접 구현한 방식부터 살펴보자.  
멀티스레드에서 레퍼런스 카운팅을 해줄 수 있게 하려면 일단 모든 객체가 RefCountable을 상속받아 제작되게 만든다  
RefCountable은 atomic 연산을 통해 멀티스레드 환경에서도 올바른 Reference Counting이 처리되도록 한다
이후 해당 객체들을 참조할 일이 생기면 RefCountable과 연계해 구현한 TSharedPtr로 참조함으로써 레퍼런스 카운트가 0이 되면 그제서야 메모리를 해제하도록 만든다.  
이렇게 하면 멀티스레드에서의 레퍼런스 카운트 문제(다른 스레드에서 디레퍼런싱하더라도 스레드가 꼬여 RefCount가 이상한 값으로 변하는 등)를 막을 수 있다.  
  
직접 구현하는 것의 단점도 물론 존재한다. 우선 모든 객체를 RefCountable을 상속받게 하는 것 자체가 구조적으로 쉽지 않고, 외부 라이브러리 등을 사용할 경우에는 아예 불가능할 수도 있다.  
또 하나는 순환 문제인데, 이건 std::shared_ptr에서도 발생하는 문제이다.  
데드락과 유사하게 두 객체가 서로 레퍼런스를 잡을때 (두 개 이상이어도 사이클이면 마찬가지) 둘 다 서로 레퍼런스를 잡고 있기 때문에 둘다 해제가 안되는 현상이 일어날 수 있다.  
Actor와 Inventory가 서로에 대한 레퍼런스를 잡고 있는 경우처럼, 흔하게 발생할 수 있는 상황이기 때문에 항상 주의가 필요하다. (Strongly connected)  

해결법은 여러가지가 있는데, 대표적인게 weak_ptr의 사용이다.  
weak_ptr를 알아보기 전에 std::shared_ptr의 구조를 알아야 하는데,  
shared_ptr 내부에는 
1) 레퍼런스하려는 객체의 포인터   
2) RefCount라는 객체의 포인터  
이렇게 두 가지가 들어 있다.  
여기서 RefCount에는 shared ptr들이 참조한 ref count와 weak ptr들이 참조한 ref count 두 개가 다 들어있다.  
shared_ptr는 기본적으로 동일한 방식으로 ref count를 진행하고 ref count가 0이 되면 가리키는 객체를 해제해버리는 건 동일하지만,  
만약 shared ptr ref count는 0이지만 weak ptr ref count이 0이 아닐 경우,  
가리키는 객체는 해제하고 RefCount는 메모리에 살려둔다.  
즉 weak ptr은 설사 원본 shared ptr이 해제되었더라도 최소한 RefCount에는 접근할 수 있다.  
```c++
shared_ptr<Actor> spr = make_shared<Actor>();
weak_ptr<Actor> wpr = spr;
// 이후 spr 해제
bool expired = wpr.expired(); // 원본 spr이 해제되었는지 확인 가능
// 만약 해제되었다면 wpr은 nullptr이 된다.  
```
