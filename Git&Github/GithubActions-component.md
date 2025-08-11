# Github Actions 컴포넌트

## yaml 파일 설정 시 큰 뼈대가 된다 (이해하고 있어야 함)

### Workflow

- 하나 이상의 작업을 실행하는 구성 가능한 자동화 프로세스
- .github/workflows 경로에 yml, yaml 파일이 있어야 Workflow가 실행된다
    
    <img width="1460" height="352" alt="Image" src="https://github.com/user-attachments/assets/c8fa0607-1776-43b7-b0cb-3237fcb44f6c" />
    
- 상단의 경로에 설정 파일이 있다면 여러 워크플로우 생성 및 사용이 가능하다
    
    <img width="955" height="330" alt="Image" src="https://github.com/user-attachments/assets/557c820e-1a84-4b60-b022-2b27e574e729" />
    

### Event

- 워크플로우 실행을 Trigger 하는 깃허브 Repository 내 특정 활동
- Trigger 방식
    - Repository 내 이벤트 기반 (ex. push, pull request, 이슈 생성 등)
    - 수동으로 Trigger 가능
    - 스케줄 기반 Trigger
    - Repository 외부에서도 Trigger 가능

> Ex) Push Event 기반 Workflow 동작
> 

<img width="599" height="619" alt="Image" src="https://github.com/user-attachments/assets/0cdb7726-21fc-4349-acf2-100e459d7b7f" />

### Runner (runs-on에 영향?)

- Job을 실행할 수 있는 Server로 각 Runner는 하나의 Job을 실행한다
- 지원되는 Runner
    - Linux
    - Windows
    - MacOS

💡 Docker의 Container와 유사한 역할을 한다고 보면 될 것 같다

### Job

- Runner에서 실행되는 Step의 집합
- Action과 Script 등을 Step에 정의해서 사용한다
- 기본적으로 각 Job은 **병렬로 실행된다**
    
    → 순차적으로 실행할 수도 있다 (needs 설정 활용)
    

### Step

- Job이 실행하는 개별 명령
- **순차적으로 실행되며 한 Step이 실패하면 그 다음 Step 부터는 실행되지 않는다**

### Action

- 특정 작업을 수행하는 코드 조각
- `uses` 키워드를 사용해서 action을 Load 한다
- 하나의 Step에서는 하나의 Action만 가능
- Action의 종류
    - Github에서 공식으로 제공하는 Action
    - Github Community에서 만든 Action (Docker Image를 만드는 Docker Hub 과 유사)
    
    → Github Market Place에 가면 있다  (📎 [Marketplace · Tools to improve your workflow](https://github.com/marketplace))
    
    - 사용자가 Custom해서 직접 만든 Action
