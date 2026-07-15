# Use-After-Free(UAF) 취약점 분석

## 1. Use-After-Free(UAF)란?
- Use-After-Free(UAF)는 동적으로 할당된 Heap 메모리가 free()를 통해 해제된 이후에도 해당 메모리를 계속 사용하는 메모리 안전성 취약점이다. C/C++과 같이 개발자가 직접 메모리를 관리하는 언어에서 자주 발생하며, 정보 유출, 임의 읽기·쓰기, 권한 상승, 원격 코드 실행(RCE) 등 보안 문제로 이어질 수 있다.
- 프로그램은 일반적으로 malloc()을 통해 Heap 메모리를 할당하고, 사용이 끝난 후 free()를 호출하여 메모리를 반환한다. 만약 free()를 호출하지 않으면 메모리가 계속 점유되어 Memory Leak이 발생하며, 반대로 free() 이후에도 해당 메모리를 계속 참조하면 UAF 취약점이 발생한다.

## 2. Dangling Pointer의 발생 원리
- UAF는 대부분 Dangling Pointer로부터 시작된다.
- 예를 들어 하나의 Heap 객체를 두 개의 포인터가 동시에 참조하고 있다고 가정한다.
```
char *f = malloc(100);
char *g = f;
```
- 이때 f와 g는 모두 동일한 Heap 영역을 가리킨다.
- 이후 ```free(f)를 호출하면 Heap 메모리는 해제된다. 하지만 g는 여전히 같은 주소값을 가지고 있으므로 이미 해제된 메모리를 계속 가리키게 된다.
- 이러한 포인터를 Dangling Pointer라고 한다.
- 중요한 점은 free()는 포인터를 삭제하는 것이 아니라 Heap 메모리를 Heap Allocator에게 반환하는 함수라는 것이다. 따라서 포인터 변수 안에 저장된 주소값은 그대로 남아 있으며, 프로그램은 여전히 해당 주소를 접근할 수 있다고 착각하게 된다.
  
* Heap Allocator: Heap 메모리를 관리하는 컴포넌트. 동일한 크기의 malloc()이 호출되면 Heap Allocator는 성능 향상을 위해 기존에 사용했던 공간을 다시 재사용하는 경우가 많다.

## 3. Heap Grooming
- Heap Grooming은 Heap Allocator의 동작을 예측하여 공격자가 원하는 위치에 원하는 객체를 배치하도록 Heap의 상태를 의도적으로 조작하는 공격 기법이다.
- 정확히 말하면 공격자가 Heap의 배치(Layout)를 설계하는 과정이다.
- Heap이 다음과 같이 존재하면
```
A
B
C
D
```
이러고 
```
free(B)
free(D)
```
를 수행하면
```
A
FREE
C
FREE
```
이 싱태가 된다.
이후에 ```malloc(sizeof(B))``` 을 수행하면 Heap Allocator는 대부분 첫 번째 FREE 공간을 재사용한다.
즉, 
```
A
X
C
FREE
```
이런식이 된다.
- 이러한 성질을 이용하여 공격자는 malloc, free를 수천번 반복하여 Heap의 배치를 원하는 형태로 만든다.
