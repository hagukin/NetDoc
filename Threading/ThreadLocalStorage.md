### Thread Local Storage
TLS는 스레드마다 존재하는 전용 메모리 공간으로, 스택과 유사하다. 스레드별로 개별적으로 가지고 있으나, 스택과는 다음과 같은 차이점이 존재한다.  
1) 스택과 달리 TLS 내부의 데이터는 (해당 스레드 내부에서) 전역변수일 수 있다. 전역이지만 해당 스레드만이 접근할 수 있다. 또한 이 전역변수에 대한 수정사항은 서로다른 스레드(main 스레드 포하)에서는 적용되지 않는다. 즉 스레드 내에서만 전역변수의 역할을 하는 것이다.   
2) 스택은 함수 콜에 사용되다보니 자주 메모리 할당/해제가 이루어지므로 불안정하다. 때문에 중요한 정보를 저장하기 적합하지 않다.  

[참고글](https://stormpy.tistory.com/m/168)  
참고영상: 17 - Thread Local Storage  

```c++
#include <iostream>
#include <mutex>
using namespace std;

thread_local unsigned int i = 0;
std::mutex g_mutex;

void OnThread(int nid)
{
	++i;
	std::unique_lock<std::mutex> lock(g_mutex);
	cout << nid << "-thread" << i << endl;
}

int main()
{
	thread th1(OnThread, 0);
	thread th2(OnThread, 1);

	std::unique_lock<std::mutex> lock(g_mutex);
	cout << "main thread : " << i << endl;
	lock.unlock();

	th1.join();
	th2.join();
	
	return 0;
}

/*
output:
0-thread1
1-thread1
main thread : 0

전역 변수로 선언된 i는 각각의 thread에서 증가를 시킴에도 같은 값을 가지며, main에서도 변하지 않음 값을 표시한다. thread_local을 사용하였기 때문에 변수는 각각의 thread에서만 동작한 것이다.
*/
```
