# Front-end CI/CD 환경 구성

## 계획
- CI
    - CI/CD 실행 시킬 서버 및 ECR(Elastic Container Registry) 구성 (AWS)
    - Dockerfile 작성
    - Jenkins 서버 구축
    - Github Webhook 세팅
    - Jenkins 파이프라인을 통해 빌드된 도커 이미지 ECR에 올리기
- CD
    - EC2 인스턴스 볼륨 늘리기 및 스왑 영역 설정
    - Jenkins pipeline 코드 구성
    - Pipeline script from SCM 으로 변경 


## CI - CI/CD 실행 시킬 서버 및 ECR(Elastic Container Registry) 구성 (AWS)
1. AWS EC2 인스턴스 구성 (by terraform)
    ```
    provider "aws" {
    region = "ap-northeast-2"
    }

    variable "instance_count" {
    default = 1
    }

    variable "instance_type" {
    default = "t3.medium"
    }

    variable "subnet_ids" {
    default = ["subnet-04d88cc60c57c4fb3", "subnet-0f93e87ed8fb436fd"]
    description = "List of subnet IDs to deploy EC2 instances"
    }

    # 새 보안 그룹 생성
    resource "aws_security_group" "jenkins_sg" {
    name        = "todayfin-fe-jenkins-server-sg"
    description = "Security group for Jenkins EC2 instances"
    vpc_id      = "vpc-0d363872e428237d5" 

    # 인바운드 규칙: SSH 접근 허용
    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    # 인바운드 규칙: Jenkins 웹 인터페이스 접근 허용
    ingress {
        from_port   = 8080
        to_port     = 8080
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    # 인바운드 규칙: 3000번 포트로 프론트앤드 서버 운영
    ingress {
        from_port   = 3000
        to_port     = 3000
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    # 아웃바운드 규칙
    egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
    }

    resource "aws_instance" "eks_ec2" {
    count                        = var.instance_count
    ami                          = "ami-062cf18d655c0b1e8"  # 지역에 맞는 AMI 입력
    instance_type                = var.instance_type
    key_name                     = "covyEc2"  # AWS 계정에 이미 생성된 키 쌍 이름
    subnet_id                    = element(var.subnet_ids, count.index % length(var.subnet_ids))
    vpc_security_group_ids       = [aws_security_group.jenkins_sg.id]  # 생성된 보안 그룹 사용

    tags = {
        Name = "todayfin-fe-jenkins-server"
    }
    }

    output "instance_ips" {
    value = aws_instance.eks_ec2.*.public_ip
    }

    ```

2. 프론트앤드 도커 이미지가 올라갈 ECR 생성
    - registry name : todayfin-fe

