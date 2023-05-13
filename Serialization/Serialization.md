* 직렬화의 필요성?
-> 메모리 주소를 그대로 기록할 수 없다, 왜? 포인터나 동적 할당된 데이터들 등 데이터가 일렬로 메모리 상에 위치하는 게 아니기 때문.  
-> 때문에 JSON이나 XML처럼 어떤 하나의 포맷으로 데이터를 변환하는 과정을 통해 데이터를 쉽게 전달, 수신 후 해석 가능하도록 할 수 있다.  

* 직렬화 방법  
구글 라이브러리 ProtoBuf와 Flatbuffer는 서로 다른 직렬화 기법을 사용하는데, 이 두 기법은 각각 장단점을 가지고 있다.  
https://stackoverflow.com/questions/25356551/whats-the-difference-between-protocol-buffers-and-flatbuffers  
가장 핵심적인 차이는, ProtoBuf는 버퍼를 복사하는 과정이 필요하다는 것, 반면 Flatbuffer은 Zero-copy를 지원해 그냥 버퍼에서 곧바로 데이터를 꺼내다 쓸 수 있다는 것이다.  
대신 ProtoBuf는 다차원 배열과 같이 동적 길이 데이터 내에 동적 길이 데이터가 들어가는 경우 더 사용하기가 간편하다.  

구체적으로 왜 이런 차이가 발생하는지에 대해서는 IOCP 패킷 직렬화1,2,3을 참고하자.  
