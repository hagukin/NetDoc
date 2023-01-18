## 논블로킹 소켓을 사용해 소켓 입출력을 구축하는 모델들  
용어 정의: Recv(), Send()와 같이 소켓에서 처리되는 네트워크 송수신을 소켓 입출력(I/O)라고 한다.  
비동기 IO와 같은 표현이 등장 시 여기서는 키보드 IO같은 하드웨어적 의미가 아닌, 비동기 소켓 통신이라고 이해하면 되겠다.  

### Select 모델  
Select 함수가 핵심이 되는 소켓 모델이다.  
주 아이디어는 소켓 함수(recv등)의 호출했을 때 성공할 수 있는지를 미리 알 수 있다는 것이다.  
소켓 셋(일종의 소켓 배열)을 이용하는데, read, write 두 개의 소켓 셋(정확히는 예외적인 상황들을 담당하는 exception까지 세 개이지만 거의 두 개만이 사용된다)을 만들고 select()에 이 세트들을 넘기면 select는 각 소켓 셋 속에 들어있는 소켓들이 지금 시점에서 read/write를 처리할 수 있는지를 판단하고, 가능한 소켓들만 set에 남기고 아닌 소켓들을 지운 후 남은 소켓들의 갯수를 반환한다.  
이후 어떤 특정 소켓이 read또는 write가 가능한지 확인하고, 가능할 경우 해당 동작(ex. recv())을 처리해주면 된다.  
다행히 이걸 직접 비트필드를 활용해 구현해줄 필요는 없고, 윈도우에서 제공하는 기능을 사용하면 된다.  

사실 아직까지 얼마만큼의 성능상의 차이가 있는지는 모르겠다. 다만 코드의 구조가 훨씬 깔끔해지는 건 확실하다.  
단점도 존재하는데, 소켓 셋 속에 들어갈 수 있는 소켓의 최대 갯수가 윈도우 기준으로 64개까지 가능한데, 때문에 64명 이상의 클라이언트를 받으려면 소켓 셋을 그만큼 많이 사용해야 한다.  

(IOCP 44 참고)  

### WSAEventSelect 모델  
소켓에서 발생하는 네트워크 이벤트를 **이벤트 객체** 를 통해 파악하는 방식.  
```c++
WSAEventSelect(소켓, 네트워크 이벤트 감지를 위해 소켓과 연동된 이벤트 객체, 감지할 이벤트의 종류(들));
// Tip: 감지할 이벤트의 종류는 long인자 하나를 받는데, 이때 여러 이벤트를 다 감지하고 싶으면 long var = FD_ACCEPT | FD_CLOSE; 식으로 비트연산을 활용해 long을 만들고 넘겨주면 된다.  

WSAWaitForMultipleEvents(...);
// select 모델의 select함수와 매우 유사함, 이벤트들의 목록 중에서 네트워크 이벤트가 처리된 이벤트의 인덱스를 반환한다. (정확히는 이벤트 인덱스에 약간 변형을 한 값이지만 이 값에서 index를 추출할 수 있음)
// 앞서 말했듯이 감지할 이벤트의 종류를 여러 개 넘겨줄 수 있으므로, WSAWaitForMultipleEvents()가 네트워크 이벤트가 실행된 이벤트의 인덱스를 반환하더라도 정확히 어떤 네트워크 이벤트가 실행되었는지를 모를 수 있음. 
// 이를 해결하기 위한 게 바로 WSAEnumNetworkEvents()임.

WSAEnumNetworkEvents(...);
// 소켓에서 정확히 어떤 네트워크 이벤트가 실행되었는 지 반환함
```
Select 모델과 거의 유사한 흐름으로 진행되며 예제 코드를 보면 쉽게 이해할 수 있음.  
매 루프마다 소켓셋을 초기화해줘야 했던 select와 다르게 한 번 이벤트들을 만들어놓으면 계속 사용할 수 있다.  
다만 소켓셋에 갯수 제한이 있었던 것처럼 한 번에 주시할 수 있는 이벤트의 최대 갯수도 제한이 있다.  
(필요하면 더 만들면 된다. 그러나 이게 복잡해지기 때문에 IOCP를 사용하는 것.)  
(IOCP 45 참고)

