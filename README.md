1. Repository 생성
2. Ec2 접속 -> docker 설치
```
	1. sudo yum update -y
	2. sudo yum install -y docker
	3. sudo systemctl start docker
	4. sudo systemctl enable docker
	5. sudo usermod -aG docker ec2-user
```
4. Ec2 종료 후 다시 접속
	1. exit
	2. Ssh or console로 재접속
	3. Docker 버전 확인 docker --version
5. SpringBoot Docker로 띄우기
	1. Git repo clone받아서 안에 프로젝트 넣기.
	2. Dockerfile 작성
4. GitHub actions에서 쓸 ssh key 생성
	1. ssh key
    ```Ssh-keygen -t rsa -b 4096 -C “ci-cd” ```
=> id_rsa / id_rsa.pub 생성됨 
=> id_rsa는 깃헙 레포 시크릿에 저장, id_rsa.pub은 EC2에 등록
	3. ec2에서 mkdir -p ~/.ssh
	4. cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
	5. chmod 600 ~/.ssh/authorized_keys
5. GitHub actions secret 등록
	1. EC2_HOST : EC2 public ip
	2. EC2_PRIVATE_KEY : id_rsa 내용 : ```cat ~/.ssh/id_rsa```
6. .github/workflows에 deploy.yml 등록
```
name: Deploy to EC2

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v3

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Build with Gradle (skip tests)
        run: ./gradlew clean build -x test

      - name: Upload JAR & Dockerfile to EC2
        uses: appleboy/scp-action@v0.1.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          source: "build/libs/*.jar,Dockerfile"
          target: "~/app"
          flatten: true  

      - name: SSH into EC2 and deploy
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          script: |
            cd ~/app
            docker stop spring-app || true
            docker rm spring-app || true
            docker build -t spring-app .
            docker run -d -p 8080:8080 --name spring-app spring-app
```
——
action 등록 완료
——
——
Blue green 으로 바꾸기
——
1. Nginx 설지
	sudo yum install nginx -y
	sudo systemctl enable nginx
	sudo systemctl start nginx
2. Blue green config
	1. sudo mkdir -p /etc/nginx/conf.d/bluegreen
	2. Blue 설정
    
```
cat <<EOF | sudo tee /etc/nginx/conf.d/bluegreen/service-blue.conf
upstream spring-app {
    server 127.0.0.1:8081;  # blue
}

server {
    listen 80;

    location / {
        proxy_pass http://spring-app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

  3. Green 설정

 ```
cat <<EOF | sudo tee /etc/nginx/conf.d/bluegreen/service-green.conf
upstream spring-app {
    server 127.0.0.1:8082;  # green
}

server {
    listen 80;

    location / {
        proxy_pass http://spring-app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
```

  4. 현재 사용할 설정 심볼릭 링크 연결
		sudo ln -sf /etc/nginx/conf.d/bluegreen/service-blue.conf /etc/nginx/conf.d/service.conf
	5. Nginx 설정 테스트 및 reload
		sudo nginx -t && sudo nginx -s reload
3. deploy.yml 파일 수정

```
name: Blue-Green Deploy to EC2

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source
        uses: actions/checkout@v3

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Build with Gradle (skip tests)
        run: ./gradlew clean build -x test

      - name: Upload JAR & Dockerfile to EC2
        uses: appleboy/scp-action@v0.1.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          source: "build/libs/*.jar,Dockerfile"
          target: "~/app"
          flatten: true

      - name: SSH into EC2 and deploy Blue-Green
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          script: |
            cd ~/app
            CURRENT_PORT=$(readlink /etc/nginx/conf.d/service.conf | grep -oE '808[1-2]')
            if [ "$CURRENT_PORT" == "8081" ]; then
              NEW_PORT=8082
              NEW_VERSION="green"
            else
              NEW_PORT=8081
              NEW_VERSION="blue"
            fi
            docker stop spring-$NEW_VERSION || true
            docker rm spring-$NEW_VERSION || true
            docker build -t spring-app .
            docker run -d -p $NEW_PORT:8080 --name spring-$NEW_VERSION spring-app
            sleep 10
            curl -f http://localhost:$NEW_PORT || (echo "Health check failed" && exit 1)
            sudo ln -sf /etc/nginx/conf.d/bluegreen/service-$NEW_VERSION.conf /etc/nginx/conf.d/service.conf
            sudo nginx -t && sudo nginx -s reload
            if [ "$NEW_VERSION" == "green" ]; then
              docker stop spring-blue || true && docker rm spring-blue || true
            else
              docker stop spring-green || true && docker rm spring-green || true
            fi

```
