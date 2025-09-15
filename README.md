# github-action

docker ps -a : 도커 프로세스 확인
docker stop {CONTAINER ID} : 도커 종료

docker cicd

yml파일을 통해서 main에 push가 됬을때 또는 어떤 이벤트가 발생했을때 어떤 작업을 하도록 코드를 통해서 지정할 수 있음
yml파일은 .gitignore를 통해 git에 올라가지 않도록 만듬

github repository와 프로젝트 연동 : ...or create a new repository on the command line 복사 후 인텔리제이 터미널에 붙여넣기.
Username for 'https://github.com' : {Username}
Password for 'https://{Username}@github.com' : {아래의 token}
Settings > Developer settings > Personal access tokens > Tokens(classic) > Generate new token > Generate new token(classic) > Note에 이름 정하고 만료일 30일, 권한은 전부 체크 > Generate token 

Settings > Secrets and variables > actions > Repository secrets > New repository secret
APPLICATION	: application.yml 파일
DOCKER_USERNAME	: 도커허브 닉네임
DOCKER_PASSWORD	: 도커허브 비밀번호 or accessKey
DOCKER_PROJECT	: 도커 프로젝트 이름
HOST_PROD	: EC2 퍼블릭 ip
PRIVATE_KEY	: EC2 접속용 pem Key

application.yml
# github repository actions 페이지에 나타날 이름
name: CI/CD using github actions & docker

# event trigger
# main이나 develop 브랜치에 push가 되었을 때 실행
on:
  push:
    branches: ["main"]

permissions:
  contents: read

jobs:
  CI-CD:
    runs-on: ubuntu-latest
    steps:

      # JDK setting - github actions에서 사용할 JDK 설정 (프로젝트나 AWS의 java 버전과 달라도 무방)
      - uses: actions/checkout@v3
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      # gradle caching - 빌드 시간 향상
      - name: Gradle Caching
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
            
			# application.yml 파일 생성후 파일 내용 넣는 과정
      - uses: actions/checkout@v3
      - run: touch ./src/main/resources/application.yml
      - run: echo "${{ secrets.APPLICATION }}" > ./src/main/resources/application.yml
      - run: cat ./src/main/resources/application.yml
   
      # gradle build
      - name: Build with Gradle
        run: ./gradlew build -x test

      # docker build & push
      - name: Docker build & push to prod
        if: contains(github.ref, 'main')
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | sudo docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          sudo docker build --platform linux/amd64/v3 -t "${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_PROJECT }}" .
          sudo docker push "${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_PROJECT }}"

      ## 실제 서버로 배포하는 작업
      - name: Deploy to prod
        uses: appleboy/ssh-action@master
        id: deploy-prod
        if: contains(github.ref, 'main')
        with:
          host: ${{ secrets.HOST_PROD }} # EC2 퍼블릭 IPv4 DNS
          username: ubuntu
          key: ${{ secrets.PRIVATE_KEY }}
          envs: GITHUB_SHA
          script: |
            sudo docker ps -a
            sudo docker stop $(sudo docker ps -q --filter "name=project")
            sudo docker rm $(sudo docker ps -aq --filter "name=project")
            sudo docker pull ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_PROJECT }}
            sudo docker run --name project -d -p 8080:8080 ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_PROJECT }}
            sudo docker image prune -f