### Overlapped 모델
Async-Nonblocking 방식이다.  
![image](https://user-images.githubusercontent.com/63915665/211182338-575661c3-c5f8-4bd7-a659-8f377c4e0f53.png)  
위 사진에서 Signal이 Event 방식이라고 생각하면 된다  

#### A. Event 방식
![image](https://user-images.githubusercontent.com/63915665/211185659-55969b1a-649f-40fc-a05d-92342cc1a0c3.png)  
핵심함수는 WSAWaitForMultipleEvents로, 비동기 IO 함수가 완료되면 Event를 Signaled로 바꿔주고, WSAWaitForMultipleEvents는 Signal되있나 살펴보는 역할을 한다. (주의: 한번에 살펴볼 수 있는 최대 이벤트(=소켓)갯수는 64개로 제한된다. 이 내용은 바로 위 WSAEventSelect 모델에서 다룬 적 있다)  

#### B. Callback 방식
![image](https://user-images.githubusercontent.com/63915665/211186187-b89301ee-3b35-43bc-8910-059a6aeb0325.png)  
  
비동기 IO 함수가 바로 처리되지 않았을 경우 해당 스레드를 Alertable wait 상태로 넘긴다.  
![image](https://user-images.githubusercontent.com/63915665/211186005-446a8ec5-e3b9-45ab-9412-651d38dfc82e.png)  
즉 WSARecv()가 바로 처리되지 않고 처리까지 대기시간이 걸릴 경우, SleepEx나 WSAWaitForMultipleEvents 함수 등을 이용해 해당 스레드 자체를 Alertable wait로 넘겨버리고(SleepEx의 활용이 훨씬 권장되는데, 이유는 WSAWaitForMultipleEvents를 사용하려면 소켓 하나당 이벤트 하나를 연결해주어야 하고, 이렇게 만들 수 있는 이벤트의 최대 갯수에 제한이 있기 때문이다), 커널에서 IO가 끝났을 때 이를 통보해주어(Alert) 다시 진행되게 만든다.  
다시 진행되므로 이때 콜백이 실행되고, 콜백이 끝나면 스레드가 Alertable wait 상태에서 탈출한다.  
![image](https://user-images.githubusercontent.com/63915665/211186259-0724fef2-6760-4a26-8d98-39ea78a86e32.png)  
주의할 사항은 위 코드를 보면 콜백함수인 RecvCallback()을 직접 호출하는 코드는 어디에도 없음에도 불구하고, 위 코드에서 SleepEx가 끝나고(즉 alertable wait에서 탈출하고) 곧장 자기 알아서 RecvCallback을 호출한다.  
(IOCP 47)  

또 하나 주의사항은, 한번에 Recv,Send 등을 여러번 해주었다고 해서 APC 큐에 여러번 진입하는 것은 아니다.  

Callback 방식의 장점은 소켓별로 일일히 이벤트를 만들어줘야 될 필요가 없어서 다량의 클라이언트에게 패킷을 보내기가 Event방식보다 용의하다는 것이다.  

그런데 WinAPI를 보면 Callback 함수의 형태가 지정되어있음을 알 수 있는데, 그 인자들이 하나같이 별 유용하지 않은 정보들인 것을 살펴보면 알 수 있다.  
즉 다시 말해, 우리는 콜백함수가 실행되어도 어떤 클라이언트에 의해 실행되었는지 현재로써는 알 방법이 없다는 것이다.  
그러나 이를 교묘하게 해결하는 방법이 있는데, 함수의 세번째 파라미터인 overlapped가 우리의 Session 구조체 내에 들어간다는 점을 이용해, Session의 비트 가장 앞부분에 Overlapped가 들어가게 한 후 세번째 인자를 Session으로 Typecast해버리면, 우리는 Session내의 다른 정보들에도 액세스 할 수 있게 된다.  
![image](https://user-images.githubusercontent.com/63915665/211186572-c525dc01-dda5-489e-b235-50ef70bf9ad9.png)  
![image](https://user-images.githubusercontent.com/63915665/211186673-261e83b4-cbd0-4a06-98ff-d313ba85d9fa.png)  


### Completion Port (IOCP) 모델
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

```c++
// 주요 함수

CreateIoCompletionPort(...)
/*
Completion Port를 생성해줄 때도 사용되고,
어떤 특정 소켓을 Completion Port에 등록해 줄때도 사용한다.
*/

// 사용예시 (IOCP - Completion Port 모델 참고)
// CP 생성
// 여기서는 iocpHandle이라는 이름으로 Completion Port (혹은 CP에 접근 가능한 객체)를 생성중이다. 편의상 일단 iocpHandle = CP라고 이해하고 넘어가자.  
HANDLE iocpHandle = ::CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
// ...
// 소켓을 CP에 등록
// accept() 이후 clientSocket라는 소켓 정보를 가지고 있는 상황
CreateIoCompletionPort((HANDLE)clientSocket, iocpHandle, (ULONG_PTR)session/*session의 메모리 주소값*/, 0);
// 해당 소켓에 대해 최초 1회 Recv를 해주어야 한다
// ...



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
    ::GetQueuedCompletionStatus(iocpHandle, &bytesTransferred, (ULONG_PTR)&session, (LPOVERLAPPED*)&overlappedEx, INFINITE); 
    // 앞에서 CreateIoCompletionPort에서 session의 주소값을 ULONG 으로 넘겼었는데, 이 주소값을 이용해 session을 이 스레드에서 복원시켜준다.  
    
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
    // 세부적인 코드들은 생략
    // ...
    ::WSARecv(session->socket, &wsaBuf, &flags, &overlappedEx->overlapped, NULL);
  }
}
```



## 각 모델들의 장단점
* Select 모델  
장점: 윈도우, 리눅스 공통 -> 연결할 대상이 어차피 서버 하나밖에 없는 클라이언트를 구축할 때는 멀티플랫폼 환경에 유리한 Select 모델이 적합할 수 있음.  
단점: 성능 최하(매번 등록해줘야 함), 소켓 64개 제한  
* WSAEventSelect 모델  
장점: 비교적 뛰어난 성능  
단점: 소켓 64개 제한  
* Overlapped - 이벤트 기반  
장점: 성능  
단점: 소켓 64개 제한  
* Overlapped - 콜백 기반  
장점: 성능  
단점: 모든 비동기 소켓 함수에서 사용가능하지는 않다, 예시로 accpet에서는 콜백 overlapped 구조 사용 불가.  
또 빈번한 alertable wait로 성능 저하가 우려됨.  
* IOCP  
클라이언트 다수를 대상으로 서버 모델로는 현재로썬 가장 이상적임.  

## 참고자료
### 참고하면 좋은 개념  
Reactor Pattern  
뒤늦게 무언가를 처리하는 구조.  
소켓 상태 확인 후 뒤늦게 recv나 send를 호출하는 형태가 Reactor pattern에 해당.  
Select, WSAEventSelect 모델이 해당.  
  
Proactor Pattern  
미리 무언가를 처리하는 구조.  
미리 recv, send를 호출한 후 그 완료 시점에 따라 처리하는 형태가 Proactor pattern에 해당.  
Overlapped 이벤트, 콜백이 해당.  
