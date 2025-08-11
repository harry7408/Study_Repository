# Github Actions (CI/CD) 개요
## Why?

- Continuous Integration + Continuous Deployment로 개발의 효율성을 높일 수 있어 현업에서 자주 사용한다
    
    → 즉 지속적으로 통합과 배포를 자동적으로 수행하여 개발자의 수고를 덜어줄 수 있다
    

### Github Actions란?

- Github에서 제공하는 자동화 도구
- 단순한 Workflow부터 복잡한 Workflow 까지 구축 가능하다
- Software 개발 Workflow를 자동화 하는데 사용된다

특징

1. 개발, 테스트, 배포 등의 Workflow를 Github 저장소 내에서 직접 관리한다
2. Github 저장소와 완전히 통합되어 추가적인 인프라 설정이나 외부 도구를 관리할 필요가 없다
3. .yml, .yaml 파일을 사용하여 Workflow를 쉽게 구성할 수 있다
    
    > 💡 yml, yaml : 데이터 구조를 간단하게 표현한 직렬화 언어
    > 
    
    <img width="1000" height="487" alt="Image" src="https://github.com/user-attachments/assets/6642dfa8-9f4d-4929-a9bb-ddc908abf1cc" />
    
4. 다양한 OS에서 지원한다
    
    > Linux, MacOS, Windows 모두 가능
    > 
5. 다양한 이벤트에 따라 실행하는 Workflow
    
    → 깃허브의 push, issue 생성 등 깃허브 이벤트 기반으로 구성된다
