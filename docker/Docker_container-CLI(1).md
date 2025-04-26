# Docker 명령어 정리

## Docker Container (1)

### Dokcer 컨테이너 생성
  ``` plain-text
  docker crate [이미지명]:[태그명]

  ex) docker create --name mysql mysql:latest
  ```
  - 입력한 이미지명 기반으로 컨테이너를 생성한다 (실행은 X)
  - 이미지가 이미 있다면 Container 생성만 한다
  - pull로 이미지 받아오는 것을 create로 생략할 수 있다
  - 생성과 동시에 실행을 해주는 명령어가 있어서 잘 사용하지 않는다
---
### 컨테이너 실행
  ``` plain-text
  docker start [컨테이너명 | 컨테이너 ID]

  ex) docker start mysql (컨테이너 ID는 docker ps로 조회 가능)
  ```
  - 생성된 컨테이너를 실행하는 명령어
---
### 컨테이너 종료
  ``` plain-text
  docker stop [컨테이너명 | 컨테이너 ID]

  ex) docker stop mysql (컨테이너 ID는 docker ps로 조회 가능)
  ```
  - 실행 중인 컨테이너를 종료하는 명령어
  - 삭제를 원한다면 별도 작업 필요
