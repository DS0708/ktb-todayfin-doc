# 프론트앤드 CI/CD 환경 구성해주기

## 계획
- CI
    - CI/CD 실행 시킬 서버 및 ECR(Elastic Container Registry) 구성 (AWS)
    - Dockerfile 작성
    - Jenkins 서버 구축 (Docker 컨테이너로 실행)
    - Jenkins 파이프라인을 통해 빌드된 도커 이미지 ECR에 올리기
- CD
    - 미정, 일단 ECR에 이미지 올라가는 것 확인하고 진행할 예정