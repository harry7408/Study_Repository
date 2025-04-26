# Docker 명령어 정리

## Docker Image
- 기본적으로 Docker Hub에서 이미지를 다운받아 활용하게 된다
  
  Docker Hub : https://hub.docker.com/</br>
  -> 이미지 저장 및 다운로드할 수 있는 공간

- 사용자가 직접 이미지를 만들어서 사용할 수 있다

  ⭐ Dockerfile을 만들어서 build를 해야한다

### Docker 이미지 다운로드
  ``` plain-text
docker pull [이미지명]:[태그명]

ex) docker pull mysql
ex) docker pull nginx:stable-per1
  ```
- [이미지명]에 입력한 이미지를 다운 받는다 (태그명을 명시하지 않을 시 latest 버전이 다운된다)
- 태그명을 입력하여 DockerHub에서 관리되는 원하는 버전을 이미지로 받을 수 있다
---
### Docker 이미지 조회 및 삭제
> 이미지 조회
  ``` plain-text
  docker image ls
  ```
<img width="473" alt="Image" src="https://github.com/user-attachments/assets/cf7dd401-8f14-4fe1-92ca-c42f398c9147" />

- Repository : 이미지 이름
- TAG : 이미지 태그명
- IMAGE ID : 이미지 아이디 (Docker에서 임의 생성)
- CREATED : Docker Hub에서 관리되고 있는 이미지 날짜 (이미지 다운 날짜 X)
- SIZE : 이미지 용량

> 이미지 삭제
1. 특정 이미지 삭제
  ```plain-text
  docker image rm [이미지명 | 이미지 ID]
  ```
  - 이미지 ID를 입력할 때는 ID 전체가 아닌 일부 값만 입력해도 Docker에서 알아서 지워준다 (일반적으로 겹치지 않게 ID를 생성하도록 설계한 것 같다)
  - Container에서 사용하고 있지 않은 이미지만 삭제 가능하다
</br>

2. 중지된 컨테이너에서 사용하고 있는 이미지 강제 삭제
  ```plain-text
  docker image rm -f [이미지명 | 이미지 ID]
  ```
  - `-f` 옵션으로 강제로 삭제 가능하다 (force)

    ⭐ **실행 중인 컨테이너에서 사용하고 있는 이미지는 삭제할 수 없다 (삭제 시도 시 에러 발생)**
  
</br>
    
3. 전체 이미지 삭제
  ```plain-text
   docker image rm $(docker image -q)

   docker image rm -f $(docker image -q)
   ```
   - `-f` 옵션이 없는 경우 컨테이너에서 사용하고 있지 않은 이미지 전체 삭제
   - `-f` 옵션이 있는 경우 컨테이너에서 사용하고 있는 이미지 포함하여 전체 삭제
   - `$(docker image -q)`는 시스템에 있는 모든 이미지의 ID를 반환한다
   - `-q` 옵션은 quite를 의미하며 각 이미지의 ID만 표시하도록 한다
