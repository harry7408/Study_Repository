# Github Actions의 다양한 기능들

<details>
  <summary>checkout</summary>
  
  ### Checkout

- 깃헙 레포지토리를 가져와 작업을 수행 (마켓 플레이스에 정의된 공식 Action)
- 레포지토리 소스를 가져와 테스트 및 빌드 작업 수행, CI/CD Workflow에서 활용 가능

📎 [Checkout · Actions · GitHub Marketplace](https://github.com/marketplace/actions/checkout)

> Ex) Readme 파일을 읽어 출력하는 Workflow
> 
- checkout 하지 않은 경우 no such file 에러 발생

```yaml
name: checkout

on:
  workflow_dispatch:

jobs:
  no-checkout:
    runs-on: ubuntu-latest
    steps:
    - name: check file list
      run: cat README.md

  checkout:
    runs-on: ubuntu-latest
    steps:
    - name: use checkout action
      uses: actions/checkout@v4
    - name: check file list
      run: cat README.md
```
</details>

<details>
  <summary>context</summary>
  
  ### context

- Workflow를 실행하는 환경에 대한 정보를 제공
- 워크플로우를 유연하게 활용하고 싶을 때 사용
    - Ex) github Context의 branch 정보를 가져와 브랜치 별 Job을 실행하도록 Workflow 구성 가능
- 다양한 Context 존재

<img width="239" height="447" alt="Image" src="https://github.com/user-attachments/assets/4654d95b-babb-4e28-b424-4753404231c0" />

> Ex) Github Context의 Property 출력
> 
- toJson 메서드로 Object를 String으로 변환

```yaml
name: context-workflow

on:
  workflow_dispatch:

jobs:
  context:
    runs-on: ubuntu-latest
    steps:
    - name: github context
      run: echo '${{ toJson(github) }}'
    - name: cat github context
      run: |
        echo ${{ github.repository }}
        echo ${{ github.event_name }}
```
</details>

<details>
  <summary>branch filter</summary>
  
  ### Branch Filter

- 특정 Branch에서 실행
    - Ex) main Branch로 push할 때만 실행되도록 설정

> Ex) main Branch에 Push 할 때 Workflow 발생
> 

```yaml
name: branch-filter-workflow

on:
  push:
  # 여기에 필터링 하고 싶은 브랜치 목록 넣기
    branches: ["main"]

jobs:
  branch-filter-job:
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
  <summary>path filter</summary>

  ### Path Filter
  
  - 특정 경로에서 실행
    - Ex) domain directory 내의 파일이 갱신될 때만 실행되도록 설정 가능
    - 특정 파일 변경 시 Workflow Trigger가 되는 것을 막고자 하면 `!`를 붙이면 된다

> Ex) 특정 경로 설정
> 

```yaml
name: path-filter-workflow

on:
  push:
    paths:
    - 'src/main/domain/model/*'
    - '!src/main/domain/repository/*'

jobs:
  path-filter-job:
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
  <summary>tag filter</summary>
  
  ### Tag Filter

- 특정 Tag에서 실행
    - Ex) v1.0으로 Tagging 해야 실행되도록 설정
- Push Event에서만 사용할 수 있다

> Ex) Tag Filter 활용
> 

```yaml
name: tag-filter-workflow

on:
  push:
    tags:
    - 'v[0-9]+.[0-9]+' # v0.0 과 같은 형식이 필요

jobs:
  tag-filter-job:
    runs-on: ubuntu-latest
    steps:
      - name: step1
        run: echo tag-filter-start
      - name: step2
        run: |
          echo Hello
          echo World
```
</details>

<details>
  <summary>timeout</summary>
  
  ### Timeout

- 특정 시간 이상 실행되면 자동으로 중단되는 기능 (timeout-minutes)
- 특정 Job이나 Step이 무한루프가 발생하면 불필요한 자원 소모가 되는데 이를 방지
- 기본 값으로 6시간
- Job Level, Step Level에서 사용 가능

