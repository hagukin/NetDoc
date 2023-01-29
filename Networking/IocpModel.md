# IOCP 입출력 모델
선수지식: Overlapped 모델  
Overlapped 모델을 요약하면 다음과 같다.  
비동기 입출력 함수가 실행되는 동안 스레드는 Alertable wait 상태가 된다.  
함수 실행이 완료되면 쓰레드 별로 있는 APC 큐에 해당 함수가 완료되면 실행될 콜백 함수의 정보와 함께 Alert를 해주라는 요청이 쌓인다.  
이후 APC큐를 비우면서 콜백을 실행하고 그 후 스레드를 Alert해주어 Alertable wait 상태에서 빠져나가게 한다.  

IOCP 모델은 Overlapped 모델과 거의 유사하지만 몇 가지 큰 차이점이 있다.  
1) 스레드마다 APC 큐가 있던 Overlapped 구조와 다르게, 중앙에서 제어하는 Completion Port 큐 하나를 사용한다. (즉 더 멀티스레드 친화적이다)  
2) 콜백이 올 때 까지 Alertable Wait 상태를 유지하던 기존 방식과 다르게 Completion Port 큐의 처리 결과를 GetQueuedCompletionStatus() 함수를 통해 처리한다. (즉 Alertable wait로 인한 부하가 줄어든다)   

전체적인 흐름을 이해하는 데 초점을 두자.  
보다 더 세부적인 IOCP 모델의 구현 방식은 다른 글에서 더 자세히 다뤄보겠다.  
실제 구현에는 이런식으로 절차지향적 구현 대신 객체화(라이브러리화) 시키는데, iocp - iocpCore 

```c++
// 주요 함수
// 1. CreateIoCompletionPort
// 2. GetQueuedCompletionStatus
// CP에서 완료된 일감을 발견할 때까지 대기하는 역할. CP 감시자.  


CreateIoCompletionPort(...)
/*
Completion Port를 생성해줄 때도 사용되고,
어떤 특정 소켓을 Completion Port에 등록해 줄때도 사용한다.
*/

// 사용예시 (IOCP - Completion Port 모델 참고)
// CP 생성
// 여기서는 iocpHandle이라는 이름으로 Completion Port (혹은 CP에 접근 가능한 객체)를 생성중이다. 편의상 일단 iocpHandle = CP라고 이해하고 넘어가자.  
// HANDLE은 메모리주소를 API단에서 노출시키지 않기 위한 포인터 wrapper느낌이다. 참고: https://stackoverflow.com/questions/902967/what-is-a-windows-handle  
HANDLE iocpHandle = ::CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);

// WorkerThread들 생성
// 얘들이 Recv 된 데이터들을 처리해준다
// 임의로 일단 5개의 스레드를 가동한다
for (int32 i=0;i<5;++i)
  GThreadManager->Launch([=]() {WorkerThreadMain(iocpHandle); }); // 스레드를 관리해주는 커스텀 스레드매니저 (구현내용은 생략한다 iocp 참고, 그냥 여기서는 스레드를 만들어주는구나 정도로만 넘어가자)

// ...
// 소켓을 CP에 등록
// accept() 이후 clientSocket라는 소켓 정보를 가지고 있는 상황
CreateIoCompletionPort((HANDLE)clientSocket, iocpHandle, (ULONG_PTR)session/*session의 메모리 주소값*/, 0);
// 해당 소켓에 대해 최초 1회 Recv를 해주어야 한다
// ...
::WSARecv(clientSocket, &wsaBuf, 1, &recvLen, &flags, &overlappedEx->overlapped, NULL);

// ...
GThreadManager->Join();
// ...이후 코드 생략


// Recv 함수 처리 이후의 추가적인 처리(즉 Recv 완료 시 실행될 부분들)는 다른 스레드를 생성해 거기서 처리한다. 
// CP에 소켓을 등록했기 때문에 다른 스레드들도 해당 Recv 함수가 완료 되었는지를 통지받을 수 있기 때문에 가능한 방식이다.
// 별도의 스레드에서 돌아가면서 Recv한 데이터를 처리해주는 WorkerThreadMain()함수를 살펴보자.  
void WorkerThreadMain(HANDLE iocpHandle)
{
  while (true)
  {
    DWORD bytesTransferred = 0;
    Session* session = nullptr;
    OverlappedEx* overlappedEx = nullptr;
    
    // CP에서 일감이 완료되면 이 함수가 처리되어 진행된다.
    BOOL ret = ::GetQueuedCompletionStatus(iocpHandle, &bytesTransferred, (ULONG_PTR)&session, (LPOVERLAPPED*)&overlappedEx, INFINITE); 
    // 앞에서 CreateIoCompletionPort에서 session의 주소값을 ULONG_PTR 으로 넘겼었는데, 이 주소값을 이용해 session을 이 스레드에서 복원시켜준다.  
    // ULONG_PTR은 포인터 연산을 하기 위해 사용되는 자료형으로, 메모리 주소를 정확하게 specify하기 위해 포인터 객체를 ULONG_PTR로 typecast하는 것 같다
    // 참고: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/21eec394-630d-49ed-8b4a-ab74a1614611
    
    if (ret == FALSE || bytesTransferred == 0)
    {
      // 예기치 못한 상황
      continue;
    }
    
    cout << "recv(): bytes - " << bytesTransferred << endl;
    /*
    여기서 사실 일감 하나에 대한 처리는 끝나는데,
    그 다음번 Recv를 호출해주어야 두번째, 세번째 패킷에 대한 지속적인 recv가 가능하다.
    */
    // ... (메인 스레드에서 처음 Recv를 호출한 것과 동일)
    ::WSARecv(session->socket, &wsaBuf, &flags, &overlappedEx->overlapped, NULL); // 재호출
  }
}
```  

