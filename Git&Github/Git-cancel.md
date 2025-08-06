# 취소

- 하단의 취소 상황 중 Commit 이후 취소와 Push 이후 취소 이외에는 개발 Tool UI로 쉽게 작업할 수 있다
    - Android & IntelliJ IDEA 경우 `Git -> Commit -> Roll back` 으로 작업한 내용을 쉽게 날려버릴 수 있음 (다른 IDE 툴도 대부분 기능 제공)
 
### Step 별 작업 취소

1. Working Directory의 수정 사항 취소 (Add 이전 취소)
    - `git status` 명령어 입력 시 “untracked file”라고 표시
    - 현재 작업한 Code를 Github과 비교하며 직접 지우거나 UI로 제공된 기능을 사용 (Rollback | Discard)

1. Staging Area에 반영 이후 취소 (Add 이후 취소)
    - `git status` 명령어 입력을 하면 “Changes to be committed“라고 표시
    - UI에서 제공하는 기능 사용 (Unstage로 돌리겠다)
    - 명령어로 Staging Area에서 제거
        
        ```bash
        git reset
        
        git restore --staged .
        ```
        
    
2. Local Repository 반영 이후 취소 (commit 이후 취소)
    - Local Repository의 HEAD 포인터를 옮기기
        
        ```bash
        // 가장 최신 커밋 취소 (unstaged 상태로 만들기 + Working Directory 유지)
        git reset HEAD~1 | git reset HEAD^
        
        // 가장 최신 커밋 취소 (staged 상태 유지 + Working Directory 유지)
        git reset --soft HEAD~1
        
        // 가장 최신 커밋 취소 (staged 상태 취소 + Wokring Directory 작업 되돌림)
        git reset --hard HEAD~1
        ```
        
        - HEAD~ 뒤에 숫자가 몇 번째 전 커밋으로 돌릴지를 의미한다 (다른 Commit ID로 입력도 가능)
        - 커밋을 되돌린다면 이미 Commit ID가 생성되어 History에는 남아있고 참조가 끊기는 형태
        - `--hard` 옵션을 사용하면 Commit History까지 삭제 (reflog로 복구는 가능)
        
3. 원격 저장소에 배포된 이후 취소 (push 이후 취소)
    - 이전에 잘못 Push를 했다면 이 내용들은 완전히 없앨 수는 없다
        - key, Token, 비밀번호 등 중요 정보를 Push 했다면 revert로도 소용이 없다
    
    ```bash
    // 되돌리고자 하는 Commit ID를 입력
    git revert [Commit ID]
    ```
    
    - 기존의 커밋 버전을 취소시키는 새로운 Commit을 생성 후에 다시 Push하는 과정
        - 이때 add, commit을 할 필요가 없다 (새로운 Commit ID 가 생성되었기 때문)
    - 위의 명령어를 입력하면 vi 편집기가 등장해 commit message를 작성하도록 나타난다
        - `Revert ~` 형태의 커밋 메세지의 Header를 만들어서 제공
    
    ⭐ Github에 올라간 소스를 지울 수 있는 방법도 있긴하지만 협업 시 사용하지 않는 것이 좋다 
    
    ⇒ 원격 Branch의 히스토리 자체를 재작성 하는 것이므로 작업 내용이 꼬일 수 있다
    
    - git reset —hared HEAD~1로 커밋 히스토리 제거
    - git push origin main —force 로 강제로 밀어넣기
