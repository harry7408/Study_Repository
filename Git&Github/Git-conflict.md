# Pull 받을 때 시나리오 별 상황

### 상황 별 대처

1. Github Repository(origin)과 Local Repository의 Commit 이력이 같은 상황에서 Working Directory에서 작업을 진행하고 pull 받는 경우
    
    ⇒ 충돌 상황이 아닌 경우
    
    - `git pull origin main` 을 해도 변화가 없다 (commit 이력이 동일하기 때문)
    - Working Directory에서 작업 후 Commit을 해 나의 Local Repository가 1개의 커밋이 앞서더라도 pull을 받아올 때 아무런 충돌이 발생하지 않는다
    - origin과 Working Directory의 커밋 이력이 같고 내가 Working Directory에서 작업 한 경우에는 pull을 받을 상황이 아닌 Commit → Push 또는 Working Directory의 작업을 reset해야 하는 상황
    
2. 원격 저장소에만 Commit 이력의 추가가 발생한 상황
    
    → 내 Local에서 작업이 없거나 작업을 해도 Commit 하지 않은 상황에서 팀원이 Github에 Push한 상황
    
    ⇒ 상황에 따라 충돌이 발생 여부가 달라진다
    
    1. 같은 파일을 작업하여 수정한 사항이 겹치는 경우 (충돌 발생 O)
        - git pull을 받는 경우 Git에서 다음과 같은 메세지를 날리며 pull을 막는다
        
        ```bash
        error: Your local changes to the following files would be overwritten by merge:
            /src/main/example.txt
        Please commit your changes or stash them before you merge.
        Aborting
        ```
        
        - 이때 fetch 까지는 성공하나 파일을 **병합하는 과정에서 충돌이 발생**
        - Local에 Commit 이력이 존재하지 않아 merge할 수 있는 병합 파일이 생성되지 않는다
            1. 우선 나의 Local에 Commit을 만든 후 다시 merge 또는 pull을 받아와 충돌 발생 후 해결한 뒤 add → commit → push 수행
            2. 내 Local 에서 작업하던 내용을 취소(backup 본 저장)하고 git pull을 받아 원격 저장소의 내용을 가져온 후 나의 소스를 추가 (fast-forward 후 내 코드 반영)
            3. 일반적인 협업 상황에서는 겹치는 작업량이 많고 겹치는 부분이 많아 Backup 을 만들기 힘들 수 있으므로 `git stash` 기능 활용
        
    
    b. origin에 있는 내용과 내 Local에서 서로 다른 파일을 작업하여 겹치지 않는 경우 (충돌 발생 X)
    
    - git pull을 받으면 Github Repository에 있는 내용이 나의 Local에 반영되어 merge 된다
    
    c. 내 Local에서 작업이 하나도 없는 상황 (충돌 발생 X)
    
    - 원격에 있는 소스를 받아와 나의 Local Commit 이력에 반영되고 나의 작업 이력이 없어 merge 되는 Commit 없이 Branch의 헤더(origin/main 및 main)가 앞으로 이동하는 것이므로 `fast-forward` 라고 지칭한다.
    
3. 원격 저장소에 Commit 이력 추가 발생 및 나의 Local에도 Commit 이력의 추가가 발생한 상황
    
    ⇒ 상황에 따라 충돌 발생 여부가 달라진다
    
    1. 서로 작업한 내용에 겹치는 부분이 없는 경우 (충돌 발생 X)
        - Origin에 있는 내용과 나의 Commit 이력이 각각 쌓이고 이를 합쳐주는 **Merge Commit이 발생**하게 된다
        - 이때는 발생한 Merge Commit을 Origin에 push 해주면 된다 (origin에 있는 내용을 가져와 병합했기 때문에 충돌이 발생하지 않음)
        
    2. 서로 작업한 내용이 같은 경우 (충돌 발생 O ⇒ 가장 빈번한 상황)
        - git pull로 우선 소스를 받아오게 되면 두 Commit ID 간 변경 내용을 비교하여 만들어지는 병합 파일이 생성된다
            - HEAD : 내 Local에서 작업한 내용
            - 하단 내용 : origin/main에서 작업 된 내용
        - 병합된 파일을 개발자가 적절하게 수정하여 충돌 상황을 제거한 후 add → commit을 하게 되면 `Merge 커밋1 커밋2` 형식의 로그를 확인할 수 있다
        - 이후 push 작업 수행
        
        Ref) 위에서 한 작업을 바로 pull 받아오지 않고 **git fetch로 Local Repository에 가져온 후 git diff로 두 소스를 확인 후 merge나 pull을 받아와서 충돌을 발생 시킨 뒤 해결하고 add → commit → push 작업 수행**