참고:  
위에서 다룬 IOCP 모델은 기본적인 흐름을 이해하기 위한 용도로, 해당 구조에는 여러 결함들이 있다.  
대표적인 게 session을 현재 주소값으로 던져주고 있다는 것인데, 만약 session이 유저 disconnection 등으로 해제되게 되면 크래시가 발생할 수 있다는 문제이다.  
(더 심각한 경우 크래시도 안나고 쓰레기값이 전달되어 코드가 엉망이 될 수도 있다)  
이 문제를 해결하려면, 어떤 세션에 대해 WSARecv()같은 입출력 함수를 호출하는 순간, reference count 등을 사용해 입출력 함수가 끝나기 전까지 절대 그 session을 메모리에서 해제되지 않도록 만들어주는 것 등이 있겠다.  

이전에 다루었던 xnew와 같은 커스텀 메모리 할당자 등이 이런 상황들에서 쓰일 수 있다.  




# 라이브러리화 (객체지향적으로 구조화)
내용의 특성상 내용들이 난잡할 수 있기에 흐름을 쭉 따라가기보다는 필요한 부분을 찾아서 사용하는 걸 권장한다  
구현은 1. Listen(accept 처리), 2. Session(recv, send 처리) 순으로 진행된다  

---  

## 1. 기본 뼈대 및 Listener(accpet()) 구현
### IocpCore
IOCP의 핵심 로직들 (completion port 만들고, 여기에 소켓을 등록하고, 소켓들에서 recv하고 등등)을 관리해주는 IocpCore라는 객체를 구현한다.

### IocpObject
IocpObject는 Completion Port에 저장할 데이터 객체이다.  
IocpObject = 소켓이라고 생각하면 이해가 빠르며, 클라이언트 소켓, accept해준느 문지기 소켓, 서버 소켓등 소켓들은 거의 다 IocpObject를 상속받아 만든다고 생각하면 된다.  

IocpObject::Dispatch(IocpEvent* iocpEvent, int32 numOfBytes) 함수는 worker thread들에게 일감을 분배하는 역할이다. 즉 실질적으로 스레드에게 일감을 전달하는 과정이 여기서 처리된다.
```c++
// virtual 함수이기 때문에 IocpObject 상속받은 클래스 내부에서 각자 구현
// Listener(문지기 소켓)의 예를 살펴보자
// Listener은 recv나 send를 애초에 처리하지 않으므로 곧장 acceptEvent로 typecast해 ProcessAccept()를 
void Listener::Dispatch(IocpEvent* iocpEvent, int32 numOfBytes)
{
	ASSERT_CRASH(iocpEvent->GetType() == EventType::Accept);
	AcceptEvent* acceptEvent = static_cast<AcceptEvent*>(iocpEvent);
	ProcessAccept(acceptEvent);
}
```
이 함수는 IocpCore::Dispatch(uint32 timeoutMs)로부터 호출된다.

