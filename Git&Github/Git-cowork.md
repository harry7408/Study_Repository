# Cowork

### fork

- 브랜치를 생성해서 push 할 수 있는 권한이 없는 외부 기여자가 PR을 날리기 위한 목적으로 사용하는 방법
- 또는 다른 사람의 저장소를 쉽게 나의 Github Repository에 가져오는 방법
    - 주로 Opensource에 기여할 때 사용된다
    1. Fork 받아온 곳에서 작업한 후 Commit
    2. PR을 만들어 날리면 해당 레포지토리의 권한이 있는 사람이 검토 후 반영하거나 거절
    

### branch

- branch 생성시 주의사항
    - **항상 최신화된 main에서 feature 브랜치 생성**
        - feature브랜치가 직접 main에 merge되어야 하는 상황이 존재하므로 dev, main 등을 운영할 때에도 **main브랜치에서 feature브랜치 생성하는 것을 권고**
    - feature 브랜치를 작은 기능단위와 짧은 기간으로 유지
    - 수정 예정 프로그램에 대한 중복이 발생하지 않도록 동료간 충분한 의사소통
- 권한
    - branch관리 rule
        - Require pull request **reviews before merging**등의 rule 지정 (리뷰 갯수 설정)
        - **targets에서 rule 적용대상 branch지정 
        → 보통 main branch에 rule을 적용한다**

### Issue

- **jira같은 협업 툴 대신 팀 이슈 생성/완료 등으로 활용 가능한 기능**

### Merge 전략

- merge전략 (PR 생성 후 Code를 Merge할 때 제공되는 3가지 옵션)
    - merge/ rebase / squash
        - merge (브랜치 재사용 가능)
            - merge는 두 브랜치의 변경 사항을 통합하는 기본적인 방법
            - **branch1에서 넘어온 commitID와 신규 merge commitID가 main브랜치에 남게됨 (branch 2개가 합쳐졌다는 크게 의미가 없는 커밋)**
        - rebase (브랜치 재사용이 어려움)
            
            <img width="602" height="521" alt="Image" src="https://github.com/user-attachments/assets/9816ac4b-c143-4a05-855f-659bac9062f2" />
            
            - 한 브랜치의 커밋을 다른 브랜치의 최신 커밋에 “재적용”(re-apply)하는 방식
            - 이때에는 **브랜치에서 넘어온 commitID가 아닌, 새로운 commitID가 발급되어 main브랜치에 생성**
            - 장점
                - **merge commitID는 남지 않게 되어, 불필요한 커밋 없이 깔끔한 커밋관리**
            - 단점
                - 기존의 commit history자체는 유지되지만, **모든 commitID가 변경됨**으로서 이후에 동일 브랜치에서 재pr시 충돌이 발생하므로, 사용하던 branch는 더이상 사용이 어려움
                → 기능 바꾸고 더이상 안쓸거면 rebase가 괜찮은 선택지가 될 수 있다
        - squash (브랜치 재사용이 어려움)
            - squash는 여러 커밋을 하나의 커밋으로 합치는 과정
            - local repository에서 여러 커밋을 발생시켰을때 해당 커밋ID를 통합하여 하나의 commitID로 만들어 dev에는 하나의 commitID로만 이력 생성
            - 장점
                - 불필요한 커밋 없이 깔끔한 커밋관리
            - 단점
                - 기존에 사용하던 branch는 더이상 사용이 어려움
                - 디테일한 Commit으로 돌아가고자 할 때 선택지가 없어진다
            
            ⭐ dev branch를 운영한다면 이 브랜치에서 main에 merge할 때는 절대로 rebase나 squash merge를 사용하면 안된다 (새로운 커밋 아이디가 생성되어 더이상 브랜치 사용이 어려워 지기 때문)
