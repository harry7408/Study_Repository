# Git Branch

### Git Branch

- 브랜치는 Git 저장소의 **특정 시점에서 작업을 분리해서 독립적으로 개발을 진행할 수 있도록 도와주는 도구
→ 일회성 작업 이후 merge 되면 삭제하는 것이 좋다**

### Git branch 전략

1. Feature Branch
    - 각 기능별로 독립적인 Branch를 생성하여 작업하는 방법
    - 서로 작업에 영향을 주지 않고 효율적인 작업이 가능하다
    
    ```bash
    1. 기준 브랜치(main)에서 새로운 Feature 브랜치 생성
    	ex) git checkout -b feature/settings
    	- 일반적으로 [feature/기능명]의 형태로 구성 (해당 기능을 설명하는 명칭 적용)
    	
    2. 새로 만든 feature branch에서 작업 진행 (복수의 Commit이 발생해도 무방)
    
    3. 기준 브랜치(main)로 병합하기 전에 원격 저장소에 feature Branch push
    
    4. Code 리뷰 후 기준 브랜치에 feature 브랜치에서 작업 한 내용 merge
    
    5. merge 후 필요에 따라 작업했던 브랜치 삭제 (삭제 권장)
    ```
    

1. GitHub Flow
    - 규모가 작은 프로젝트에 적합
    - 빠른 개발 주기와 지속적인 배포에 초점을 둔다
    - Pull Request를 날려 보다 협업을 잘 할 수 있다
    - Github의 Issue와 연동하여 Branch를 만들면 Github Flow전략을 쉽게 사용 가능
    
    ```bash
    1. 기준 브랜치(main)에서 새로운 브랜치 생성
    	ex) git checkout -b splash-screen
    	- 일반적으로 브랜치 명은 작업 내용을 설명하며 
    		기준 브랜치는 항상 배포 가능한 상태 유지
    
    2. 작업 진행 후 Commit, push
    
    3. Github에서 PR을 생성하여 코드 리뷰 요청
    
    4. 코드 리뷰 후 기준 브랜치로 병합하고 PR 종료
    
    5. 필요에 따라 작업했던 브랜치 삭제 (삭제 권장)
    ```
    

1. Git Flow
    - 대규모 프로젝트에서 코드 관리와 릴리즈를 체계적으로 진행하는 방법
    - 기준 브랜치 외에 일반적으로 develop 브랜치, Hotfix 브랜치 등을 두어 관리한다
    - Main은 항상 배포 가능한 상태를 유지해야하기 때문에 Main Branch에 Protection Rule을 적용해야한다
    
    ```bash
    // 기능 개발
    1. Develop 브랜치에서 Feature 브랜치를 생성
    2. 기능 개발 완료 후 Develop 브랜치로 Merge
    
    // 배포 준비
    3. Develop 브랜치에서 Release 브랜치 생성
    4. 버전 번호를 부여하고 문서 작업 등 Release 관련 작업 진행
    
    // 배포 작업
    5. Release 브랜치를 Main 브랜치로 merge
    6. main에 머지할 때 해당 Commit에 Tag를 부여하고 Release 버전을 명시
    7. Release 브랜치는 Develop 브랜치로 merge
    
    // 버그 수정
    8. Main 브랜치에서 발견된 버그는 Hotfix 브랜치를 생성하여 수정
    9. 수정 후 main 브랜치와 develop 브랜치로 merge
    ```
    
    > 각 Branch 별 역할
    > 
    
    ```bash
    Main : 상용화 되어 배포되는 안정적인 코드가 저장되는 브랜치
    Develop : 개발 중인 코드를 관리하는 브랜치 (GitHub-Flow 전략의 main의 역할)
    Feature : 새로운 기능 개발을 위한 브랜치 (Develop 브랜치로부터 분기)
    Release : 새로운 버전 배포를 준비하는 브랜치 (Develop 브랜치로부터 분기)
    Hotfix : 긴급한 버그를 수정해야할 때 사용하는 브랜치 (Main 브랜치로부터 분기)
    ```
    

### Branch 관련 명령어

```bash
// Local 저장소에 있는 모든 브랜치 목록 확인 (현재 checkout 된 브랜치는 다르게 표시됨)
git branch

// 원격 저장소 branch 목록까지 모두 조회
git branch -a

// 브랜치 생성 명령어
git branch [브랜치 명]
	-> 기존에 Checkout 되어있는 브랜치 Commit Base로 신규 브랜치가 생성된다
		(기준을 잡을 브랜치 확인 주의)

// 브랜치를 생성하며 전환
git checkout -b [브랜치 명]

// 특정 로컬 브랜치 강제 삭제 (다른 브랜치로 전환 후 삭제를 해야한다)
git branch -D [브랜치 명]

// 병합된 브랜치만 삭제
git branch -d [브랜치 명]

// 원격 브랜치 삭제 (push 명령어로 지워달라고 요청하는 것임)
git push origin --delete [브랜치 명]
	-> 깃허브에서 GUI로 직접 지울 수도 있음
```

### 참고

- fetch 관련 옵션
    
    ```bash
    // 모든 브랜치 fetch (삭제된 브랜치 제외)
    git fetch --all
    
    // 모든 브랜치 fetch (삭제된 브랜치 포함)
    git fetch --all --prune
    ```
    

- clone 관련 옵션
    
    ```bash
    // 모든 브랜치 이력 포함 Clone (push도 같은 옵션을 주면 된다)
    git clone --mirror [저장소 주소]
    ```