Dispatch가 인자로 받는 IocpEvent는 예전에 Session에서 Enum으로 eventType(Connect, Accept, Recv, Send) 을 저장한 것과 동일하지만 이걸 객체로 한 번 더 감싸줬다고 생각하면 된다.  
IocpEvent가 OVERLAPPED를 상속받았다는 것이 중요한데, OVERLAPPED를 상속받았기에 메모리 앞부분은 OVERLAPPED가 되고 자연스럽게 typecast없이 상위 클래스인 OVERLAPPED로 typecast가 가능해진다.  
(주의: 상위 클래스로 형변환이 가능해지는 대신 virtual 함수의 사용이 불가능해진다. vtable이 메모리 맨 앞을 차지해버리기 때문에 typecast하면 꼬여버린다.)

### IocpCore::Register()
IocpCore에는 여러 기능들이 있지만 그중 Register()에 대해 잠깐 살펴보자.  

Register()의 목적은 소켓을 CP에 등록하는 것인데, 우리는 이미 위에서 CreateIoCompletionPort() 함수를 이용해 소켓을 CP에 등록하는 법을 배운 적이 있다. 다만 이것보다 조금 더 복잡해지는데, 예전에 우리가 CreateIoCompletionPort에 임시로 제작했던 Session 객체를 넘겨준 것과 다르게 이제 제대로 IocpObject 객체를 넘겨줄 것이다.  

최종적으로 Register는 다음과 같은 형태가 된다.
```c++
bool IocpCore::Register(IocpObject* iocpObject)
{
  return ::CreateIoCompletionPort(
    iocpObject->GetHandle(), // 소켓의 핸들을 반환
    _iocpHandle,
    reinterpret_cast<ULONG_PTR>(iocpObject),
    0
  );
}
```

### IocpCore::Dispatch()
위쪽의 라이브러리화 이전의 약식 코드를 먼저 보고오면 이해가 더 빠르다.  
IocpCore::Dispatch()는 전부 메인 스레드가 아닌 개별 worker 스레드에서 처리된다는 걸 명심하자.  
(즉 Dispatch로 인해 발생하는 모든 IO 처리들 (recv,accept 등)도 마찬가지로 해당 worker 스레드에서 동작한다)  

```c++
bool IocpCore::Dispatch(uint32 timeoutMs)
{
  DWORD numOfBytes = 0;
  IocpObject* iocpObject = nullptr;
  IocpEvent* iocpEvent = nullptr;
  
  
  if (::GetQueuedCompletionStatus(
    _iocpHandle, 
    OUT &numOfBytes, 
    OUT reinterpret_case<PLONG_PTR>(&iocpObject), 
    OUT reinterpret_case<LPOVERLAPPED*>(&iocpEvent), // Register 항목에서 언급했듯이 iocpEvent는 상위 클래스인 OVERLAPPED로 typecast가 가능하다
    timeoutMs) == true)
  {
    // 여기 진입했다는 건 CP가 뭔가 소켓io작업이 끝났음을 감지했다는 의미
    // 고로 worker thread에게 이후 작업을 맡겨야 한다
    iocpObject->Dispatch(iocpEvent, numOfBytes); // 실질적으로 타입에 맡게 스레드에게 작업을 맡기는 과정이 여기서 처리된다
    // 즉 iocpCore의 Dispatch()가 iocpObject의 Dispatch()를 호출
  }
  else
  {
    // 편의상 에러 처리 및 timeout 처리 생략
  }
  
  return false;
}
```

---  

## Session
특정 클라이언트 하나와 관련된 모든 정보들을 담고 있는 객체이다.  
당연히 CP에 등록해야 하므로 IocpObject를 상속받아 구현한다.  
자세한 것은 아래에서 recv, send 구현 시 다시 다룬다.  

---  

## Listener
accept를 해주는 소켓을 나타낸다. (식당 문지기)  
CP에 등록해야 되는 소켓이므로 IocpObject를 상속받아 구현한다.  
또 IocpEvent 구현 당시 같이 구현해두었던 AcceptEvent의 사용도 필요하다.  
(소켓을 accept해줘야 하므로)  
IocpObject의 인터페이스 함수들의 구현은 거의 동일하므로 넘어가고, 핵심 함수만 간략히 살펴보자.  

* Listener::Dispatch(IocpEvent* iocpEvent, int32 numOfBytes)  
* Listener::ProcessAccept(AcceptEvent* acceptEvent)  
* Listener::RegisterAccept(AcceptEvent* acceptEvent)  
  
이 함수들을 하나의 흐름으로 살펴보자.  

