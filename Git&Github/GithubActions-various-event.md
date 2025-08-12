# Github Actions의 다양한 이벤트

<details>
  <summary>Push</summary>

### Push

- github 레포지토리에 Push 발생 시 Trigger 되는 Event

<img width="840" height="323" alt="Image" src="https://github.com/user-attachments/assets/7c1117d5-af6d-4148-a8e6-6201cdeff1fb" />

- Activity Type이 별도로 존재하지 않는다

> Ex) push.yaml
>

```yaml
name: push-workflow
on: push # push 발생 시 Trigger (공식 문서에 나온 사용법)

jobs:
  push-job:
    runs-on: ubuntu-latest # ~에서 실행하겠다
    steps:
    - name: step1 # 이름은 생략이 가능하긴하다
      run: echo hello world
    - name: step2
      run: | # Multi Line Command 작성 방법 ('|')
        echo Multi
        echo Line
```

- 해당 파일을 Repository에 `.github/workflows`  경로 안에 설정하면 Repository에 push가 발생할 때마다 Trigger 되어 Workflow가 실행된다.                                                                               


</details>

<details>
  <summary>Pull Request</summary>

### Pull Request

- Github에서 PR(Pull Request)이 발생 시 Trigger 되는 이벤트

<img width="831" height="670" alt="Image" src="https://github.com/user-attachments/assets/1e6eb969-a7d9-476d-83db-e901976a0d60" />

- Activity Type이 여러 Case가 존재한다
    - Activity Type을 지정하지 않으면 Default로 `open`, `synchronize`, `reopened` 3가지가 발생할 때마다 Workflow가 Trigger 된다

> EX) pull_request.yaml
>

```yaml
name: pull-request-workflow
on:
  pull_request:
    types: [opened] #Activity Type 지정하는 방법
    # PR이 처음 오픈 될때만 트리거 되도록 지정

jobs:
  pull-request-job:
    runs-on: ubuntu-latest
    steps:
    - name: step1
      run: echo hello world
    - name: step2
      run: |
        echo hello world
        echo github action

```

- on을 사용해서 Activity Type을 지정할 수 있다
</details>

<details>
  <summary>Issue</summary>

### Issue

- Github에서 Issue 생성 시 Trigger 되는 Work-flow

⭐ 반드시 Settings에 설정 되어있는 Default Branch에서만 동작한다

<img width="783" height="518" alt="Image" src="https://github.com/user-attachments/assets/7efd7f3a-024a-41b0-b3dc-a61387139acf" />

- 모든 Activity Type이 Default로 되어있어 Activity Type을 명시하지 않으면 Trigger 된다

> EX) issue.yaml
>

```yaml
name: issue-workflow
on:
  issues:
    types: [opened] # Open 될때만 Trigger 되도록 설정

jobs:
  issue-job:
    runs-on: ubuntu-latest
    steps:
    - name: step1
      run: echo issue-workflow-start
    - name: step2
      run: |
        echo Hello
        echo World
```
</details>

<details>
  <summary>Issue Comment, Pull Request Comment</summary>
  
### Issue Comment

- 이슈 및 Pull Request에 Comment를 달았을 때 Trigger되는 Work-flow

⭐  Issue와 마찬가지로 Default Branch에서만 동작한다

<img width="787" height="215" alt="Image" src="https://github.com/user-attachments/assets/4132531b-4145-4724-8ff5-1eb6b7e3c2f2" />

> Ex) issue-comment.yaml
>
- if문에 따라 Pull Request에 Comment를 남길 때와 Issue에 남길 때로 분기

```yaml
name: issue-comment-workflow

on: issue_comment

jobs:
  pr-comment:
    if: ${{ github.event.issue.pull_request }}
    runs-on: ubuntu-latest
    steps:
    - name: pr comment
      run: echo ${{ github.event.issue.pull_request }}

  issue-comment:
    if: ${{ !github.event.issue.pull_request }}
    runs-on: ubuntu-latest
    steps:
      - name: issue comment
        run: echo ${{ github.event.issue.pull_request }}

```
</details>

<details>
  <summary>Schedule</summary>
  
### Schedule

- 특정 시점에 Work-flow가 실행되도록 설정 가능하다
- 지정한 시간에 정확하게 동작하지 않을 때도 있다(부하가 있으면 Delay)

⭐ Default Branch에서만 동작 가능하다

<img width="840" height="205" alt="Image" src="https://github.com/user-attachments/assets/e03b6da7-60ff-4da1-91e4-a580748cd1c8" />

<img width="810" height="279" alt="Image" src="https://github.com/user-attachments/assets/7c9be470-50f1-49e2-8949-71ecc9d98362" />
</details>