> Ex) Timeout 활용
> 

```yaml
-------------------------------- Job Level
name: timeout-workflow

on: push

jobs:
  timeout:
    runs-on: ubuntu-latest
    timeout-minutes: 2 # 2분 제한
    steps: # 무한루프
    - name: loop
      run: |
        count=0
        while true; do
        echo "seconds: $count"
        count=$((count+1))
        sleep 1
        done
    - name: echo # 첫번째 step 실패로 job이 취소
      run: echo hello
      
 ---------------------------- Step Level
 name: timeout-workflow

on: push

jobs:
  timeout:
    runs-on: ubuntu-latest
    timeout-minutes: 2 # 2분 제한
    steps: # 무한루프
    - name: loop
      run: |
        count=0
        while true; do
        echo "seconds: $count"
        count=$((count+1))
        sleep 1
        done
      timeout-minutes: 1 # 첫번째 step이 1분안에 완료되도록 제한
    - name: echo # 2분 걸리면 2번째 job 실패
      run: echo hello

```
</details>

<details>
  <summary>cache</summary>
  
  ### Cache

- 자주 사용되는 데이터를 저장하여 빠르게 불러올 수 있도록 한다 (Github 마켓플레이스에 정의된 공식 Action)
- 의존성 설치 시간을 단축시킬 수 있다
- 처음에 Cache에 저장된 것이 없으면 `Cache not found` 있다면 `Cache Hit 및 restore` 발생
- 캐시가 발생하면 좌측 Management 사이드 메뉴에서 캐시를 관리할 수 있다

> Ex) Node js 의존성 빠르게 설치하기
> 

```yaml
name: cache-workflow

on:
  push:
    paths:
      - 'my-app/**' # 원하는 경로 입력

jobs:
  cache:
    runs-on: ubuntu-latest
    steps:
    - name: checkout
      uses: actions/checkout@v4
    - name: setup-node
      uses: actions/setup-node@v3
      with:
        node-version: 18
    - name: Cache Node.js modules
      uses: actions/cache@v3
      with:
        path: ~/.npm
        key: $${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-
    - name: Install dependencies
      run: |
        cd my-app
        npm ci
    - name: npm build
      run: |
        cd my-app
        npm run build
```
</details>

<details>
  <summary>artifact</summary>
  
  ### Artifact

- upload-artifact, download-artifact 2가지 Action이 공식적으로 정의되어있다
- 워크플로우 실행 중 생성된 파일들의 모음
    - 직접 사용자에게 보여지진 않고 Github 서버에 임시 저장되며 사용자는 Job 하단 부분에 Artifacts 항목에서 이를 관리할 수 있다
- 동일 Workflow 내에서 job 사이에 데이터를 공유한다
- Workflow가 종료된 후에도 데이터가 3달 동안 유지
- 필요한 경우 생성된 파일을 다운로드 받을 수 있다 (서로 주고 받는 아티팩트 이름이 일치해야한다)

> Ex) 아티팩트 Upload 및 Download
> 

```yaml
name: artifact-workflow

on: push

jobs:
  upload-artifact:
    runs-on: ubuntu-latest
    steps:
    - name: echo
      run: echo hello-world > hello.txt
    - name: upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: artifact-test
        path: ./hello.txt

  download-artifcat:
    runs-on: ubuntu-latest
    needs: [upload-artifact]
    steps:
    - name: download artifact
      uses: actions/download-artifact@v4
      with:
        name: artifact-test # 아티팩트 다운로드
        path: ./
    - name: check
      run: cat hello.txt
```
</details>

<details>
  <summary>output</summary>
  
  ### Output

- 한 Job에서 생성된 데이터를 여러 Step, Job 간 데이터를 쉽게 공유할 수 있도록 한다
    - 동일한 Job의 Step 또는 다른 Job의 데이터를 공유
    - 다른 Job에서 Output을 사용하려면 needs 와 Job Level의 Output을 만들어야 한다