클라를 나타내는 Session 객체를 여기서 생성해준다.  
클라에 대한 정보는 AcceptEvent를 통해 받고, Session 생성 후 다시 AcceptEvent 내부에 저장해준다.  
그 후 SocketUtils::AcceptEx( ... )를 통해 winapi 함수 wrapper를 호출해 winsock의 비동기 accept를 해준다.  

이후 accept가 되면 (비동기적으로 되면) 이걸 iocp가 감지해 IocpCore에서 Dispatch 호출, 여기서 다시 IocpObject(여기서는 Listener)::Dispatch()호출, 그 후 dispatch 함수 안에서 accept된 이후의 일감이 있다는 알림을 받는다. 그후 실제로 accept한 소켓에 대한 일처리를 해주기 위해서 dispatch함수 내에서 Listener::ProcessAccept(AcceptEvent* acceptEvent) 함수를 마지막으로 호출한다.  
아까 위에서 말했듯이 AcceptEvent* acceptEvent내에는 Session 정보, 즉 클라이언트에 대한 정보가 저장되어 있으므로 우리는 여기서 클라와 관련된 모든 정보처리를 해줄 수 있다.  

Dispatch  
![image](https://user-images.githubusercontent.com/63915665/215312019-176ba0ae-0236-4536-bec1-42473da1816f.png)  

ProcessAccept  
![image](https://user-images.githubusercontent.com/63915665/214218201-5d18ea5e-1226-4a2c-ab26-21f09e76d1e6.png)  

RegisterAccept  
![image](https://user-images.githubusercontent.com/63915665/215310774-5bad2a65-52ec-4265-8324-0983b64fd036.png)  
  
최종적으로 이렇게 라이브러리회된 Iocp의 핵심 기능들을 간이로 작동시키기위해 main함수에서 다음과 같이 테스트해줄 수 있다.  
![image](https://user-images.githubusercontent.com/63915665/214219115-30d9f17f-48a1-485a-9c9f-244878695d7c.png)  

여기서 StartAccept는 최초 1회, Listener를 등록해주며 동시에 첫 RegisterAccept()를 호출해주는 과정이다.  
이부분에서 for loop의 사용은 추후 구조 개선 시 변경될 부분이니 일단 넘어가자.  
![image](https://user-images.githubusercontent.com/63915665/215310943-4e82d94e-9229-4a8c-8b25-0f7b11e951a0.png)  

지금까지 구현한 Listener과 accept()를 라이브러리화 된 IOCP로 처리하는 과정의 흐름을 쭉 정리해보자.  
  
그 전에 RegisterAccept()는 블로킹 함수이고, accept에 성공할 때까지 대기함을 명심하자. 성공 후에는 그냥 바로 종료되는데, 예기치 못한 에러(WSA_IO_PENDING이 아닌 나머지 모든 에러)가 발생했을 경우에만 재귀한다. 이는 원래대로라면 accept 성공한 걸 감지한 다른 worker 스레드에서 RegisterAccept()를 실행해 항상 Accept 가능한 상태를 유지해야 하는데, 에러가 나면 성공을 감지를 못해 상태가 끊기기 때문이다.  

1. Listener 등록    
Listener 생성 -> StartAccept() -> Listener CP에 등록, StartAccept()에서 RegisterAccept() 호출 -> RegisterAccept()에서 클라이언트 accept 할때까지 블락 -> 성공 시 CP에 알림 보내면서 종료  
  
2. 클라이언트들 accept  
worker 스레드들 생성 -> 각 스레드별로 IocpCore::Dispatch() -> CP에서 accept 성공 알림받고 IocpObject::Dispatch() 실행 (정확히는 virtual이므로 Listener::Dispatch()) -> Listener::Dispatch() 실행되는 시점에서 Listener과 관련된 IO 이벤트(accept) 처리가 완료되었음을 의미 (즉 타 스레드의 Listener에서 신규 클라이언트에 대한 AcceptEx가 성공했음, 우리의 상황에서 메인 스레드의 RegisterAccept()가 첫번째 클라를 accept 성공한 경우이다) -> worker 스레드에서 Listener::ProcessAccept()를 호출해 Client Connected! 띄우고 RegisterAccept() 실행 -> 마찬가지로 두번째 클라 accept 성공할 때까지 블락 -> 성공 시 CP에 알림 보내면서 종료 -> 종료 후 메인스레드의 while loop에 의해 다시 IocpCore::Dispatch()가 실행되며 다음번 IO 신호를 기다리며 대기  

3. 반복  
다른 스레드들에서도 IocpCore::Dispatch() 돌고 있음 -> CP에서 accept 성공 알림받고 ... (이후 동일)  

즉 스레드들이 서로 돌아가며 RegisterAccept()를 실행하며, 최소 하나의 스레드는 언제나 RegisterAccept()를 실행하는 상태가 유지되어야 한다.  

---  

현재의 구조상에서 한 가지 보완할 점이 있다면 CP에 저장되어있는 IocpObject(Session,Listener 등의 모체)가 클라이언트 튕김 등의 예기치 못한 이유로 worker 스레드에서 일을 처리하던 도중에 삭제될 경우 크래시가 발생한다는 것이다.  
(주의: 클라이언트가 튕긴다고 바로 IocpObject가 날라간다기 보다는, 내부적인 로직(클라가 답신이 없으면 disconnect시키고 Session delete한다던가)에 의해 IocpObject가 삭제되거나 메모리 일부가 오염되는 경우에 더 가깝다)  

이를 어떻게 해결할까?  
방법은 다양하다.  
우선 레퍼런스 카운팅을 이용해 IocpObject의 Reference count를 추적하고 ref cnt가 0이 아닌 이상 절대 삭제하지 않는 방법이 있다. 이걸 직접 구현하기보다는 shared_ptr을 활용하는 게 유리하다.  

우선 그동안 만든 모든 IocpObject들에 대한 shared_ptr 형을 미리 선언해준다. (편의를 위함)  
![image](https://user-images.githubusercontent.com/63915665/215318220-1809ab71-676b-4e55-8ff6-1a2a54b0f45d.png)  
그 후 IocpObject를 shared_ptr로 바로 사용할 수 있도록 해준다.  
이걸 안해주면 가령 IocpObject 내부에서 스스로를 shared_ptr로 사용해야 할 경우 곤란해진다. 
```c++
// 잘못된 방법들
class A
{
	// ...
	shared_ptr<A> a;
	
	a = this; // 에러, this는 shared_ptr<A>가 아니라 A*임
	a = shared_ptr<A>(a); // 에러는 안나지만 심각한 결과를 초래함. shared_ptr가 두개 만들어짐, 고로 ref count를 따로 세게 되어 한쪽이 지워졌지만 다른쪽은 남아있는 상황 등이 발생할 수 있음
}

// 올바른 방법
class A : public enable_shared_from_this<A>
{
	// ...
	shared_ptr<A> a;
	
	a = shared_from_this(); // 내부적으로는 weak_ptr를 사용
}
```  
![image](https://user-images.githubusercontent.com/63915665/215317812-bd6a6680-4a8f-47e8-b953-21f03faa6d3e.png)  

그 후 기존 코드들의 모든 IocpObject 관련 부분들의 type을 shared_ptr을 사용하도록 리팩토링해준다. 또 기존에 IocpObject와 IocpEvent를 둘다 넘기는 방식에서, IocpEvent 내에 IocpObjectRef를 저장한 후 IocpEvent만 넘기는 방식으로 변경한다.  
![image](https://user-images.githubusercontent.com/63915665/215317945-02f42077-7cc3-4749-be90-95d4dee9f032.png)  
```c++
bool IocpCore::Dispatch(uint32 timeoutMs)
{
  DWORD numOfBytes = 0;
	ULONG_PTR key = 0; // -> 추가, 0이라는 값을 줌으로서 사실상 쓰이지 않게 되었다
  // IocpObject* iocpObject = nullptr; -> 삭제
  IocpEvent* iocpEvent = nullptr;
  
  
  if (::GetQueuedCompletionStatus(
    _iocpHandle, 
    OUT &numOfBytes, 
    OUT &key, // OUT reinterpret_case<PLONG_PTR>(&iocpObject), -> 삭제
    OUT reinterpret_case<LPOVERLAPPED*>(&iocpEvent),
    timeoutMs) == true)
  {
		IocpObjectRef iocpObject = iocpEvent->owner; // -> 추가
    iocpObject->Dispatch(iocpEvent, numOfBytes);
  }
  else
  {
    // 편의상 에러 처리 및 timeout 처리 생략
  }
  
  return false;
}
```  

여기까지 수정 후 실행하면 동일하게 잘 작동하는 것을 확인할 수 있다.  

---  

## 2. Sevice 구현 (하나의 세션에 대해 해당 세션과 타 세션 간의 통신을 관리해주는 객체. 서버-클라, 클라-클라, 서버-서버)  





