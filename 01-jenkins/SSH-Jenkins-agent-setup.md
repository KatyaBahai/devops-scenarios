### Постоянный агент Jenkins через SSH

Агент (где будет выполняться работа, worker) будет запущен в контейнере Docker с настроенной JDK 21.

- Генерация ключей SSH
На мастер-машине с Jenkins нужно создать пару ключей: private и public.

Подключитесь к контейнеру Jenkins с помощью команды:
```
docker exec -it jenkins bash
```
​
Внутри контейнера выполните генерацию ключей:
```
ssh-keygen -t rsa -b 4096 -C "devops@job4j.ru"
```
​
Нажимайте Enter на всех этапах для использования настроек по умолчанию.

После завершения генерации вы получите два ключа:
```
ls ~/.ssh/

id_rsa id_rsa.pub
```
​
- Настройка доступа в Jenkins
1) Откройте настройки Jenkins в UI и перейдите в раздел Credentials.
2) Выберите домен Jenkins и добавьте новую запись.
3) Укажите тип подключения: SSH Key.
4) Укажите имя пользователя (например, jenkins) и вставьте содержимое файла id_rsa в поле Private Key:
```
cat ~/.ssh/id_rsa 
```
​
Скопируйте текст из терминала и вставьте в соответствующее поле.
5) Сохраните изменения.

- Создание контейнера для агента

Чтобы выйти из контейнера Jenkins, введите команду:

```
exit
```
  ​
Проверьте текущего пользователя:
``` 
whoami
``` 
​Имя должно быть именем сервера с jenkins.

Запустите контейнер с предустановленной JDK и клиентом Jenkins SSH:
```
docker run --name=agent1 \
-e "JENKINS_AGENT_SSH_PUBKEY=[id_rsa.pub]" \
jenkins/ssh-agent:alpine-jdk17
```
-e — задаёт переменные среды. Укажите содержимое файла /home/jenkins/.ssh/id_rsa.pub, удалив квадратные скобки.

Пример:

```
docker run -d --name=agent1  \
-e "JENKINS_AGENT_SSH_PUBKEY=ssh-rsa AAAAB3Nza-------------mSZM+Vf7cUfRTw== devops@job4j.ru" \
jenkins/ssh-agent:alpine-jdk17
```
Проверим работу через ssh.

Получите адрес контейнера agent1:
```
docker inspect agent1 | grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.3",
                    "IPAddress": "172.17.0.3",
```                     
Далее зайдите в контейнер jenkins:
```
docker exec -it jenkins bash
```
Подключитесь в нем через ssh к контейнеру agent1:
```
ssh 172.17.0.3
The authenticity of host '172.17.0.3 (172.17.0.3)' can't be established.
ED25519 key fingerprint is SHA256:clcZuWVsC4SsY0WgWHiHiKfjfeHJpC9f9G8PCG7chyw.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.3' (ED25519) to the list of known hosts.
Welcome to Alpine!
```

- Настройка узла в Jenkins
1) Откройте настройки Jenkins и перейдите в раздел Nodes.
2) Добавьте новый узел.
3) Укажите параметры:
Имя: agent1. Удалённая директория: /home/jenkins. Способ запуска: SSH.
4) Укажите IP-адрес контейнера Agent1.
5) Выберите тип авторизации Jenkins и отключите проверку ключа (опция Не проверять).
6) Нажмите Save. После сохранения агент начнёт создаваться.
7) Откройте вкладку Log и убедитесь, что в логе нет ошибок.

Если всё корректно, в списке Nodes появится ваш узел с информацией о контейнере.
