# 프로젝트 생성 및 수정
### 1. Github에서 Repository 선 생성 후 작업 시작 + 이미 작업 중이던 작업물이 있는 경우

- Github에서 먼저 프로젝트를 생성
- 프로젝트를 Local에 Clone 받아오기
    
    ```bash
    git clone [레포지토리 주소]
    ```
    
- 이미 작업물이 있는 경우는 Clone을 해야하지만 새로 프로젝트를 생성하는 경우 개인적으론 Repository는 먼저 생성 후 Clone을 받아오지는 않는다
</br>

### 2. 로컬에서 먼저 작업 후 Github Repository에 작업 업로드

- 로컬 레포지토리 저장소 생성 (.git 파일)
    
    ```bash
    git init
    ```
    
    - .git 파일에는 원격 Repository에 대한 정보, 사용자 정보, Commit 정보 등 git 관련 모든 정보가 저장된다 → .git  파일을 지우면 커밋 History도 사라지게 된다
    - git init을 통해 생성하면 Git을 설치한 후 최초 브랜치 명이 master이나 이를 main으로 초기에 1번 바꿔주는 것이 좋다
        
        ```bash
        // .git 파일 설정 정보 조회
        git config --list
        
        // main으로 기본 브랜치 변경
        git config --global init.defaultBranch main
        ```
        
- 원격 저장소에 대한 정보를 추가해준다
    
    ```bash
    git remote add origin [Github에서 만든 Repository 주소]
    
    // 원격 저장소 정보 확인하는 방법
    git remote -v
    
    // 연결된 원격 저장소 정보 삭제
    git remote remove origin
    
    // 원격 url 변경하는 방법 (로컬 커밋 내역은 유지)
    // 오픈소스를 Clone 받은 후 내가 만든 Repository에 넣고 싶을 때 활용 가능
    git remote set-url origin [변경할 원격 저장소 주소]
    ```
</br>    

### 오픈 소스 활용할 때

🚧 명시된 라이센스 확인 후 이를 지키는 것은 중요

1. 해당 Repository의 Commit 이력을 가지지 않고 소스를 가져와 내 Repository에 업로드
    - 소스 코드를 Clone 받은 후 .git 파일 삭제 후 다시 git init으로 새로운 .git 파일 생성
    - Github 상에서 Commit 히스토리 확인 후 원하는 Commit 내역에 가서 Browse files에 들어간 후 .zip 파일로 받고 git init으로 .git 파일 생성
    - 두 작업 모두 정보를 모두 날리는 것이기 때문에 remote 정보 추가 + add, commit, push 작업 필요
    
2. 해당 Repository의 Commit History를 가지고 소스를 가져와 내 Repository에 업로드
    - git remote set-url origin [변경 할 주소]로 나의 Repository로 remote origin 을 변경 한 후 push
