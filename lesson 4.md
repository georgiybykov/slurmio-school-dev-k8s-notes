---
date created: 2021-10-12 19:01:26 (+03:00), Tuesday
---

# Урок 4: Хранение конфигураций. Вечерняя школа «Kubernetes для разработчиков» [youtube](https://www.youtube.com/watch?v=-xZ02dEF6kU)


## Вступление ([00:01:45](https://youtu.be/-xZ02dEF6kU?t=105))
- Каким образом в кластере k8s можно передавать конфигурации в наши приложения
    - Самый простой и неправильный вариант - захардкодить конфигурацию в контейнер и запускать приложение в таком неизменном виде. Не нужно так делать!
    - Более цивилизованные варианты будут представлены далее

## Представление преподавателя ([00:02:31](https://youtu.be/-xZ02dEF6kU?t=151))
- Сергей Бондарев
- Архитектор Southbridge
- Инженер с 25-летним стажем
- Certified Kubernetes Administrator
- Внедрения Kubernetes: все куб-проекты Southbridge, включая собственную инфраструктуру
- Один из разработчиков kubespray с правами на принятие pull request

## Env, переменные окружения ([00:03:49](https://youtu.be/-xZ02dEF6kU?t=228))
- Работаем в каталоге `~/school-dev-k8s/practice/4.saving-configurations/1.env`
- Открываем манифест `deployment-with-env.yaml`
```yaml
# deployment-with-env.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: quay.io/testing-farm/nginx:1.12
        name: nginx
        env:            # Описываем желаемые переменные окружения
        - name: TEST    # Имя переменной окружения
          value: foo    # Значение переменной окружения
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
...
```
- В этом манифесте мы указываем вариант доставки конфигураций внутрь нашего приложения так как это советует 12-factor applications, то есть, доставляем конфигурацию в виде переменных окружения
- У нас среда исполнения (docker, containerD, crio) позволяет при запуске процесса внутри контейнера создать этому процессу переменные окружения
- Самый простой вариант в kubernetes - описать все наши переменные окружения, который должны быть в контейнере, в манифесте (пода, репликасета, но как правило, это деплоймент)
- Смотрим в манифест, комментариями отмечен раздел с переменными окружения в описании контейнера
- Когда мы применим такой манифест в нашем кластере, в соответствующем контейнере появится переменная **TEST** со значением **foo**:
```shell
$ kubectl apply -f deployment-with-env.yaml

deployment.apps/my-deployment created

$ kubectl get pod

NAME                             READY   STATUS    RESTARTS   AGE
my-deployment-7b54b94746-blvpg   1/1     Running   0          26s

$ kubectl describe pod my-deployment-7b54b94746-blvpg

... # в выводе будет много информации, нас интересует данный блок, относящийся к контейнеру
    Environment:
      TEST:  foo # мы видим, что в контейнере создана переменная "TEST" со значением "foo"
...
```
- С помощью переменных окружения можно передавать различные конфигурации в наши приложения, соответственно, приложение будет считывать переменные окружения и применять их на своё усмотрение, например, таким образом можно передавать различные данные, как правило, конфигурационные
- Единственный, но достаточно большой минус такого подхода - если у вас есть повторяющиеся настройки для различных деплойментов, то нам придётся все эти настройки придется повторять для каждого деплоймента, например, если поменяется адрес БД, то его придётся изменить, скажем, в десятке деплойментов
- Как этого избежать? Очень просто:

## ConfigMap ([00:08:19](https://youtu.be/-xZ02dEF6kU?t=499))
- ConfigMap, как и всё в kubernetes, описывается в yaml файле, в котором, в формате key:value описаны настройки, которые можно использовать в нашем приложении
- в ConfigMap имеется раздел data, собственно там мы и задаём наши настройки, а потом этот словарь, этот ConfigMap, целиком указать в манифесте деплоймента, чтобы из него создать соответствующие переменные окружения в нашем контейнере
- Таким образом, за счёт ConfigMap мы можем уменьшить дублирование кода и упростить настройку однотипных приложений

