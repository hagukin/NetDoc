# IOCP 입출력 모델
선수지식: Overlapped 모델  
Overlapped 모델을 요약하면 다음과 같다.  
비동기 입출력 함수가 실행되는 동안 스레드는 Alertable wait 상태가 된다.  
함수 실행이 완료되면 쓰레드 별로 있는 APC 큐에 해당 함수가 완료되면 실행될 콜백 함수의 정보와 함께 Alert를 해주라는 요청이 쌓인다.  
이후 APC큐를 비우면서 콜백을 실행하고 그 후 스레드를 Alert해주어 Alertable wait 상태에서 빠져나가게 한다.  

우리가 만들 것은 멀티스레드 IOCP 모델인데, 멀티스레드 IOCP 모델을 한 마디로 요약하면  
Completion Port로부터 비동기 I/O 처리가 완료되었음을 알림받는 함수인 GetQueuedCompletionStatus() 함수를 여러 worker thread들에서 돌리면서 I/O 함수 완료에 대한 후속 처리를 여러 스레드가 분할해서 처리하는 소켓 입출력 구조를 의미한다.  
아직 다루지 않은 개념들이 등장하므로, 글을 더 읽고 나중에 돌아와 다시 확인해보자.  

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
    /*
    아주 중요한 사실 중 하나가, 여러 스레드들에서 가동중인 GetQueuedCompletionStatus 중 단 하나에만 알림이 전달된다는 점이다.
    때문에 서로 다른 두 스레드가 같은 알림을 받아 같은 work을 처리하는 일은 다행히도 벌어지지 않는다.
    */
    
    
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
내용의 특성상 내용들이 난잡할 수 있다.  
또 뒤의 내용에서 앞의 내용에 대한 변경이 있을 수 있으니 (주로 뒤에서 만든 내용을 앞의 것들과 병함시키는 과정에서의 사소한 코드 변화인 경우가 많긴 하지만) 나중에 참조할 일이 있으면 이걸 감안하자.  
구현은 1. Listen(accept 처리), 2. Service 3. Session(recv, send 처리) 순으로 진행된다  

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
worker 스레드에서 (거의) 상시 동작하는 함수로, CP로부터의 알림을 기다리는 함수이다.  
알림이 오면 어떤 객체(IocpObject)에서 알림이 왔는지를 추출해 해당 객체에 맞는 처리를 해주기 위해 IocpObject.Dispatch()를 실행한다.  

IocpCore::Dispatch()는 항상 메인 스레드가 아닌 개별 worker 스레드에서 처리된다는 걸 명심하자.  
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
Listener 생성 -> StartAccept() -> Listener CP에 등록, StartAccept()에서 RegisterAccept() 호출 -> RegisterAccept()에서 클라이언트 accept 할때까지 블락(정정: 정확히 말하면 블락이 아니다. WSA 함수들은 호출 즉시 완료 여부와 관계없이 바로 반환하기 때문에 논블로킹 함수이다. 그러나 처리가 완료가 되어야 CP에 알림을 보낸다는 점에서 처리가 될 때까지 블락한다는 느낌으로 로직을 이해하는 편이 더 직관적이긴 하다.) -> accept 성공 시 CP에 알림 보내면서 종료  
  
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

## 2. Sevice 구현 
Service는 서버 통신 가장 상위단에서의 로직을 관리하는 느낌의 핵심적인 객체로, 기존 main() 함수에 작성했던 로직들이 Service에서 처리된다.  

Service 내에 Session, IocpCore 등을 멤버로 저장한다. (서버(ServerService)의 경우, 내부에 Listener도 멤버로 가진다. ServerService.Start()하면 알아서 내부적으로 Listener만들고 Accept해주는 것이다)  
(Geophyte에서의 Engine과 유사한 느낌의 역할이다)  
  
Service는 자신과 연결된 다른 세션(들)을 관리한다.  
Service는 클라이언트, 서버 모두가 사용하며, Service를 상속받은 ClientService, ServerService를 사용한다.  
Service는 Iocp를 이용해 자신과 연결된 세션들과의 IO를 처리한다. (이말인즉슨 클라이언트 또한 IOCP를 사용해 서버와의 IO를 처리한다는 것이다)  
ServerService의 경우 내부에 Listener IocpObject를 갖고 관리한다. (ClientService에는 없다)  
반대로 Listener 또한 ServerService에 대한 참조를 갖고 각종 정보들을 Service로부터 받아와 사용한다.  
Service는 SessionFactory라는 함수 레퍼런스를 멤버 변수로 갖는데, Service.CreateSession()시 SessionFactory를 사용해 필요한 타입에 맞는 함수를 호출해 Session을 만든다.  
(주석:  
IOCP 강의 내용 중 조금 마음에 들지 않는 부분은 CreateSession()이 Session을 만들고 IOCP에 추가해주는 것까지 한번에 해준다는 것인데, 이 과정을 두 함수로 쪼개야 하지 않나 라는 생각이 든다. 또 StartAccept()내에서 호출한 CreateSession() 이후 만약 함수 내에 뭔가 에러가 발생한다면 CreateSession()에서 IOCP에 추가된 Session은 계속 IOCP에 남아있는게 아닌가, 그러면 성능 상의 저하를 불러일으키는 게 아닌가, 하는 생각이 든다. 나중에 실제로 내 서버를 구현할 때 테스트해보자.)  

