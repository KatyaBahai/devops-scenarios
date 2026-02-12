## Запуск Jenkins с доступом к Docker
Чтобы контейнер Jenkins мог использовать Docker, необходимо примонтировать сокет Docker и задать соответствующие разрешения.

### Запуск Jenkins с доступом к Docker (Docker Desktop + WSL2)
В Docker Desktop уже есть Docker Engine внутри WSL2, поэтому нам не нужно вручную настраивать пользователя docker и UID/GID.

### Подготовка постоянного хранилища для Jenkins
Вместо /var/jenkins_home на Linux используем папку на Windows.

Пример (PowerShell или CMD):
```
mkdir C:\docker-data\jenkins
```
Docker Desktop сам корректно смонтирует эту папку в контейнер через WSL2, права на файлы выставлять вручную не нужно, как в linux OS. 

### Запуск контейнера Jenkins
```
docker run -d ^
  --name jenkins ^
  -p 9074:8080 ^
  -p 50000:50000 ^
  -v C:\docker-data\jenkins:/var/jenkins_home ^
  -v /var/run/docker.sock:/var/run/docker.sock ^
  jenkins/jenkins:lts-jdk17
```

- -v C:\docker-data\jenkins:/var/jenkins_home - постоянное хранилище Jenkins (на Windows)
- -v /var/run/docker.sock:/var/run/docker.sock - даёт Jenkins доступ к Docker Engine Docker Desktop
- 9074 — веб-интерфейс Jenkins
- 50000 — агенты

### Установка Docker CLI внутри контейнера Jenkins

Docker Engine уже есть в Docker Desktop, но внутри контейнера Jenkins нет Docker CLI, его нужно поставить.

- Зайти в контейнер:
```
docker exec -u root -it jenkins bash
```
- Установить Docker CLI:
```
apt-get update && apt-get install -y docker.io
```
Проверка:
```
docker --version
docker ps
```

3. Группа Docker (в Docker Desktop + WSL2 НЕ НУЖНО)
```
getent group docker
--group-add ...
```
Почему?
В Docker Desktop:
нет группы docker как на обычном Linux
доступ к сокету /var/run/docker.sock уже разрешён
Jenkins работает внутри той же среды, что и Docker Engine

4. Завершение установки Jenkins

Открыть в браузере:
http://localhost:9074

Получить пароль администратора:
```
docker logs jenkins

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:
2d7erwerewr2954746549ava8a50266rr69086747
```
Ввести пароль на странице установки, следовать настройкам по умолчанию, вписать администратора
