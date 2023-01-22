# IOCP 입출력 모델
## 개요
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

## 라이브러리화 (객체지향적으로 구조화)
내용의 특성상 내용들이 파편화된 형태로 기록될 수 있기에 흐름을 쭉 따라가기보다는 필요한 부분을 찾아서 사용하는 걸 권장한다

### IocpCore
IOCP의 핵심 로직들 (completion port 만들고, 여기에 소켓을 등록하고, 소켓들에서 recv하고 등등)을 관리해주는 IocpCore라는 객체를 구현한다.

#### Register()
IocpCore에는 여러 기능들이 있지만 그중 Register()에 대해 잠깐 살펴보자.  

Register()의 목적은 소켓을 CP에 등록하는 것인데, 우리는 이미 위에서 CreateIoCompletionPort() 함수를 이용해 소켓을 CP에 등록하는 법을 배운 적이 있다. 다만 이것보다 조금 더 복잡해지는데, 예전에 우리가 CreateIoCompletionPort에 임시로 제작했던 Session 객체를 넘겨준 것과 다르게 이제 제대로 IocpObject라는 객체를 구현해 넘겨줄 것이다.  

IocpObject는 Dispatch(IocpEvent* iocpEvent, int32 numOfBytes)라는 함수를 가지며 이 함수는 worker thread들에게 일감을 분배하는 역할이다. 

Dispatch가 인자로 받는 IocpEvent는 예전에 Session에서 Enum으로 eventType(Connect, Accept, Recv, Send) 을 저장한 것과 동일하지만 이걸 객체로 한 번 더 감싸줬다고 생각하면 된다.  