Service 구현 및 기존 구조에 Integrate하는 과정과 관련된 코드들은 크게 어렵지 않으므로 생략한다. (IOCP Server Service 중반부부터 참고)  

Service 사용 모습은 다음과 같은 형태이다.  
기존과 로직 자체는 동일하다.  
![image](https://user-images.githubusercontent.com/63915665/215775049-55e4f1ca-dbe0-4baf-bac1-ebef3c342558.png)  
(주석: GameSession의 경우 아직은 클라측에 딱히 뭘 구현한게 없기 때문에 그냥 Session 상속만 한채 그대로 사용중이다.)  


## 3. Session 구현  
Session은 A 입장에서 연결된 상대방의 정보들을 관리하기 위해 만든 객체로, 하나의 세션은 하나의 다른 B와의 연결을 나타낸다.  
A,B라는 표현으로 알 수 있듯이 Session은 서버, 클라이언트 양쪽 모두가 사용한다. (각각 ServerSession, GameSession으로 상속받아 사용)  

Session이 해주는 일은 Recv, Send, Connect, Disconnect 로직의 처리 및 해당 액션들의 처리 이후 해줄 일들에 대한 처리라고 생각하면 된다.  
(OnRecv, OnSend, OnConnect, OnDisconnect. 이 함수들은 RegisterXXX() 내부에서 비동기IO의 처리 이후 넘어온 ProcessXXX()함수 내에서 실행되며, 실제로 IO한 결과를 어떻게 활용할지 여기에다 작성해주면 된다. Recv 시 몇바이트 받았다고 로그 띄우던 것도 OnRecv()내부에서의 구현이다.)  
  
Session은 같은 IocpObject를 상속한 Listener과 굉장히 흐름이 흡사하다.  
실제로 둘 다 ServerService의 멤버변수로 저장된다. (Session이 recv, send, connect, disconnect를 맡고, listener가 accept를 맡는다. 클라이언트의 GameService의 경우 Accept할 일이 없으므로 listener은 없다.)  
때문에 반드시 위 내용들을 숙지하고 오는 것을 권장한다.  
  
디테일을 살펴보기에 앞서 간단히 요약해보겠다.  
둘다 ProcessXXX() 호출 후 RegisterXXX() 호출, 여기서 비동기 winsock 함수 호출한 후 그 함수가 실행완료되어 IOCP로 통보될 경우 대기중이던 worker thread에서 Dispatch 실행 -> Session.Dispatch() 실행, 잔업을 처리한다. (예시의 경우에는 dispatch로 recv 완료 통보를 받을 경우 수신한 패킷의 크기를 콘솔에 출력함)  
잔업 처리 이후 Listener과 마찬가지로 RegisterXXX() 호출을 통해 무한 루프를 돌게 한다. 이후 반복.  
  
위 과정 처리중에 (Listener가 AcceptEvent로 그러했듯이) 정보전달에는 XXXEvent들이 사용된다. 세션의 경우 Listener보다 가능한 행동의 타입들이 다양하다 보니(Recv,Send,Connect) .Dispatch() 에서 이벤트타입에 따라 어떤 액션을 취할지 나눠주는 로직이 처리된다.  

여기서 주의해야 할 것은, Dispatch()를 돌리다가 요청이 오면 처리가 가능한 Recv와 다르게 Send의 경우 보낼 데이터가 있을 때 필요한 시점에 직접 호출해주는 형태이기 때문에 처리 구조가 기존 Recv, Accept와 약간 달라지게 된다는 점이다.  

우선 Recv부터 살펴보자. Session 1 참고  
TODO : Recv 작성  

이번엔 Send를 살펴보자. Session 2 참고  
TODO : Send 작성  

이번엔 Connect, Disconnect를 살펴보자. Session 3 참고  
Connect의 경우,  
Session::Connect() -> RegisterConnect() -> ConnectEx(), CP에서 알림 전송 -> 기다리던 worker thread의 GetQueuedCompletionStatus에서 수신해 ProcessConnect() 호출 -> RegisterRecv() 순으로 처리된다.  
![image](https://user-images.githubusercontent.com/63915665/221198617-896133c5-28aa-4bf9-b8e4-28d704702300.png)  
마지막에 Register**Recv**하는 부분에 유의하자.  
Connect는 필요 시 호출되는 느낌이기 때문에 ProcessDisconnect()가 끝나도 RegisterConnect()를 호출해주지 않는다.  
그러나 Connect 한번 했으면 그 뒤부터는 받을 패킷들에 대비해야 하기 때문에 RegisterConnect가 아닌 RegisterRecv를 실행해 루프가 이어지도록 한다.  

이번에는 Disconnect를 살펴보자. Disconnect는 현재 SocketUtils::Close를 이용해 소켓을 닫아버리고 있다. 이걸 Connect와 유사하게 Disconnect() -> RegisterDisconnect() -> DisconnectEx() -> 타 스레드에서 Dispatch 후 ProcessDisconnect() -> **종료** 순으로 호출되고, DisconnectEx의 파라미터 중 TF_REUSE_SOCKET을 이용해 소켓을 재사용할 수 있게 만들어준다. (실제 재사용 로직은 나중에 추가하고, 일단 재사용 가능함을 winsock 단에서 표기하는 것이라고 생각하자.)  
Connect와 다르게 RegisterRecv를 실행하지 않는데, 당연하게도 이는 상대방과의 연결이 끊겼으므로 더 이상 해당 worker 스레드는 더이상 해당 IocpObject에 관해 뭘 더 해줄 필요가 없기 때문에 다시 IocpCore::Dispatch()를 실행하러 가는 것이다.  
참고:  
![image](https://user-images.githubusercontent.com/63915665/221200416-658fc39e-542f-44f5-af4a-482af3d85114.png)  


## 4. RecvBuffer 구현  
사실 recvBuffer이 이미 있긴 한데, 이건 우리가 그냥 데이터 전송이 잘 되나 확인하기 위해 임시로 만들어둔 BYTE(=char) recvBuffer[1000]; 즉 char array에 불과하고, 이를 제대로 객체화시켜서 버퍼 정책같은 걸 적용하기 쉽게 만들어보자.  

한 가지 짚고 넘어갈 점은 RecvBuffer와 SendBuffer은 별도의 개별적인 객체로 구현할 것인데, 이는 둘의 버퍼 정책이 다르기 때문이다.  

우선 Recv의 경우, 하나의 Service(클라 혹은 서버)에 대해 항상 하나의 Recv()만이 작동중이라는 것을 알 수 있다.  
A라는 Service로 누군가 패킷을 보내면 대기중이던 RegisterRecv() 내의 WSARecv()에서 처리하고, 이걸 worker 스레드 **1개**가 알림을 받아 ProcessRecv() -> OnRecv()를 처리하고, 이게 다 끝나면 해당 worker 스레드가 RegisterRecv()를 다시 대기시킴으로써 처리된다.  
결국 Recv 로직을 처리하는 모든 과정에서 WSARecv()는 딱 하나의 스레드에서만 동작할 수 있다는 것이다. 덕분에 recvBuffer은 멀티스레드 관련 문제로부터 매우 자유롭다.  

이 사실을 인지한 상태로 코드를 살펴보자.  

전:  
![image](https://user-images.githubusercontent.com/63915665/221202595-4b4b44b9-53f6-4f36-b3c3-6a0bdcefd5bb.png)  
![image](https://user-images.githubusercontent.com/63915665/221202689-82f13bb6-cd3a-4eb3-9232-c7393bb544cc.png)  

현 방식의 문제점은 뭘까?  
1. 패킷이 쪼개져서 올 수도 있다. (TCP)  
우리는 현재 Register을 할 때마다 \_recvBuffer을 .buf로 사용하고, 또 최대 크기도 1000만큼 갖는 WSABUF를 만들어서 거기에 Recv를 하고 있다. (즉 실질적으로 저장되는 공간은 char array인 \_recvBuffer이다)  

그런데 우리가 사용중인 TCP의 특성 상 x바이트를 보냈을 때 한번에 x바이트만큼 다 온다는 보장이 없다.  
x 바이트중 일부만 왔을 수도 있는데, 우리의 코드에서는 이걸 무시하고 그냥 RegisterRecv()가 한번 걸리면 그대로 쭉 처리해버린 후 OnRecv()에서 바로 \_recvBuffer 내용을 출력해주고 있다. 즉 한 번에 모든 정보가 왔음을 상정한 채 진행해버리고 있는 것이다.  

그동안이야 데이터 크기가 작았으니 별 문제 없었지만, 크기가 커지면 당연히 정보가 분할될 가능성이 높아진다. 때문에 이에 대한 수정이 필요하다.  

이 문제에 대한 한 가지 해결법은 패킷 앞부분 헤더에 데이터의 크기를 기입하는 것으로 전 처리하는 것인데, 이건 나중에 구현해보자.  

2. 버퍼의 크기가 유한한데, 이걸 계속 재사용중이다.  
지금이야 계속 같은 길이의 패킷을 보내고 있으니 위 방식대로 매번 WSABUF가 같은 버퍼(\_recvBuffer)를 사용해도 별 문제가 없지만, 만약 매 번 발송하는 패킷의 크기가 바뀐다면 버퍼를 비우거나 하지 않는 현재 코드의 로직상 불필요한 정보가 같이 출력되거나 데이터가 섞이거나 하는 참사가 발생할 수 있다.  

이를 해결하는 방법은 다음과 같다.  
일단 read, write cursor을 버퍼에 배치한다.  
1) Circular buffer 사용  
끝까지 데이터가 차면 다시 앞에서부터 채우는 방식  
구현이 다소 까다롭다.  
2) 유사 Circular buffer 사용
write cursor가 끝까지 갔다면(=데이터가 차면) 현재 read cursor부터 write cursor까지의 데이터를 복사해 버퍼의 맨 앞으로 가져다 놓는 방식.  
r~w까지의 데이터만이 사용 가능성이 있는 데이터이기에 사용할 수 있는 방식이다.  

우선 2번 방식으로 구현해보자. (나중에 Circular buffer 구현도 해보자)  

후:  
세부적인 구현 코드는 생략한다. 크게 이해하기에 어렵지 않다.  
![image](https://user-images.githubusercontent.com/63915665/221352589-fae0e9cc-e9d1-4d83-abdd-1d3898f4ab9c.png)  
핵심적인 역할을 하는 함수가 Clean()인데, 주기적으로 버퍼 데이터를 앞으로 끌고와주는 역할로, 안쓰는 데이터들을 지우는 게 아니라 그냥 커서를 사용해 데이터를 덮어써버리는 것을 알 수 있다.  
![image](https://user-images.githubusercontent.com/63915665/221353044-38a53cfc-497b-437b-b279-786d66d18f23.png)  
이렇게 데이터를 앞으로 복사해오는 횟수를 최대한 줄여 성능을 개선시키기 위해 사용하는 방법이 바로 BUFFER_SIZE와 BUFFER_COUNT의 사용인데, 세부적인 내용은 IOCP RecvBuffer 참고.  

아래 코드를 비롯해 Session에서 약간의 수정들을 거치면 사용이 가능하다.  
![image](https://user-images.githubusercontent.com/63915665/221352865-25de0e1b-dcee-4e20-ad1b-f91905df570d.png)  


## 5. SendBuffer 구현  
기존에 임시로 만들어둔 SendBuffer는 하나의 정보를 발송하기 위해 한 번의 Send() 호출이 필요했다. 이를 큐를 이용해 개선하여, 한 번에 여러 정보를 하나의 버퍼에 모두 복사해놓고 한 번에 버퍼를 발송해버리는 Scatter-Gather 방식을 사용할 것이다. SendBuffer의 앞부분에서 이 내용을 다룬다.  
(WSASend()함수에 우리가 그동한 1로 고정해두었던 dwBufferCount라는 파라미터에 버퍼의 수를 넣어주면 된다)  

추가로, 게임서버에서는 같은 정보를 여러 클라이언트에게 보내야 하는 상황이 잦은데(e.g. x지점에 몬스터가 스폰), 이를 Broadcasting이라고 한다. 이런 경우 매번 동일한 내용의 패킷을 복사해서 버퍼만들어주고 하는 것보다 같은 패킷을 재활용하는게 유리하기 때문에 이 역시 구현한다.  
구현 방식은 GameSessionManager라는 별도의 클래스를 구현하고, 이 클래스에서 GameSessionManager::Broadcast(SendBufferRef sendBuffer) 함수를 호출하는 형태로 이루어진다.  

(기타 수정사항: 예전에 만들어둔 DeadLockProfiler에 대한 수정이 필요하다. 기존 코드가 왜 문제가 되는지는 SendBuffer 참고)  

SendBuffer를 할당할 때 큰 크기를 하나 잡고 보내는 현재의 방식을 Pooling 기법을 적용해 개선할 수 있는데, 관련된 내용은 나중에 기회가 되면 별도로 다뤄보자. 불필요하게 큰 덩어리의 메모리를 계속 할당할 필요가 없어지니 당연히 성능 향상이 이루어진다. (SendBuffer Pooling 참고)  
















  
