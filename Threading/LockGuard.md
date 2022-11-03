#### Lock Guard란?  
간략하게 요약:  
뮤텍스로 락을 잠궜다 풀어줬다를 수동으로 해주는 과정에서 까먹거나 하면 문제가 발생할 수 있기 떄문에, 애초에 락을 걸어주는 시점에서 뮤텍스 대신 LockGuard를 사용하면 LockGuard 생성시 알아서 락을 걸어주고 해당 LockGuard가 스코프에서 벗어났을 경우 알아서 락을 해제해준다. RAII(Resource Acquisition Is Initialization) 철학에 입각한 작동방식이다.  

내부적으로는 mutex를 사용해 락을 걸어준다.  

#### Unique Lock Guard
락 가드와 동일하나 락을 거는 시점을 수동으로 지정할 수 있다.  

[참고글](https://a-researcher.tistory.com/22)