- Key - value 형태의 단순한 값을 전달할 때 사용
- `echo “{key}={value}" >> $GITHUB_OUTPUT` 형태로 사용 가능
- 동일한 Key값에 대한 방어가 필요하기 때문에 Output이 정의된 Step의 고유 ID를 사용해야 한다 (각 Step에 고유한 ID)

> Ex) Output 활용
> 

```yaml
name: output-workflow

on: push

jobs:
  create-output:
    runs-on: ubuntu-latest
    outputs:
      test: ${{ steps.check-output.outputs.test }}
    steps:
    - name: echo output
      id: check-output
      run: |
        echo "test=hello" >> $GITHUB_OUTPUT
    - name: check output
      run: |
        echo ${{ steps.check-output.outputs.test }}
  
  get-output:
	  # 종속성이 있어야 output 값 활용 가능
    needs: [create-output]
    runs-on: ubuntu-latest
    steps:
    - name: get output
      run: echo ${{ needs.create-output.outputs.test }}

```
</details>

<details>
  <summary>environment variable </summary>
  
  ### Environment Variables

- step이나 job에서 사용할 수 있는 환경 변수
- key-value 형태로 저장

⭐ 동일한 job에서만 데이터 공유 가능

- 환경 변수 구성 방법
    - env 사용 : workflow 내에서 정의
        - workflow level (우선 순위 가장 낮음)
        - job level
        - step level (우선 순위 가장 높음)
        - `echo “{key}={value}" >> $GITHUB_ENV` 형태로 사용 가능
    
    > Ex) env-1.yaml
    > 
    
    ```yaml
    name: environment-variable-workflow
    on: push
    
    # 환경 변슈
    env:
      level: workflow
    
    jobs:
      get-env-1:
        runs-on: ubuntu-latest
        steps:
        - name: check env
          run: echo "LEVEL ${{ env.level }}"
    
      get-env-2:
        runs-on: ubuntu-latest
        env:
          level: job
        steps:
        - name: check env
          run: echo "LEVEL ${{ env.level }}"
    
      get-env-3:
        runs-on: ubuntu-latest
        env:
          level: job
        steps:
          - name: check env
            run: echo "LEVEL ${{ env.level }}"
            env:
              level: step
    
      get-env:
        runs-on: ubuntu-latest
        steps:
        - name: create env
          run: echo "level=job" >> $GITHUB_ENV
        - name: check env
          run: echo "LEVEL ${{ env.level }}"
    ```
    
    - 미리 값 정의
        - 미리 환경변수 정의 후 ${{ vars.[정의한 환경 변수 이름] }}
        - 정의된 환경변수를 바꾸면 Workflow 결과가 달라질 수 있다
        - Github Repository 의 Settings → 좌측 메뉴의 Secretes and variables → Actions의 Variables 탭에서 New Repository Variable로 환경 변수 제작 가능
        
        > EX) env-2.yaml
        > 
        
        ```yaml
        name: environment-variable-workflow
        on: push
        
        jobs:
          get-env:
            runs-on: ubuntu-latest
            steps:
              - name: create env
                run: echo ${{ vars.level }}
        ```
</details>

<details>
  <summary>secrets</summary>
  
  ### Secrets

- 민감한 데이터를 안전하게 저장하여 Workflow에서 활용
- Settings → Secrets and Variables의 secrets 항목 선택 후 제작
- 안전한 저장
    - Github에서 안전하게 암호화
- 로깅 방지
    - 로그에 기록되지 않고 마스킹 된다
- 접근 제한
    - Workflow 실행 중에만 접근 가능
- `${{ secrets.시크릿 변수 명 }}` 으로 활용 가능

> Ex) secrets.yaml
> 