### Идём в терминал ([00:09:27](https://youtu.be/-xZ02dEF6kU?t=567))
- В том же каталоге `~/school-dev-k8s/practice/4.saving-configurations/1.env` открываем манифест
```yaml
# configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap-env
data:                   # интересующий нас раздел с настройками
  dbhost: postgresql    # параметр 1
  DEBUG: "false"        # параметр 2
...
```
- Применяем наш ConfigMap и смотрим что получилось
```shell
$ kubectl apply -f configmap.yaml

configmap/my-configmap-env created

$ kubectl get cm # cm - сокращение для configmap

NAME               DATA   AGE
kube-root-ca.crt   1      11d   # служебный configmap, созданный автоматически
my-configmap-env   2      79s   # результат наших действий
```
- DATA со значением 2 означает, что в данном конфигмапе находится 2 ключа

### Как использовать ConfigMap ([00:10:56](https://youtu.be/-xZ02dEF6kU?t=656))
- Традиционно, через специально подготовленный манифест
```yaml
# deployment-with-env-cm.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: quay.io/testing-farm/nginx:1.12
        name: nginx
        env:
        - name: TEST
          value: foo
        envFrom:                    # раздел для загрузки переменных окружения извне
        - configMapRef:             # указываем, что будем брать их по ссылке на конфигмап
            name: my-configmap-env  # указываем наш объект ConfigMap, из которого будут загружаться данные
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
...
```
- Результатом применения данного файла будет деплоймент, содержащий в себе контейнер с переменными окружения, прописанными в разделе DATA конфигмапа **my-configmap-env**
- Но если мы заглянем в информацию о созданном таким образом поде, то увидим похожую картину:
```shell
...
    Environment Variables from:
      my-configmap-env  ConfigMap  Optional: false
    Environment:
      TEST:  foo
...
```
- То есть, переменные, заданные через env мы можем видеть, а через envFrom - нет, только название объекта, из которого они загружаются
- Мы можем заглянуть в ConfigMap, чтобы увидеть, что должно быть записано в переменных окружения:
```shell
$ kubectl get cm my-configmap-env -o yaml

apiVersion: v1
data:                   # интересующий нас раздел
  DEBUG: "false"
  dbhost: postgresql
kind: ConfigMap
# далее ещё много всего, в основном, создаваемые автоматически значения по умолчанию
```
- Также, мы можем сходить внутрь контейнера и через его шелл посмотреть переменные окружения, примерно так:
```shell
$ kubectl exec -it my-deployment-7d7cff784b-rjb55 -- bash
# "--" - это разделитель, обозначающий конец команды по отношению к kubectl и начало команды для пода
# в реультате выполнения увидим приглашение:
root@my-deployment-7d7cff784b-rjb55:/#
# выйти можно по сочетанию ctrl+d или командой exit

# также, мы можем сразу же обратиться к переменным окружения:
$ kubectl exec -it my-deployment-7d7cff784b-rjb55 -- env
# на выходе будет список всех переменных окружения
```

###  Что будет, если данные в ConfigMap поменяются? ([00:15:22](https://youtu.be/-xZ02dEF6kU?t=922))
- У запущенного контейнера переменные окружения поменять не так просто и kubernetes этим не занимается
- Если контейнер или под, уже запущены, то изменения ConfigMap на его переменных окружения не отразятся
- Таким образом, если мы хотим, чтобы приложения стало работать с новыми переменными окружения, для этого необходимо убить поды и создать новые

### Какой приоритет env и envFrom? ([00:16:13](https://youtu.be/-xZ02dEF6kU?t=973))
- Сергей предлагает поэкспериментировать

 ### Что будет, если разные ConfigMap будут содержать одинаковые переменные ([00:16:38](https://www.youtube.com/watch?v=-xZ02dEF6kU&t=998s))
 - Вероятно, один из конфигмапов "победит"

 #### Ответ на оба вопроса
