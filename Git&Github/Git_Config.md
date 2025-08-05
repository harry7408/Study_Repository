# Git Config 설정
### 이름 및 이메일 설정

- Local의 Git의 config 파일에 이름과 이메일 설정이 필요하다(누가 Commit 및 Push 하는지 식별하기 위함)
    
    **⭐ Github은 Settings → Emails 항목에 있는 이메일이 식별자 역할을 하여 Config 설정 시 Commit을 매칭하려면 email 항목을 일치 시켜야한다**
    

1. 전역적 설정
- Window 기준 C:Users/사용자이름/**.gitconfig** 파일 안에 설정 값이 들어간다
    
    ```bash
    git config --global user.name "사용자 이름"
    
    git config --global user.email "깃허브 이메일과 매칭"
    ```
    

1. 지역적 설정
- 해당 Repository의 .git/config 파일에 값이 설정된다
    
    ```bash
    git config --local user.name "사용자 이름"
    
    git config --local user.email "깃허브 이메일과 매칭"
    ```
</br>    

### 설정된 값 확인 방법

- 특정 Repository 안에서 조회하면 그 Repository에 대한 지역 정보가 표시되고 Repository 밖에서 조회 시 전역 정보가 나타난다
1. 개별 값 조회
    
    ```bash
    // 이름 조회
    git config user.name
    
    // 이메일 조회
    git config user.email
    ```
    

1. git 관련 모든 설정 값 조회
- 아무 옵션 없이 입력하면 System, Global, Local 3가지 범위 중 우선순위가 가장 높은 값이 출력 된다
    
    ```bash
    git config --list [조회하고자 하는 범위]
    
    // 조회 이후 q를 눌러 빠져나갈 수 있다 -> Vim Editor 기반으로 만들어졌기 때문
    ```