```yaml
name: secrets-workflow
on: push

jobs:
  secrets-job:
    runs-on: ubuntu-latest
    steps:
      - name: get secrets
        run: echo ${{ secrets.API_KEY }}
```
</details>

<details>
  <summary>environment</summary>
  
  ### 환경변수 & Secrets

- 특정 환경에서만 사용 가능한 환경 변수와 Secrets 관리
- Organization Level (가장 낮은 우선순위)
    - Organization Settings에서 설정
- Repository Level
    - Repository Settings에서 설정
- Environment Level (가장 높은 우선순위)
    - Repository Settings에서 Environments 항목에서 만든 후 값들 설정 가능

<img width="861" height="711" alt="Image" src="https://github.com/user-attachments/assets/4206f08a-0f45-463f-aaa8-85891c7e5ce0" />
</details>

<details>
  <summary>matrix</summary>
  
  ### Matrix

- 변수 기반으로 여러 Job을 실행하는 기능
- 변수로 Java Jdk version을 설정한다면 하나의 Job을 구성하여 같은 코드베이스를 각 버전별 테스트가 가능해진다
- `jobs.strategy.matrix.변수명` 으로 설정 가능하며 `${{ matrix.변수명 }}` 으로 활용할 수 있다
- 변수의 모든 가능한 조합에 대해 별도의 잡을 생성한다

> Ex) matrix.yaml
> 

```yaml
name: matrix-workflow
on: push

jobs:
  matrix-job:
    strategy:
      matrix:
        os: [windows-latest, ubuntu-latest]
        version: [12,14]
    runs-on: ${{ matrix.os }}
    steps:
      - name: check matrix
        run: |
          echo ${{ matrix.os }}
          echo ${{ matrix.version }}
```
</details>

<details>
  <summary>If Condition</summary>
  
  ### ⭐ If Condition

- 특정 조건이 충족될 때 실행되도록 하는데 사용
- 개발 언어에서 사용하던 if 문과 비슷하게 제공되는 Operator를 사용하여 조건을 걸 수 있다
- Job Level, Step Level에서 실행 여부를 결정할 수 있다
    - workflow가 Trigger 된 이후 job과 Step 과정을 세밀하게 제어
- 특정 Job과 Step을 강제로 실행 가능한 `if:always()`

> Ex) If Condition
> 

```yaml
name: if-condition-workflow

on:
  push:
  workflow_dispatch:

jobs:
  job1:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
      - name: get event name
        run: echo ${{ github.event_name }}

  job2:
    runs-on: ubuntu-latest
    if: github.event_name != 'push'
    steps:
      - name: get event name
        run: echo ${{ github.event_name }}

  job3:
    runs-on: ubuntu-latest
    steps:
      - name: get PUSH
        if: github.event_name == 'push'
        run: echo "PUSH"
      - name: get Workflow Dispatch
        if: github.event_name != 'push'
        run: echo "Workflow Dispatch"
```
</details>

<details>
  <summary>String Functions</summary>
  
  ### StartsWith, EndsWith, Contains

- 문자열 처리 함수
- 문자열에 대한 조건 검사를 수행하여 Job 또는 Step 실행 여부를 결정한다
- startsWith(검사 대상, 찾으려는 값)
    - 검사 대상의 시작이 찾으려는 값이면 true 반환
- endsWith(검사 대상, 찾으려는 값)
    - 검사 대상의 끝이 찾으려는 값이면 true 반환
- contains(검사 대상, 찾으려는 값)
    - 검사 대상이 찾으려는 값을 포함하고 있으면 true 반환

> Ex) Strings.yaml
> 

```yaml
name: String-Fun-Workflow
on: push

jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - name: startsWith
        if: startsWith('ABC', 'A')
        run: echo "A"

      - name: endsWith
        if: endsWith('ABC', 'BC')
        run: echo "BC"

      - name: contains
        if: contains('ABC,A,BC', 'BC')
        run: echo "BC"
```
</details>
