### DFS를 이용한 사이클 탐지
알고리즘 자체에 대한 설명은 생략, 1번 락을 잡은 이후 2번 락을 잡는 경우를 1->2 간선으로 표시한 후 해당 그래프에서 역방향 간선을 찾으면 이게 곧 데드락을 발견함을 의미한다.  
데드락을 잡는 클래스 DeadLockProfiler는 글로벌하게 하나만 만들어도 되지만 TLS별로 만들어두어도 됨.  
IOCP 24강 참조  
