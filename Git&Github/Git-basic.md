# Git 기본 명령어

### 상태 파악

- 현재 작업중인 디렉토리와 Staging Area의 상태를 보여주는 명령어
    
    ```bash
    git status
    ```
</br>    

### 추적을 위한 준비

- 버전 관리를 하고자 하는 소스들을 Staging Area(Commit 될 대상을 파악)에 올리는 명령어
    
    ```bash
    // 현재 디렉토리 기반 모든 소스들을 Staging Area에 추가
    git add .
    
    // 특정 파일만 추가
    git add [특정 파일]
    ```
    
- git add 이후에는 git status 명령어로 Staging 상태를 확인할 수 있다

</br>

### 버전 관리 시작

- Commit을 하면 Commit ID가 생성되며 Local Repository에 이력이 남게 된다
    
    ```bash
    // 헤더는 필수 사항이고 나머지는 선택 사항
    git commit -m "헤더" -m "본문" (-m "Footer")
    ```
    
- git commit만 입력하게 되면 메세지 입력모드로 전환되어 vi 편집기에서 첫째 줄에는 Title 두 번째 줄부터 본문이 입력 된다

> 번외
> 
- add 와 커밋 과정을 동시에 수행할 수도 있따
    
    ```bash
    git commit -am "헤더"
    ```
</br>

### 원격 저장소에 Local Repository의 이력 업로드

- 연결된 GitHub Repository에 소스 업로드
    
    ```bash
    // main 브랜치에 업로드 하는 경우
    git push origin main
    
    // 특정 브랜치에 업로드 하는 경우
    git push [특정 Branch 명]
    OR
    git push
    ```
    
- 충돌이 나서 업로드가 막힐 때는 `--force` 옵션을 추가해서 강제로 업로드할 수 있으나 혼자 공부할 때만 사용하는 것을 권장 (협업 시 좋지 않음)

</br>

### 커밋 이력 조회

- 지금까지 발생한 Commit History를 조회
    
    ```bash
    // 일반적인 조회 방법 (현재 Branch)
    git log
    
    // 간결한 형태로 조회
    git log --oneline
    
    // 그래프 형태로 조회 (브랜치 및 Merge 된 사항을 파악 가능)
    git log --graph
    
    // 현재 Checkout 된 브랜치 이외에 관계된 모든 Branch 이력을 조회
    git log --all
    ```
    
    - `git log` 명령어는 현재 Checkout 된 브랜치의 HEAD 부터 올라가는 커밋 이력을 보여준다
    - `git log --all` 명령어로는 현재 나의 branch와 원격 저장소와 나의 저장소 상황 비교할 때 사용(협업 시 다른 사람이 Push 한 경우 origin/main이 나의 main 보다 앞서게 된다 ⇒ fetch 명령어 입력 후 확인 가능)
        - main : 내 Local Repository의 Main Branch
        - origin/main : 원격 저장소의 Main Branch
        - HEAD : 현재 내가 작업 중인 커밋을 가리키는 포인터  (현재 Checkout 되어있는 Branch를 가리킨다)
    - `--grpah` 옵션의 경우 명령어 입력보다 소스트리 같은 Tool 또는 Github의 Insights → Network 항목에서 쉽게 그래프의 형태를 파악할 수 있다
    - Github에서 Commit 내역에 `Verified` 마크가 생성되는 것은 GPG 서명된 커밋이거나 Github 웹 UI 상에서 파일을 수정하면 자동으로 Github 서명 키를 이용해서 Commit 되기 때문에 붙는다

</br>

### HEAD 전환 방법

- 과거 커밋 이력 또는 다른 Branch로 전환
    
    ```bash
    // 특정 커밋으로 이동 (git log로 커밋 ID 조회 가능)
    git checkout [특정 Commit ID]
    
    // 브랜치 전환
    git checkout [브랜치 명]
    ```
    
    - 특정 Commit ID를 입력할 땐 앞에 4~5개만 입력해도 알아서 인식해준다

</br>

### 원격 저장소의 내용을 가져오기 (소스 코드 까지)

