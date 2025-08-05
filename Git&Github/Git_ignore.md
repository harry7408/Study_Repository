# gitignore
### gitignore 파일

- .gitignore 확장자를 갖는다 (파일 명 필요 없음)
- git에서 추적하는 파일 목록 중 제외하고자 하는 대상을 나열
- 프로젝트 Root 경로 하위에 제작

</br>

### ignore 파일 생성 방법

1. 수동으로 생성
    - 파일 생성 시 .gitignore 확장자로 제작
2. Template를 활용한 ignore 파일 생성
    - Android Studio, IntelliJ에서는 .ignore 이라는 Plugin을 다운 받아서 이를 쉽게 제작할 수 있도록 도와준다 (기본적으로 제외 해도 되는 파일들을 쉽게 포함해줌)

</br>

### gitignore 활용

1. 중요한 정보가 들어가 있을 때 (Token, IP 주소 값, Key 값 등등)
2. 불필요한 소스코드를 올리고 싶지 않을 때 (/.idea, /out  빌드 및 컴파일 된 결과 등등)

</br>

### Git 캐시 삭제

- Git에서 현재 추적하고 있는 파일들을 Staging Area에서 제거하는 방법 (캐시 제거)
- .gitignore을 수정했는데 이미 추적 중인 파일이 계속 커밋에 포함될 때 사용

 ⭐  But Commit 및 Push를 하고 캐시를 지운다 해도 이미 Commit History에 반영되기 때문에 중요한 Key 값 등을 다시 .gitignore에 포함하고 캐시를 지운다 해도 이력에는 남게 되므로 중요 정보는 제작 초기부터 .gitignore 파일에 포함시키는 것이 중요

- Android Studio나 IntelliJ 로 .idea 파일을 처음에 .gitignore에 포함하지 못했을 때 사용하면 좋다
    
    ```bash
    // 캐시 지우기 -> 지우고 .gitignore 파일에 내용 포함 하면 반영됨
    git rm -r --cached .
    ```