<details>
  <summary>Workflow Dispatch</summary>
  
### Workflow dispatch

- 사용자가 직접 수동으로 Trigger 가능 (Actions에서 우측에 Runworkflow 햄버거 버튼 클릭 후 Run workflow 버튼 클릭)
- Input 값을 넣을 수 있다

  <img width="809" height="342" alt="Image" src="https://github.com/user-attachments/assets/5602230c-d97e-4466-9e9c-45ddbec19749" />


⭐ Default Branch에서만 동작 가능하다

<img width="796" height="198" alt="Image" src="https://github.com/user-attachments/assets/c9025c49-c54f-496e-bd7d-592dafc28e79" />

> Ex) workflow-dispatch.yaml
>

```yaml
name: workflow-dispatch

on:
  workflow_dispatch:
	  # inputs 설정 가능
    inputs:
      name:
        description: '이름 설정'
        required: true
        default: '테스트'
        type: string
      environment:
        description: '선택'
        required: true
        default: '선택 1'
        type: choice
        options:
        - '선택 1'
        - '선택 2'

jobs:
  workflow-dispatch-job:
    runs-on: ubuntu-latest
    steps:
      - name: step1
        run: echo hello world
      - name: step2
        run: |
          echo hello world
          echo github action
      - name: step3
        run: |
          echo "이름 : ${{ inputs.name }}"
          echo "선택 : ${{ inputs.environment}}"

```
</details>

<details>
  <summary>다양한 Event로 하나의 Workflow Trigger</summary>

### 다양한 이벤트

- 다양한 이벤트로 하나의 Workflow를 Trigger 하는 방법
- on: 항목에 Trigger 될 이벤트들을 나열하면 된다

> Ex) various-event.yaml
>

```yaml
name: various-event-workflow

on:
  issues:
    types: [opened]
  push:
  workflow_dispatch:

jobs:
  various-event-job:
    runs-on: ubuntu-latest
    steps:
      - name: step1
        run: echo hello world
      - name: step2
        run: |
          echo hello world
          echo github action
```
</details>

<details>
  <summary>Needs</summary>

### Needs

- 기본적으로 Github Action의 Job들은 병렬로 실행
- 하나의 Workflow에서 여러 Job을 실행할 때 Job 간에 종속성을 만들어 하나의 Job 완료 되어야 다른 Job이 실행 되도록 할 때 사용 (Job의 실행 순서를 조절 가능)

  <img width="685" height="372" alt="Image" src="https://github.com/user-attachments/assets/51c65e0f-5418-448f-8063-a9461e2f1cf0" />


> Ex) needs.yaml
>

```yaml
name: needs-workflow

on: push

jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
    - name: echo
      run: echo "Job1 Finish"

  job2:
    runs-on: ubuntu-latest
    needs: [job1]
    steps:
      - name: echo
        run: echo "Job2 Finish"

  job3:
    runs-on: ubuntu-latest
    steps:
      - name: echo
        run: |
          echo "Job3 failed"
          exit 1

  job4:
    runs-on: ubuntu-latest
    needs: [job3]
    steps:
      - name: echo
        run: echo "Job4 Finish"

```
</details>

<details>
  <summary>Re Run</summary>

### Re Run

- 과거에 실행 했던 Workflow를 재실행
- 성공 및 실패 여부와 상관 없이 재실행 가능 (Trigger 된 시점을 다시 실행)
- 파일을 수정하고 단순히 Re-run을 통해 다시 워크플로우를 실행해도 수정한 내용이 반영되지 않고 수정된 파일이 포함된 후에 Workflow가 Trigger 되어야 반영된다
- 실행 했던 WorkFlow에 가서 우측 상단에 Re-run Jobs를 누르면 재실행 한다

<img width="886" height="199" alt="Image" src="https://github.com/user-attachments/assets/d1ebbd07-ddaf-4749-9236-10975e4c5385" />
</details>

## 추가 Event

- 공식 문서 확인하기

  📎 https://docs.github.com/ko/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows

  <img width="620" height="1061" alt="Image" src="https://github.com/user-attachments/assets/e8a06f4e-83e1-48b8-acfe-bda908acc8a3" />

- 메세지

    <img width="832" height="106" alt="Image" src="https://github.com/user-attachments/assets/17026b57-d6a8-4283-a1b6-114d2b1a6e6c" />

    - Summary에 보면 Triggered via ~ 라고 어떤 이벤트에 의해 Workflow가 실행되는지 파악할 수 있다
    - Workflow Dispatch 같은 경우에는 Mannually 라고 직접 실행했다는 메세지가 나온다