## CI - Dockerfile 작성
- feat/ci 브랜치 파서 진행
- [도커파일 작성](https://github.com/vercel/next.js/tree/canary/examples/with-docker)
    ```
    FROM node:18-alpine AS base

    # Install dependencies only when needed
    FROM base AS deps
    # Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
    RUN apk add --no-cache libc6-compat
    WORKDIR /app

    # Install application dependencies using npm
    COPY package.json package-lock.json* ./
    RUN npm ci
    # Ensure sharp is installed. You may need to add specific platform tools if sharp installation fails due to binary compatibility.
    RUN npm install sharp



    # Rebuild the source code only when needed
    FROM base AS builder
    WORKDIR /app
    COPY --from=deps /app/node_modules ./node_modules
    COPY . .

    # Next.js collects completely anonymous telemetry data about general usage.
    # Learn more here: https://nextjs.org/telemetry
    # Uncomment the following line in case you want to disable telemetry during the build.
    # ENV NEXT_TELEMETRY_DISABLED 1

    RUN npm run build


    # Production image, copy all the files and run next
    FROM base AS runner
    WORKDIR /app

    ENV NODE_ENV production
    # Uncomment the following line in case you want to disable telemetry during runtime.
    # ENV NEXT_TELEMETRY_DISABLED 1

    RUN addgroup --system --gid 1001 nodejs
    RUN adduser --system --uid 1001 nextjs

    COPY --from=builder /app/public ./public

    # Set the correct permission for prerender cache
    RUN mkdir .next
    RUN chown nextjs:nodejs .next

    # Automatically leverage output traces to reduce image size
    # https://nextjs.org/docs/advanced-features/output-file-tracing
    COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
    COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

    USER nextjs

    EXPOSE 3000

    ENV PORT=3000

    # server.js is created by next build from the standalone output
    # https://nextjs.org/docs/pages/api-reference/next-config-js/output
    CMD HOSTNAME="0.0.0.0" node server.js
    ```
    - sharp 관련 에러 떠서 `npm install sharp` 추가
    - npm에 맞게 조건문 삭제
- .dockerignore 작성
    ```
    Dockerfile
    .dockerignore
    node_modules
    npm-debug.log
    README.md
    .next
    !.next/static
    !.next/standalone
    .git
    ```
- next.config.mjs에 컨테이너 형태의 배포를 위한 `output: "standalone` 추가
    ```
    /** @type {import('next').NextConfig} */
    const nextConfig = {
        output: "standalone",
    };

    export default nextConfig;
    ``` 
- feat/ci 브랜치에 push
- PR 올리고 코드 리뷰 받기
- Approve 받고 dev에 merge



## CI - Jenkins 서버 구축
EC2 인스턴스에 도커 컨테이너로 젠킨스를 실행하려고 했지만, docker 소켓 이슈, 도커 컨테이너 내부에서 systemctl이 안되는 이유로 그냥 ubuntu위에 바로 깔아서 진행하기로 했다.

- Jenkins, java, docker 설치 (공식 문서 참고)
    ```
    sudo su

    wget -O /usr/share/keyrings/jenkins-keyring.asc   https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]"   https://pkg.jenkins.io/debian-stable binary/ | sudo tee   /etc/apt/sources.list.d/jenkins.list > /dev/null

    apt-get update
    apt-get install jenkins

    apt update
    apt install fontconfig openjdk-17-jre

    apt install docker.io -y
    chown jenkins:jenkins /var/run/docker.sock

    systemctl enable jenkins
    systemctl start jenkins
    systemctl status jenkins
    ```

- Jenkins 최초 접속 후 설정
    - public-ip:8080 접속
    - Password 입력, 다음 명령어로 확인 가능
        ```
        sudo cat /var/lib/jenkins/secrets/initialAdminPassword
        ```
    - Install suggested plugins 선택
    - Jenkins 관리 - Plugins - Available plugins에서 `GitHub Integration, Docker Pipeline, Amazon ECR, AWS Credentials` 이렇게 4개의 플러그인 설치 후 Jenkins 재시작 체크


## CI - Github Webhook 세팅
- 레포 Settings - Webhooks - Add webhook (Github에서 진행)
    - Payload URL: http://<JENKINS_SERVER_IP>:8080/github-webhook/
    - Content type : application/json
    - Secret: 빈칸
    - Just the push event 선택
- Generate Token - ghp_XXX 토큰 복사 -> Jenkins 대시보드 - Jenkins 관리 - System - Github Server 검색 - + Add - Credential 생성 - Secret text의 Secret에 ghp_XXX 토큰 붙여넣기
- Manage hooks 체크 후 저장


## CI - Jenkins Credential 세팅 (ECR 접근을 위한 AWS IAM Access key 등록)
- Jenkins 관리 - Credentials - (global)클릭 + add credential선택 - Kind : AWS Credentials 선택
- ID : ecr_credentials_id
- 관리자 권한 가진 사용자의 액세스키 발급후, key id, secret key 복붙

## CI - CI Pipeline 생성
- Jenkins 메인페이지에서 좌측 새로운 Item 클릭 - Pipeline 선택 후 OK - GitHub project - Project url : github.com/user-id/repo-name
- GitHub hook trigger for GITScm polling -> 선택을 무조건 해줘야한다.
- Jenkins CI Pipeline Script
    ```
    pipeline {
        agent any

        environment {
            REPO = 'KTB-LuckyVicky/todayfin-fe'
            ECR_REPO = '905418374604.dkr.ecr.ap-northeast-2.amazonaws.com/todayfin-fe'
            ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:ecr_credentials_id'
        }

        stages {
            stage('Checkout') {
                steps {
                    git branch: 'feat/cd', url: "https://github.com/${REPO}.git"
                }
            }

            stage('Build Docker Image') {
                steps {
                    script {
                        dockerImage = docker.build("${ECR_REPO}:latest")
                    }
                }
            }

            stage('Push to ECR') {
                steps {
                    script {
                        docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                            dockerImage.push('latest')
                        }
                    }
                }
            }
        }
    }
    ```

- `지금 빌드` 클릭하고 ECR에 잘 올라가는지 확인
- 확인 후 해당 이미지 내려 받아서 Local에서 실행해보기 (awscli랑 IAM있어야 가능)
    - Docker 클라이언트를 ECR에 로그인하려면, 먼저 ECR 로그인 커맨드를 생성해야 한다. AWS CLI를 사용하여 필요한 도커 로그인 커맨드를 얻는다.
        ```
        aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin [계정ID].dkr.ecr.[리전].amazonaws.com  
        ```
    - 이미지 Pull
        ```
        docker pull [계정ID].dkr.ecr.[리전].amazonaws.com/[리포지토리 이름]:[태그]
        ```
    - pull되었는 지 확인
        ```
        docker images
        ```
    - docker run 실행후 localhost 접속해보기
        ```
        docker run -itd -p 3000:3000 [이미지 이름]
        ```


## CD - EC2 인스턴스 볼륨 늘리기 및 스왑 영역 설정
서버 비용때문에 일단은 t3.mideum에 젠킨스와 프론트앤드를 동시에 실행시켜보기로 했다. 위와 같은 terrafrom 코드로 배포 시 EBS 볼륨이 8gib라 볼륨을 먼저 늘리고 스왑 영역을 설정하려고 한다. 그리고 jenkins는 peak시 1.1GB, docker는 peak시 2.1GB의 메모리를 사용하고 t3.mideum은 4GB여서 OOM을 방지하기 위해 스왑 영역도 설정하려고 한다.

- AWS Ec2의 EBS 볼륨에 들어가서 수정하기 눌러서 볼륨 늘리기 -> 8gib -> 30gib
- 이제 파일 시스템이 늘어난 볼륨을 인식하도록 해야 한다. 먼저 명령어로 볼륨 정보 확인
    ```
    df -h
    lsblk
    ```
- 파티션 확장
    ```
    sudo growpart /dev/nvme0n1 1
    ```
- 파일 시스템 확장
    ```
    sudo resize2fs /dev/nvme0n1p1
    ```

> 이제 파일 시스템이 늘어난 볼륨을 인식하도록 했으므로 스왑 영역을 설정해야 한다.

- 스왑 파일 생성
    ```
    sudo fallocate -l 4G /swapfile
    ```
- 파일 권한 설정
    ```
    chmod 600 /swapfile
    ```
- 생성된 파일을 스왑 영역으로 설정
    ```
    sudo mkswap /swapfile
    ```
- 스왑 활성화
    ```
    sudo swapon /swapfile
    ```
- 부팅 시 스왑 활성화 설정
    ```
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
    ```
- 설정 확인
    ```
    free -h
    ```

## CD - Jenkins pipeline 코드 구성
기존의 CI Pipeline에서 ECR에 도커 이미지가 Push되면 해당 이미지를 내려 받아 docker run 하는 stage를 추가한다.
```
pipeline {
    agent any

    environment {
        REPO = 'KTB-LuckyVicky/todayfin-fe'
        ECR_REPO = '905418374604.dkr.ecr.ap-northeast-2.amazonaws.com/todayfin-fe'
        ECR_CREDENTIALS_ID = 'ecr:ap-northeast-2:ecr_credentials_id'
        ALPHA_VANTAGE_API_KEY = credentials('alpha_vantage_api_key')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'feat/cd', url: "https://github.com/${REPO}.git"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${ECR_REPO}:latest")
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy on Local Server') {
            steps {
                script {
                    // Jenkins 자격증명을 사용하여 Docker 로그인
                    docker.withRegistry("https://${ECR_REPO}", "${ECR_CREDENTIALS_ID}") {
                        //기존에 도커 컨테이너 삭제
                        sh "docker rm -f todayfin-fe"

                        // ECR에서 이미지 pull
                        sh "docker pull ${ECR_REPO}:latest"

                        // 도커 컨테이너 실행
                        sh "docker run -d -e ALPHA_VANTAGE_API_KEY=${ALPHA_VANTAGE_API_KEY} --name todayfin-fe -p 3000:3000 ${ECR_REPO}:latest"
                    }
                }
            }
        }
    }
}
```
- Front-end에서 .env파일에서 apikey를 사용하는 부분이 있는데, .env파일은 깃허브에 올라갈 수 없으므로 Jenkins의 Credentials에 해당 ALPHA_VANTAGE_API_KEY를 등록하여 사용하였다.
- 기존에 도커가 3000번 포트로 실행 시 충돌을 일으키기 때문에, 기존의 도커 컨테이너 삭제 후 이미지 Pull, 컨테이너 실행을 진행하였다.


## CD - Pipeline script from SCM 으로 변경 
Jenkinsfile을 프로젝트의 루트 디렉토리에 생성해야한다.

- Definition을 Pipeline script from SCM으로 변경
- SCM을 Git으로 설정
- Repository URL에 git clone시 사용 되는 Repo URL 설정
- Credentials은 필요 없었다.
- Branches to build 설정, 일단 */feat/cd로 지정 하였다. 아마 여기서 지정한 브랜치에서 Jenkinsfile을 찾을 것이다
- 그 후 feat/cd 브랜치에 Jenkinfile 생성 후 push, PR, merge 진행



## 나중에 더 진행해야할 것
- 현재 EC2인스턴스 하나에서 젠킨스 서버와 프론트앤드 서버가 돌아간다. 문제가 생기면 이것을 분리해야 한다.
- 도커 이미지 빌드 시 시간이 오래 걸린다. 약 3분 정도
- 지금 테스트 용으로 트리거 브랜치를 feat/cd로 설정하였다. 일단 dev로 바꾼 후에 Front-end개발자가 main 브랜치로 merge하면 그때 트리거 브랜치랑 Jenkinsfile찾는 브랜치를 main으로 변경해야 한다.
- 무중단 배포 -> 현재 배포 시 잠깐 동안 도커 컨테이너가 종료된다. LB나 nginx를 이용하여 블루/그린 무중단 배포 진행하기.
- 특정 브랜치에만 trigger 되도록 설정하기