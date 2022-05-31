# Домашнее задание к занятию "13.4 инструменты для упрощения написания конфигурационных файлов. Helm и Jsonnet"
В работе часто приходится применять системы автоматической генерации конфигураций. Для изучения нюансов использования разных инструментов нужно попробовать упаковать приложение каждым из них.

## Задание 1: подготовить helm чарт для приложения

<details>

  <summary>Описание задачи</summary>  
Необходимо упаковать приложение в чарт для деплоя в разные окружения. Требования:
* каждый компонент приложения деплоится отдельным deployment’ом/statefulset’ом;
* в переменных чарта измените образ приложения для изменения версии.

</details>

## Решение

> каждый компонент приложения деплоится отдельным deployment’ом/statefulset’ом;

В директории templates расположены:

- back-deployment.yaml
- front-deployment.yaml
- db-statefulset.yaml
  
  
> в переменных чарта измените образ приложения для изменения версии.

Chart.yaml
```yaml
apiVersion: v2
name: app
description: A Helm chart for Kubernetes

type: application

# Версия приложения определяет тег образов для front и back приложения.
# Указано в back-deployment.yaml и front-deployment.yaml 
# в виде переменных по умолчанию -  image: "{{ .Values.image.back.repository }}:{{ .Values.image.back.tag | default .Chart.AppVersion }}".

version: 0.1.0

appVersion: "1.0.0"
```

values.yaml
```yaml
replicaCount:
  front: 1
  back: 1

image:
  front:
    repository: rdegtyarev/netology-hw-13.04-front
    # При указании пустого tag (tag: "") будет использоваться тег по умолчанию, соотвествующий версии приложения (appVersion)
    tag: ""
  back:
    repository: rdegtyarev/netology-hw-13.04-back
    # При указании пустого tag (tag: "") будет использоваться тег по умолчанию, соотвествующий версии приложения (appVersion)
    tag: ""

resources:
   limits:
     cpu: 200m
     memory: 256Mi
   requests:
     cpu: 100m
     memory: 128Mi

appPort:
  front: 80
  back: 9000
```

Для подключения back к базе необходимо определить неймспейс сервиса. Использую {{ .Release.Namespace }} (./app/templates/back-deployment.yaml)

---

## Задание 2: запустить 2 версии в разных неймспейсах

<details>

  <summary>Описание задачи</summary>  
Подготовив чарт, необходимо его проверить. Попробуйте запустить несколько копий приложения:
* одну версию в namespace=app1;
* вторую версию в том же неймспейсе;
* третью версию в namespace=app2.

</details>

## Решение

1. Создаем namespace:

```bash
kubectl create namespace app1
namespace/app1 created

kubectl create namespace app2
namespace/app2 created
```

2. Запускаем версию 1.0.0 в namespace app1

```bash
helm install app ./charts/v1 -n app1
NAME: app
LAST DEPLOYED: Tue May 31 22:33:54 2022
NAMESPACE: app1
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
---------------------------------------------------------

Content of NOTES.txt appears after deploy.
Deployed front version 1.0.0
Deployed back version 1.0.0
```

Проверяем версию через запрос к api back (модифицировал flask back из предыдущих заданий). Можно также форварднуть front и открыть браузером по 80 порту, откроется страница с указанием номера версии.

```bash
kubectl get pods -n app1
NAME                     READY   STATUS    RESTARTS   AGE
back-7bcf744549-6vsrl    1/1     Running   0          7m45s
db-0                     1/1     Running   0          7m45s
front-7c6ddc646b-25wp2   1/1     Running   0          7m45s

kubectl exec -it -n app1 back-7bcf744549-6vsrl -- curl http://localhost:9000/api/version
"app version 1.0.0"
```

3. Запускаем версию 1.1.0 в namespace app1
С использованием того же имени приложения (app)

```bash
helm install app ./charts/v2 -n app1
Error: INSTALLATION FAILED: cannot re-use a name that is still in use
```
Не удается, т.к. имя уже используется.

С другим именем (app2)

```bash
helm install app2 ./charts/v2 -n app1
Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists. Unable to continue with install: Service "back" in namespace "app1" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-name" must equal "app2": current value is "app"
```
Не удается, т.к. helm применяет манифесты, ресурсы которых уже есть в этом неймспейсе.

3. Запускаем версию 1.2.0 в namespace app2


```bash
 helm install app ./charts/v3 -n app2
NAME: app
LAST DEPLOYED: Tue May 31 22:38:13 2022
NAMESPACE: app2
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
---------------------------------------------------------

Content of NOTES.txt appears after deploy.
Deployed front version 1.2.0
Deployed back version 1.2.0

---------------------------------------------------------
```

Проверяем версию через запрос к api back (модифицировал flask back из предыдущих заданий). Можно также форварднуть front и открыть браузером по 80 порту, откроется страница с указанием номера версии.

```bash
kubectl exec -it -n app2 back-6676d64b7d-qw7sm -- curl http://localhost:9000/api/version
"app version 1.2.0"
```

---
