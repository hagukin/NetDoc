IOCP 29 참조  

# Stomp Allocator
메모리 오염으로 인한 버그를 방지하는데 유용한 기법이다.  
언리얼에서도 제공하는 기능인만큼 보편적으로 사용된다.  

```c++
// 메모리 오염의 아주 간단한 예

// 1. Use after free problem
Player* p1 = new Player();
p1->name = "John";
delete p1;
p1->name = "Tom"; // 메모리 오염! 이미 해제한 메모리를 건드리고 있음에도 크래시가 나지 않는다.
// 만약 해당 위치에 어떤 유효한 데이터가 위치하고 있었다면 이는 정보 손실 등의 끔찍한 결과를 초래할 수 있다.
// 사실 이건 스마트포인터를 활용해서 해결할 수도 있다.

// 2. 벡터 clear 이후 접근
vector<int> v{1,2,3,4,5};
for (int i=0; i<5; ++i)
{
  int val = v[i];
  if (val == 3)
  {
    v.clear();
  }
  doSomething(v[i]); // v가 해제되었을 수도 있음에도 사용중!
}
// 사실 이 예시는 컴파일러에서 잡히긴 하지만 안잡히는 경우도 있을 수 있으므로 위험하다

// 3. 상속 관계에서 발생하는 문제
/*
Entity를 상속한 Actor라는 클래스가 있을 때
Entity* entity;
entity를 actor = static_cast<Actor>(entity)로 형변환 강제로 시킨 이후
actor->actorHp; 식으로 접근할 떄 문제가 발생할 수 있다.
애초에 액터로 형변환하면 안되는 엔티티였을 수도 있기 때문이다.
*/
```


