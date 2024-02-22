# 로컬 jenkins에서 build후 ec2로 배포

aws에서 빌드해도 되지만 aws는 비싸서 비용절감을 위해 로컬에서 build 후 ec2에 배포한다.

## 1. jenkins 설치

### 환경 구현

로컬 가상머신 위에서 build를 위해 docker와 jenkins를 설치 구동한다.

```
$ sudo apt-get update
$ sudo apt install docker.io -y
$ docker run --name myjenkins --privileged -p 8080:8080 jenkins/jenkins:lts-jdk17
```

## 2. jenkins 설정

### 로그인

jenkins 비밀번호 확인

```
$ docker exec myjenkins sh -c 'cat /var/jenkins_home/secrets/initialAdminPassword' 
```

http://127.0.0.1:8080으로 접속

### 파이프라인 구성

GitHub project 체크, Repository url입력

GitHub hook trigger for GITScm polling 체크

```shell
pipeline {
    agent any

    stages {
        stage('git pull') {
            steps {
                echo 'pull start'
                git branch: '----', credentialsId: '----', url: '----'
                echo 'pull end'
            }
        }
        stage('build') {
            steps {
                echo 'build start'
                sh './gradlew build'
            }
        }
        stage('push') {
            steps {
                echo "push success"
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'myec2_05', transfers: [
                                sshTransfer(
                                    cleanRemote: false, excludes: '',
                                    execCommand: 'echo "push success"',
                                    execTimeout: 0,
                                    flatten: false,
                                    makeEmptyDirs: false,
                                    noDefaultExcludes: false,
                                    patternSeparator: '[, ]+',
                                    remoteDirectory: 'test/',
                                    remoteDirectorySDF: false,
                                    removePrefix: 'build/libs',
                                    sourceFiles: 'build/libs/*SNAPSHOT.jar'
                                )
                            ],
                            usePromotionTimestamp: false,
                            useWorkspaceInPromotion: false,
                            verbose: false
                        )
                    ]
                )
            }
        }
    }
}

```

#### pipeline syntax - sshPublisher

**SSH Server**

1. Name: 사용할 이름 아무거나
2. Transfers
   1. Source files: 생성된 jarfile의 위치 입력 [build/libs/*SNAPSHOT.jar]
   2. Remove prefix: jarfile의 파일위치 입력 [build/libs]
   3. Remote directory: ec2에 jar파일이 올라갈 위치입력 [test/]
   4. Exec command: connection 후 실행될 명령어 [echo "push success"]

## 3. github 설정

### Ngrok 연결

로컬 jenkins를 github가 인식할 수 있게 ngrok을 이용해 ip를 연결

```
$ ngrok http http://localhost:8080
```

### webhook 생성

Settings -> Webhooks -> add webhook

Payload URL: http://<ngrok에 나오는 주소>/github-webhook/

Content type: application/json
