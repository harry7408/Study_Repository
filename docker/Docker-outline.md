## Docker Basic
### Why Docker? (⭐ Portability)
- 독립적인 환경을 구성하여 항상 일관되게 프로그램을 설치할 수 있다 (환경 설정, 옵션, 버전 etc..)
- 컨테이너를 잘 관리해두면 귀찮은 설치 과정들을 매번 하지 않아도 된다
<br></br>
### Docker의 구성
#### Container
- 독립적인 환경을 구성할 수 있는 공간
  
  -> Virtual Box의 각각의 환경, Annaconda의 env 와 유사한 개념

  ⭐ 각 컨테이너마다 저장 공간, 네트워크를 고유하게 가지고 있다

#### Image
- MYSQL, NGINX, Ubuntu 등 Container에서 사용하고자 하는 프로그램 및 환경

  -> Steam에서 구매하는 Steam 게임의 역할 : Steam(컨테이너)에서 게임(Image) 골라서 실행

- Image에는 프로그램을 실행하는데 필요한 환경 설정, 옵션, 버전, 설치 과정 등의 정보가 포함되어 있다

  -> 프로그램 실행에있어 중요한 정보들이 모두 담겨있다

