### 현재 내 Local PC(Working Directory) 및 Staging Area 추적 상황

```bash
git status
```

- IntelliJ & Android Studio 상에서 파일 추적 형태

| 색상 | 의미 |
| --- | --- |
| **녹색** | 새로 추가된 파일 (`git add` 됨) → **Staged 상태** |
| **청록색** | 수정된 파일 (`git add` 전) → **Modified 상태** |
| **파란색** | 수정되었고 `git add`도 된 상태 → **Staged + Modified** |
| **회색** | Git이 추적하지 않는 파일 → `.gitignore`에 포함되었거나 Untracked |
| **갈색/황토색** | 삭제된 파일 (`git rm`) 상태 |
| **보라색** | 이름이 변경된 파일 (`git mv`) 등 |

<br></br>
## Staging Area에 내용 추가

```bash
// 현재 디렉토리 및 하위에 있는 내용을 Staging Area에 추가 (삭제된 파일 제외)
git add .

// 현재 디렉토리 및 하위에 있는 내용을 Staging Area에 추가 (삭제된 파일 포함)
git add -u

// 작업 디렉토리 내에 있는 모든 파일을 Staging Area에 추가 (삭제된 파일 포함)
git add -A

// 특정 파일을 Staging Area에 추가
git add [파일 및 디렉터리 경로]
```
<br></br>
### Local Repository에 이력 추가 (Staging Area가 비워짐)

- Commit을 하게되면 본격적으로 이력이 생성된다
- Vim 편집기 상에서 메세지를 입력한다면 첫째 줄이 Header, 둘째 줄부터 Body 추가적인 내용은 Footer로 볼 수 있다.

```bash
git commit -m "Header(커밋 제목)" -m "Body(자세한 이력 남기기)" -m "Footer(별도 사항)"

Ex)
git commit -m "feat: 로그인 기능 추가" \
            -m "로그인 성공 시 홈 화면으로 이동하도록 처리함\n에러 메시지도 표시함" \
            -m "Closes #1"
            
-----------------------------------
feat: 로그인 기능 추가

로그인 성공 시 홈 화면으로 이동하도록 처리함
에러 메시지도 표시함

Closes #1
```

- Commit을 하게 되면 Commit ID가 생성된다 (매우 긴 난수 값이지만 일부만 입력해도 시스템이 알아서 인식)
<br></br>
### 원격 저장소로 코드 업로드 (Github OR GitLab etc..)

- 소스코드 변경사항 (Commit ID)을 업로드

```bash
// main branch에서 작업한 코드 Upload
git push origin main

// 다른 브랜치로 분기해서 작업한 경우
git push | git push [브랜치 명]
```
