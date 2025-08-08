# 작업 중인 사항 임시 저장
### 충돌 상황에서 유용하게 쓰일 수 있는 명령어

- 작업 중인 변경 사항을 임시로 저장하고 나중에 다시 적용할 수 있게 함
- 이 명령어를 입력하면 작업했던 코드들이 잠시 사라지게 된다 (Working Directory의 변경 사항 저장)
- 임시 저장한 내용들은 Stack 구조로 쌓이게 된다 (가장 최근에 한 작업이 0번째 Index에 저장)
    
    ```bash
    // 임시 저장하기
    git stash
    
    // 임시 저장한 내용들 목록 확인
    git stash list
    
    // 상세 조회 방법
    git stash show -p [인덱스]
    
    // 임시 저장 내용 불러오기 (가장 최근 작업 내용을 list에서 제거하면서)
    // Wokring Directory에 적용
    git stash pop
    
    // 임시 저장 내용 불러오기 (list에서 제거하지 않은 채 Wokring Directory에 적용) 
    git stash apply
    
    // 특정 stash 항목 삭제
    git stash drop [인덱스]
    
    // 임시 저장 내용 전체 삭제
    git stash clear
    ```