- Можно найти в [документации](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#envfromsource-v1-core), ctrl+f по "EnvFromSource array", также, разжевано на [SO](https://stackoverflow.com/questions/54398272/override-env-values-defined-in-container-spec)
- Значится там следующее - значения заданные в env не могут быть перезаписаны, а если ключ содержится в разных источниках (таких как ConfigMap), будет использовано значение из последнего

## Secret ([00:17:16](https://youtu.be/-xZ02dEF6kU?t=1036))
- Используется для работы с чувствительными данными, такими как токены доступа, пароли, приватные ключи сертификатов
- Имеет структуру аналогичную ConfigMap, данные задаются в разделе data
- Бывает нескольких типов:
    - generic - самый распространенный тип, обычно используется для токенов и логинов с паролями
    - docker-registry - данные для авторизации в docker registry
        - фактически, является секретом с заранее определённым списком ключей в массиве data, содержит, в частности:
            - ключ, отвечающий за адрес репозитория
            - ключи, отвечающий за логин, пароль и почту
    - tls - предназначен для хранения сертификатов для шифрования данных, для HTTPS TLS протокола
        - как правило, используется в Ingress
        - имеет 2 предопределённых поля:
            - приватный ключ
            - сам подписанный сертификат

### Экспериментируем
#### Создаём секрет ([00:20:53](https://youtu.be/-xZ02dEF6kU?t=1252))
- Переходим в `~/school-dev-k8s/practice/4.saving-configurations/2.secret`
- Подсматриваем в **README.MD** и выполняем команды оттуда:
```shell
# создаём секрет
$ kubectl create secret generic test --from-literal=test1=asdf --from-literal=dbpassword=1q2w3e

secret/test created

$ kubectl get secret

NAME                  TYPE                                  DATA   AGE
default-token-wgc7r   kubernetes.io/service-account-token   3      11d
s024713-token-8vmmm   kubernetes.io/service-account-token   3      11d
test                  Opaque                                2      8s

$ kubectl get secret test -o yaml

apiVersion: v1
data:
  dbpassword: MXEydzNl
  test1: YXNkZg==
kind: Secret
metadata:
  creationTimestamp: "2021-10-12T17:43:10Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:dbpassword: {}
        f:test1: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-10-12T17:43:10Z"
  name: test
  namespace: s024713
  resourceVersion: "31214172"
  selfLink: /api/v1/namespaces/s024713/secrets/test
  uid: 9a98d624-1a40-40d0-a347-9feb852373ab
type: Opaque
```
- Важно! Секреты типа **kubernetes.io/service-account-token** удалять не нужно, т.к. они отвечают за работу с учебным кластером, они автоматически пересоздадутся, но доступ будет утерян
- При создании generic секрета мы получаем на выходе секрет с типом **Opaque**, что означает "непрозрачный". Данное расхождение в именовании сложилось исторически, все об этом знают и живут с этим.
- Заглядывая в вывод `kubectl get` мы видим, что значения секретов в разделе data отличаются от заданных, это они же, закодированные base64
- Важно понимать, что кодирование - это не шифрование и данные приводятся к исходному виду очень просто, без всяких ключей шифрования
- Зачем это нужно, если механизм декодирования так прост?
    - Одна из причин - обработанные таким образом строки хорошо воспринимаются yaml'ом, не нужно думать об экранировании
    - В kubernetes есть механизм RBAC, позволяющий ограничивать права пользователей на доступ к различным объектам kubernetes
        - По умолчанию, возможность просматривать и редактировать секреты через edit есть только у роли администраторов
        - Однако, не стоит забывать, что настройки, доставленные в приложение, могут быть обработаны этим самым приложением, так что, при желании, разработчики могут их получить. Обычно это уже головная боль специалистов по безопасности

#### Посмотрим на деплоймент, который будет использовать наш секрет ([00:28:38](https://youtu.be/-xZ02dEF6kU?t=1718))
```yaml
# deployment-with-secret.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: quay.io/testing-farm/nginx:1.12
        name: nginx
        envFrom:
        - configMapRef:
            name: my-configmap-env
        env:
        - name: TEST
          value: foo
        - name: TEST_1      # здесь мы можем указать название переменной, отличное от имени ключа в секрете или конфигмапе
          valueFrom:        # таким образом можно получать значения конкретных ключей из конфигмапов и секретов
            secretKeyRef:
              name: test
              key: test1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
...
```
- Применяем и смотрим что получилось:
```shell
$ kubectl apply -f deployment-with-secret.yaml

deployment.apps/my-deployment configured

$ kubectl get pod

NAME                             READY   STATUS    RESTARTS   AGE
my-deployment-57b8cc674c-qc9rk   1/1     Running   0          22s

$ kubectl describe pod my-deployment-57b8cc674c-qc9rk

# нас интересует данная секция:
...
    Environment Variables from:
      my-configmap-env  ConfigMap  Optional: false
    Environment:
      TEST:    foo
      TEST_1:  <set to the key 'test1' in secret 'test'>  Optional: false
...

```
- Видим, что в describe нашего объекта значение секрета скрыто
- Но если мы сходим внутрь пода через exec, то мы сможем увидеть содержимое переменных окружений (если такая возможность не отключена, как правило, именно так и поступают)

### stringData в секретах ([00:34:37](https://youtu.be/-xZ02dEF6kU?t=2077))
- Пример манифеста:
```yaml
# secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: test
stringData:
  test: updated
...
```
- Мы видим, что здесь значение ключа test незакодировано
- Данный раздел был придуман для упрощения работы с секретами и их перекодировкой из/в base64
- Данные, занесенные в раздел stringData будут перенесены в секреты и закодированы соответствующим образом 
```shell
$ kubectl apply -f secret.yaml
Warning: resource secrets/test is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
secret/test configured
```
- В предупреждении речь идёт о том, что для корректной работы, а именно, для обработки мерджей конфигураций (активной и применяемой по apply), рекомендуется выполнять создание объектов определённым образом, иначе могут возникнуть побочные эффекты, а в нашем конкретном случае kubectl сам исправит данный недочёт
- В конце урока обещали скинуть ссылки на почитать
- После применения данного манифеста, если мы посмотрим на наш секрет, увидим следующее:
```shell
$ kubectl get secret test -o yaml
apiVersion: v1
data:
  dbpassword: MXEydzNl
  test: dXBkYXRlZA==
  test1: YXNkZg==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"test","namespace":"s024713"},"stringData":{"test":"updated"}}
# далее инфа, нам не актуальная
```
- В секции data добавился секрет test
- В аннотации добавился ключи last-applied-configuration, с информацией о применённых нами изменениях

- Почему это важно? Потому что, если мы попробуем исправить наш yaml (в нашем случае мы меняем имя ключа test на test1) и повторно выполнить apply, то мы увидим странное:
```shell
$ kubectl get secrets test -o yaml
apiVersion: v1
data:
  dbpassword: MXEydzNl
  test: ""
  test1: dXBkYXRlZA==
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"test","namespace":"s024713"},"stringData":{"test1":"updated"}}
# далее инфа, нам не актуальная
```
- Наш секрет записался в ключ test1, а значение ключа test обнулилось, т.к. за счёт данных в last-applied-configuration kubernetes понял, что раньше мы работали с ключом test, а теперь его нет
- Далее ведущий демонстрирует, что изменение секрета не затрагивает запущенный под, затем убивает его и проверяет, что в новом поде секрет обновился

## Q&A ([00:44:28](https://youtu.be/-xZ02dEF6kU?t=2668))
- |
    - Q: А если надо передать IP новой POD'ы? Например, заскейлили новый memcached - приложению надо знать все IPs POD'ов memcached'а?
    - A: Это делается по другому, через сущность под названием "сервисы", будет позже, в лекции Марселя, конкретно про memcached, там будут безголовые, так называемые headless сервисы
- |
    - Q: Как из vault передавать пароли в kubernetes
    - A: Это есть в курсе Слёрм-Мега, если вкратце, в vault есть модуль для интеграции с kubernetes и можно научить приложение ходить в vault с токеном, который ему даст kubernetes и по этому токену брать данные из vault
- |
    - Q: Зачем "--" перед env в команде типа `kubectl exec -it my-deployment-7d7cff784b-rjb55 -- env`
    - A: Отвечает за разделение между окончанием команды kubectl и началом команды к поду

## Volumes ([00:47:04](https://youtu.be/-xZ02dEF6kU?t=2824))
- Если вспомнить про docker, docker compose, то мы знаем, что системы исполнения контейнеров позволяют монтировать внутрь контейнера не просто переменные окружения, но еще и файлы, причём, монтировать файлы с диска внутрь контейнера
- В kubernetes пошли дальше и сделали возможность монтировать в контейнер содержимое секретов и конфигмапов в качестве файлов
- В ConfigMap можно указывать многострочные значения, затем к ним можно будет обратиться через секцию манифеста volumes, таким образом, в контейнер можно монтировать файлы, со значениями из ConfigMap, обычно там хранят целые конфигурации

### Экспериментируем ([00:48:32](https://youtu.be/-xZ02dEF6kU?t=3756))))
- Переходим в `~/school-dev-k8s/practice/4.saving-configurations/3.configmap`
- видим манифест ConfigMap с куском конфига nginx:
```yaml
# configmap.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  default.conf: |
    server {
        listen       80 default_server;
        server_name  _;

        default_type text/plain;

        location / {
            return 200 '$hostname\n';
        }
    }
...
```
- В данном случае, всё что после символа | - это многострочное значение ключа default.conf
- Многострочное значение оканчивается там, где отступ будет меньше чем в начале этого значения, т.е. на том же уровне, что и у ключа, в нашем случае default.conf
- Про многострочный текст через символ | можно почитать [здесь](https://habr.com/ru/post/270097/). Официальная документация для меня достаточно трудочитаема, кажется, что-то из этого - [1](https://yaml.org/spec/1.2.2/#8111-block-indentation-indicator), [2](https://yaml.org/spec/1.2.2/#812-literal-style)
- Далее смотрим на манифест деплоймента:
```yaml
# deployment-with-configmap.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: quay.io/testing-farm/nginx:1.12
        name: nginx
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
        volumeMounts:                       # Здесь мы описываем точки монтирования томов внутри контейнера
        - name: config                      # Указываем имя тома для монтирования
          mountPath: /etc/nginx/conf.d/     # Здесь мы указываем точку монтирования 
      volumes:                          # Здесь (на уровень выше!) мы описываем тома
      - name: config                    # Задаём имя тому
        configMap:                      # Указываем, из какого конфигмапа создать том
          name: my-configmap
...
```
- Зачем делать это в 2 этапа, сначала формировать вольюм, затем подключать, почему сразу не подключить данные из конфигмапа в контейнер?
- Потому что обычно один и тот же том монтируют сразу в несколько контейнеров
- Аналогичные манипуляции можно применять не только к конфигмапам, но и к секретам
- Если мы применим данный конфигмап и деплоймент, то сможем увидеть в нашем контейнере файл, и его содержимое:
```shell
$ kubectl exec -it my-deployment-5dbbd56b95-xdb2j -- ls /etc/nginx/conf.d/

default.conf

$ kubectl exec -it my-deployment-5dbbd56b95-xdb2j -- cat /etc/nginx/conf.d/default.conf

server {
    listen       80 default_server;
    server_name  _;

    default_type text/plain;

    location / {
        return 200 '$hostname\n';
    }
}
```
- Причём, мы видим, что в созданном файле никаких лишних отступов, как при записи в yaml файле, нет
- Есть ограничения на размер создаваемых файлов (не понятно, техническое или речь о здравом смысле), не стоит создавать файлы по 20МБ
- Очень часто таким образом монтируются файлы с tls сертификатами из секретов

#### Кое что ещё про монтирование конгфигмапов как файлы ([01:21:40](https://youtu.be/-xZ02dEF6kU?t=4900))
- Если мы в поде перейдём в директорию /etc/nginx/conf.d и набёрём ls -lыa, мы увидим что часть файлов являются симлинками (l)
```shell
$ k exec -it my-deployment-5dbbd56b95-lcztg -- bash
# cd /etc/nginx/conf.d
# ls -lsa

total 12
4 drwxrwxrwx. 3 root root 4096 Oct 13 20:49 .
4 drwxr-xr-x. 3 root root 4096 Apr 30  2018 ..
4 drwxr-xr-x. 2 root root 4096 Oct 13 20:49 ..2021_10_13_20_49_15.281823334
0 lrwxrwxrwx. 1 root root   31 Oct 13 20:49 ..data -> ..2021_10_13_20_49_15.281823334
0 lrwxrwxrwx. 1 root root   19 Oct 13 20:49 default.conf -> ..data/default.conf
```
- Остановился здесь #todo

## kubectl port-forward ([00:54:48](https://youtu.be/-xZ02dEF6kU?t=3528))
- Позволяет настроить перенаправления порта извне кластера в под
- Полезно при отладке, тестировании, в процессе разработке, в общем, не для боевого использования
- Выполняется следущим образом:
```shell
$ kubectl port-forward my-deployment-5b47d48b58-l4t67 8080:80 &
# амперсанд в конце нужен, чтобы отправить команду в фон. Вернуться к ней можно через команду %, завершить, как обычно, по ctrl+c
# возможно, нужно будет поиграть с подбором порта, если 8080 будет занят
# видим на выходе что-то подобное:
[1] 26503
Forwarding from 127.0.0.1:8080 -> 80

$ curl 127.0.0.1:8080
# стучимся на данный порт и видим ответ:
Handling connection for 8080
my-deployment-5dbbd56b95-xdb2j
```
- Причём тут port-forward? Чтобы проверить, что наш конфиг из предыдущего шага применился и отработал 
- Далее меняем в нашем конфигмапе строку с возвратом, например, добавляем туда `OK\n` в конце, чтобы увидеть разницу
- В отличие от переменных окружения, изменение конфигмапа, который мы подключаем как том, отразится на содержимом файла в контейнере (не мгновенно, через несколько секунд)
- Далее снова пробуем стучаться на проброшенный порт, но ответ неизменен
- Это происходит потому что nginx сам по себе не перечитывает конфиг

## Downward API ([01:06:54](https://youtu.be/-xZ02dEF6kU?t=4014))
- Документация:
    - [Expose Pod Information to Containers Through Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)
    - [Expose Pod Information to Containers Through Files](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
- Позволяет передать приложению некоторые параметры манифестов как переменные окружения или как файлы
- Позволяет передать приложению различную информацию о том, где оно запущено, например:
    - адрес узла
    - название узла
    - название неймспейса
    - название пода
    - адрес пода
    - реквесты и лимиты, которые описаны в манифесте пода
        - на прошлой лекции был вопрос, как приложение может узнать, какие ему заданы реквесты и лимиты, есть 2 варианта:
            - если приложение или джава там всякая (чиво ?) уже умеет смотреть в [cgroups](<https://ru.wikipedia.org/wiki/%D0%9A%D0%BE%D0%BD%D1%82%D1%80%D0%BE%D0%BB%D1%8C%D0%BD%D0%B0%D1%8F_%D0%B3%D1%80%D1%83%D0%BF%D0%BF%D0%B0_(Linux)>) и брать значения оттуда
            - взять значения из переменных окружения, которые мы можем передать туда с помощью Downward API из манифеста нашего контейнера

### Экспериментируем ([01:08:30](https://youtu.be/-xZ02dEF6kU?t=4110))
- Идем в каталог `~/school-dev-k8s/practice/4.saving-configurations/4.downward`
- Заглядываем в деплоймент:
```yaml
# deployment-with-downward-api.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: quay.io/testing-farm/nginx:1.12
        name: nginx
        env:
        - name: TEST
          value: foo
        - name: TEST_1
          valueFrom:
            secretKeyRef:
              name: test
              key: test1
        - name: __NODE_NAME                         # Начало первого интересующего нас блока
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName              # Название узла, ноды, где запущен под
        - name: __POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name              # Имя пода из нашего манифеста
        - name: __POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace         # Неймспейс пода
        - name: __POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP               # IP-адрес пода
        - name: __NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP              # IP-адрес ноды
        - name: __POD_SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName    # Задаём сервис-аккаунт, будем разбираться далее, в лекции по RBAC
        ports:                                      # Конец первого интересующего нас блока
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi
        volumeMounts:                               # Начало второго интересующего нас блока
        - name: config
          mountPath: /etc/nginx/conf.d/
        - name: podinfo                             # Монтируем том downwardAPI, аналогично монтированию тома конфигмапы
          mountPath: /etc/podinfo
      volumes:
      - name: config
        configMap:
          name: my-configmap
      - name: podinfo                               # Описываем создание тома downwardAPI
        downwardAPI:
          items:
            - path: "labels"                        # Указываем название создаваемого файла
              fieldRef:                             # Указываем, что мы заполняем наш файл на базе ссылки на поля манифеста
                fieldPath: metadata.labels          # Указываем раздел манифеста, из которого мы будем забирать информацию
            - path: "annotations"
              fieldRef:
                fieldPath: metadata.annotations
...                                                 # Конец второго интересующего нас блока
```
- Нам интересны блоки подобные этому:
```yaml
- name: __POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
```
- В поле fieldPath мы указываем интересующие нас поля, которые мы хотим передать внутрь контейнера, это путь согласно yaml манифесту, структура обращения такая же как при вызове команды kubectl explain
- Некоторые значения берутся из манифеста, некоторые подставляет kubernetes, например, те что в секции status. О каждом имеет смысл читать отдельно в документации
- Аннотации (annotations) - кастомные поля для манифеста, применяются для данных, под которые еще не придумали специальных полей (к данной теме особо не относится)

## Q&A 
- [01:14:35](https://youtu.be/-xZ02dEF6kU?t=4475s)
    - Q: Для чего сделана такая реализация (подтягивание изменений из configmap/secret)? Стоит ли использовать этот механизм для обновления конфига своих приложений "налету"?
    - A: Собственно, для этого такая реализвация и придумана, чтобы обновлять конфиг приложений на лету. Стоит ли использовать - вопрос архитектору приложения. По мнению Сергея лучше выполнять передеплой приложения в случаях, когда изменилась конфигурация, git как единая точка правды, IaC.
- [01:15:37](https://youtu.be/-xZ02dEF6kU?t=4537)
    - Q: Как port-forward проксирует запрос в кластере? как-то настраивает ингрес контролер?
    - A: Нет, не нужно никаких ингрес контроллеров, просто kubectl создаёт соединение с API сервером, остаётся запущенным в бэкграунде, слушает указанный для форварда порт, в нашем примере 8080, получает запрос, оборачивает его в HTTP запрос, передаёт его в kubernetes API, API дальше передаёт запрос это конкретному поду, получает ответ и обратно через такой вот туннель передаёт

### Возвращаемся к консоли ([01:16:39](https://youtu.be/-xZ02dEF6kU?t=4599))
- Сергей примененяет деплоймент и смотрит в переменные окружения пода, демонстрирует, что переменные, начинающиеся с `__` заполнены
- Далее Сергей сверяет полученные данные с теми, что можно увидеть по команде `kubectl get pod -o wide`, данная команда даёт расширенную информацию, в том числе IP адрес пода, имя ноды

#### Смотрим в полученный том podinfo ([01:20:10](https://youtu.be/-xZ02dEF6kU?t=4810))
- Идём внутрь пода и выводим содержимое файлов в папке /etc/podinfo:
    - annotations содержит различные автоматически сгенерированные данные, в частности:
        - Когда под был создан
        - Как он был создан, в нашем случае, через API
        - Какая применена политика безопасности, в нашем случае restricted
    - labels показывает метки на нашем поде, в нашем случае это `app="my-app"`, если вспомнить прошлую лекцию, по этим меткам деплоймент определяет принадлежность подов