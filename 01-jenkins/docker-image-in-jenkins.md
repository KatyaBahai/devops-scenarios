# Создание образа приложения в Jenkins pipeline.

Если мы проанализируем Jenkinsfile, то увидим, что он уже выполняет часть этапов, описанных в Docker.
По сути, нам нужно сразу скопировать JAR и собрать образ только для запуска приложения. 
## Давайте исправим Dockerfile и Jenkinsfile.

Dockerfile:
```
FROM openjdk:21-jdk as builder
​
COPY . .

RUN jar xf /build/libs/DevOps-1.0.0.jar

RUN jdeps --ignore-missing-deps -q \
    --recursive \
    --multi-release 21 \
    --print-module-deps \
    --class-path 'BOOT-INF/lib/*' \
    /build/libs/DevOps-1.0.0.jar > deps.info
​
RUN jlink \
    --add-modules $(cat deps.info) \
    --strip-debug \
    --compress 2 \
    --no-header-files \
    --no-man-pages \
    --output /slim-jre
​

FROM debian:bookworm-slim
ENV JAVA_HOME /user/java/jdk21
ENV PATH $JAVA_HOME/bin:$PATH
COPY --from=builder /slim-jre $JAVA_HOME
COPY --from=builder /build/libs/DevOps-1.0.0.jar .
ENTRYPOINT ["java", "-jar", "DevOps-1.0.0.jar"]
```
​
Jenkinsfile:
```
pipeline {
    agent { label 'agent-jdk21' }​

    tools {
        git 'Default'
    }
​
    stages {
        stage('Prepare Environment') {
            steps {
                sh 'chmod +x ./gradlew'
            }
        }
        stage('Check') {
            steps {
                sh './gradlew check'
            }
        }
        stage('Package') {
            steps {
                sh './gradlew build'
            }
        }
        stage('JaCoCo Report') {
            steps {
                sh './gradlew jacocoTestReport'
            }
        }
        stage('JaCoCo Verification') {
            steps {
                sh './gradlew jacocoTestCoverageVerification'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t job4j_devops .'
            }
        }
    }
​
    post 
        always {
            script {
                def buildInfo = """
                    Build number: ${currentBuild.number}
                    Build status: ${currentBuild.currentResult}
                    Started at: ${new Date(currentBuild.startTimeInMillis)}
                    Duration: ${currentBuild.durationString}
                """
                telegramSend(message: buildInfo)
            }
        }
    }
}
```
​
### Теперь мы настроим нового агента для работы с Docker.

Сначала запустите контейнер:
```
docker run -d --name=agent-jdk21 \
-p 2221:22 \
--group-add $(getent group docker | cut -d: -f3) \
-v /var/agent-jdk21:/var/agent-jdk21 \
-v /var/run/docker.sock:/var/run/docker.sock \
-e "JENKINS_AGENT_SSH_PUBKEY=$[SSH]" \
jenkins/ssh-agent:latest-jdk21
``` 
​
Замените SSH на ключ от контейнера Jenkins. Подробнее: [здесь](https://github.com/KatyaBahai/devops-scenarios/blob/main#%D0%BF%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%BD%D1%8B%D0%B9-%D0%B0%D0%B3%D0%B5%D0%BD%D1%82-jenkins-%D1%87%D0%B5%D1%80%D0%B5%D0%B7-ssh).

На хост-машине выполните команду:
```
getent group docker
```​
Результат:
```
docker:x:999:
``` ​
GROUP_ID = 999 — идентификатор группы, имеющей доступ к демону Docker.
``` 
Зайдите в образ Jenkins:
```
docker exec -u root -it agent-jdk21 bash
```
​
Установите Docker CLI:

```
apt-get update && apt-get install -y docker.io
```
​
Проверьте доступ:

```
getent group docker
``` 
​
Если вывод пуст, добавьте разрешения:

```
groupadd -g <GROUP_ID_FROM_HOST> docker
usermod -aG docker jenkins
```
​
Проверьте пользователя:
```
id jenkins
```
​
Результат:
```
uid=0(jenkins) gid=0(jenkins) groups=0(jenkins),999(docker)
```
​
Выйдите из контейнера:
```
exit
```
​
И на хост-машине перезапустите образ:

```
docker restart agent-jdk21
```
