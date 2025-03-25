# **Домашнее задание к занятию «Конфигурация приложений»**-***Вуколов Евгений***
 
### Цель задания
 
В тестовой среде Kubernetes необходимо создать конфигурацию и продемонстрировать работу приложения.
 
------
 
### Чеклист готовности к домашнему заданию
 
1. Установленное K8s-решение (например, MicroK8s).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым GitHub-репозиторием.
 
------
 
### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания
 
1. [Описание](https://kubernetes.io/docs/concepts/configuration/secret/) Secret.
2. [Описание](https://kubernetes.io/docs/concepts/configuration/configmap/) ConfigMap.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.
 
------
 
### Задание 1. Создать Deployment приложения и решить возникшую проблему с помощью ConfigMap. Добавить веб-страницу
 
1. Создать Deployment приложения, состоящего из контейнеров nginx и multitool.
2. Решить возникшую проблему с помощью ConfigMap.
3. Продемонстрировать, что pod стартовал и оба конейнера работают.
4. Сделать простую веб-страницу и подключить её к Nginx с помощью ConfigMap. Подключить Service и показать вывод curl или в браузере.
5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.
 
------
 
### Задание 2. Создать приложение с вашей веб-страницей, доступной по HTTPS 
 
1. Создать Deployment приложения, состоящего из Nginx.
2. Создать собственную веб-страницу и подключить её как ConfigMap к приложению.
3. Выпустить самоподписной сертификат SSL. Создать Secret для использования сертификата.
4. Создать Ingress и необходимый Service, подключить к нему SSL в вид. Продемонстировать доступ к приложению по HTTPS. 
5. Предоставить манифесты, а также скриншоты или вывод необходимых команд.
 
------
 
### Правила приёма работы
 
1. Домашняя работа оформляется в своём GitHub-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.


# **Решение**

## **Задание 1**

- Для первого задания слъоздаю namespace netology-config.

1. Создал Deployment с двумя контейнерами, nginx и multitool:

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool-deployment
  namespace: netology-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
```

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20134054.png)

 После запуска Deployment, возникла ошибка, nginx запустился, multitool не запустился. Причина этого, конфликт портов, так как внутри контейнера multitool есть встроенный nginx, 
который также пытается использовать порт 80.

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20134154.png)

- Эту проблему решаю с помощью файла configmap.yaml: 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: netology-config
data:
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20135417.png)

- Затем обновляю файл nginx-multitool.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool-deployment
  namespace: netology-config
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
        - name: nginx-html-volume
          mountPath: /usr/share/nginx/html
      - name: multitool
        image: wbitt/network-multitool:latest
        command: ["sh", "-c", "while true; do sleep 3600; done"]
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
      - name: nginx-html-volume
        configMap:
          name: nginx-html
```
- Команда command: ["sh", "-c", "while true; do sleep 3600; done"] исправляет ошибку которая возникала при первоначальном запуске ещё не изменённого кода, эта команда запускает бесконечный цикл в shell.
Так как образ wbitt/network-multitool по умолчанию пытается запустить nginx, который конфликтует с основным контейнером nginx из-за порта 80. 
После неудачного запуска контейнер завершается, и Kubernetes перезапускает его снова и возникает "CrashLoopBackOff".
Эта команда переопределяет стандартное поведение образа, запрещая ему запускать nginx.

- После добавления изменений, в pod запустилось 2 котейнера: 
 
- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20152351.png)
- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20152528.png)

4. Делаю простую веб-страницу и подключаю её к Nginx с помощью ConfigMap, а также создаю и подключаю файл Service и вывожу командой curl результат: 

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20163141.png)

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20164718.png)


- Ссылки манифеста: 

[nginx-multitool.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/nginx-multitool.yaml)

[html-configmap.yam](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/html-configmap.yaml)

[configmap.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/configmap.yaml)

[service-nginx-multitool.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/service-nginx-multitool.yaml)


## **Задание 2**

- Для этого задания создаю namespace netology-https:

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20171715.png)

1. Создаю Deployment приложения, состоящего из Nginx:

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20173510.png)

2. Создаю собственную веб-страницу и подключаю её как ConfigMap к приложению.

3. Делаю самоподписной сертификат SSL. Создаю Secret для использования сертификата:

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20173835.png)
- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20192629.png)
- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20192913.png)

4. Создаю Ingress и необходимый Service, подключаю к нему SSL в вид.И демонстрирую доступ к приложению по HTTPS:

- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20174804.png)
- ![scrin](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/Снимок%20экрана%202025-03-24%20174908.png)


- Ссылки манифеста: 

[nginx-config.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/nginx-config.yaml)

[nginx-https.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/nginx-https.yaml)

[nginx-ingress-https.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/nginx-ingress-https.yaml)

[nginx-service-https.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/nginx-service-https.yaml)

[nginx-tls-secret.yaml](https://github.com/Evgenii-379/2.3-2.3.md/blob/main/config.yaml/nginx-tls-secret.yaml)