- Github Repository에 쌓인 작업 내용을 Work space에 가져오기 **(fetch + merge)**
    
    ```bash
    // 현재 Checkout 된 브랜치 기반 
    git pull 
    
    // 명시적으로 브랜치 지정
    git pull origin [브랜치 명]
    
    // upstream 확인 방법
    git branch -vv
    ```
    
    - `git pull` 명령어를 사용하면 현재 Checkout 된 브랜치가 어느 원격 브랜치와 연결 되어있는지 확인한 후 그 Branch 기반으로 fetch와 merge 작업을 수행
    - `git pull origin main` 과 같이 명시적으로 원격과 브랜치를 지정하면 어떤 브랜치를 추적 중인지 관계 없이 `origin/main` 을 가져와 merge 또는 rebase 수행

</br>

### 원격 저장소의 History 가져오기 (소스 코드 제외)

- Github Repository에 쌓인 작업 내용을 Local Repository에 가져오기
    
    ```bash
    git fetch origin [브랜치 명]
    ```
    

- 나의 Working Space에 이력 가져오기 (Fetch 작업 선행)
    
    ```bash
    git merge [브랜치 명]
    
    ex) git merge origin/main
    		git merge origin/feature-login
    ```
    
    - fetch를 하게 되면 내 Local Repository에 `origin/` 이름이 붙은 Branch가 생성된다
        - **`origin/` 이 접두어로 붙은 브랜치는 나의 Local Repository에서 원격 저장소 상태를 추적하는 역할을 하는 원격 추적 Branch이다 (remote-tracking Branch)**
    - pull이 더 간편하지만 fetch → merge 작업 이전에 git diff 명령어로 변경 내용을 확인할 수 있다는 장점이 있다

</br>

### 커밋 기반 변경

- 내 Branch의 Commit 들을 다른 Branch의 최신 Commit 뒤로 옮겨 붙이는 방법
- 원격의 내용과 내 작업 내용 OR 두 작업 브랜치에서 Commit History에 차이가 있을 때 소스 코드 합칠 시  유용
- 기반을 바꾼다고 해서 `re-base` 라는 이름이 붙은 것 같다
    
    ```bash
    git rebase [기준 브랜치 명]
    
    ex) 나의 모든 작업 사항 Commit 
    		git fetch origin main
    		git rebase origin/main
    ```
    
    - 위의 예시에서 main branch 기반으로 GitHub에 내용이 추가 되었을 때 추가된 내용과 내가 작업한 내용이 합쳐지길 원할 때 fetch로 Local Repository에 소스를 가져온 후 rebase를 하게되면 나의 작업과 원격에 있는 소스가 합쳐진다 (충돌이 나지 않는다고 가정)
    - **Branch의 시작점을 다른 Commit으로 바꾸게 되므로 기존의 나의 Commit 들이 새로운 Hash 값이 생성되어 Commit ID가 변경된다**
    
    > 협업 시 Branch를 만들어서 작업 후 main 기반 Rebase를 수행하는 과정
    > 
    - main에는 직접 push하지 않고 pull request를 통해 관리한다고 가정
        - 이때 나의 작업물은 최신화 된 main branch의 내용을 포함하고 있어야 한다
    
    ```bash
    // main을 pull 받아서 rebase
    1. 나의 작업내용 Commit ** 매우 중요
    2. git checkout main
    3. git pull origin main
    4. git checkout featrue-login
    5. git rebase main
    
    // main 을 fetch 후 rebase
    1. 나의 작업 내용 Commit
    2. git fetch origin
    3. git rebase origin/main
    ```
    
</br>    

### 차이 비교

- Commit  또는 Branch 간 작업 된 내용 비교 (앞에 입력한 옵션이 기준점)
    
    ```bash
    // 두 커밋 간 비교
    git diff [commit 1] [commit 2]
    
    // 두 브랜치 간 비교
    git diff [브랜치명 1] [브랜치명 2]
    
    // 브랜치와 Commit 섞어서 사용
    git diff [브랜치 명] [Commit ID]
    
    // 변경된 파일 목록만 조회
    git diff --name-only
    
    // 변경 요약 (추가/삭제 라인 수)
    git diff --stat
    
    // 특정 파일만 비교
    git diff [commit 1] [commit 2] -- <file>
    ```
