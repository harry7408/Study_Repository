# Github Auth
IntelliJ OR Android Studio에서 Get From VCS 또는 File → Settings → Version Control → Git를 통해 연결을 시도

### 1. OAuth (제 3자 인증)

- Github 계정에 로그인을 요청하고 이 정보를 바탕으로 토큰이 저장
- 매우 간편하지만 권한 제한 설정을 할 때는 한계가 있음
- 보안 관리가 덜 중요하거나 일반적으로 Public Repository 관리 할 때 사용

### 2. Personal Access Token (PAT)

- Github의 Settings → Developer Settings → personal access token 항목에서 토큰을 직접 발급 받아 사용
- 토큰을 발급 받은 후 직접 자격증명 및 KeyChain에 내용 추가
    - 주소 필드에 “git:https://github.com” 내용 추가
    - 사용자 이름 입력 및 암호에 토큰 값 추가
- 다양한 기능 및 권한 제한을 설정할 때는 좋으나 토큰 만료기간 및 값을 관리할 때 주의가 필요하다 (토큰 최초 생성 시 노출 되는 값을 분실 시 사용할 수 없음

### 자격 증명 현황 살펴보기

1. Window
    - 제어판 → 사용자 계정 → 자격 증명 관리자 → Windows 자격 증명
    - 일반 자격증명 항목에 Github 아이디와 토큰이 저장되어 있는 것을 볼 수 있음

1. Mac
    - Spotlight에 키체인 접근 검색
    - 상단 검색창에 Github 입력 시 계정과 암호가 들어있는 것을 볼 수 있음
