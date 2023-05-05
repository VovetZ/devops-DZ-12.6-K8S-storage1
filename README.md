# devops-DZ-12.6-K8S-storage1
# Домашнее задание к занятию «Хранение в K8s. Часть 1»

### Цель задания

В тестовой среде Kubernetes нужно обеспечить обмен файлами между контейнерам пода и доступ к логам ноды.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке MicroK8S](https://microk8s.io/docs/getting-started).
2. [Описание Volumes](https://kubernetes.io/docs/concepts/storage/volumes/).
3. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1

**Что нужно сделать**

Создать Deployment приложения, состоящего из двух контейнеров и обменивающихся данными.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.
2. Сделать так, чтобы busybox писал каждые пять секунд в некий файл в общей директории.
3. Обеспечить возможность чтения файла контейнером multitool.
4. Продемонстрировать, что multitool может читать файл, который периодоически обновляется.
5. Предоставить манифесты Deployment в решении, а также скриншоты или вывод команды из п. 4.

------

### Задание 2

**Что нужно сделать**

Создать DaemonSet приложения, которое может прочитать логи ноды.

1. Создать DaemonSet приложения, состоящего из multitool.
2. Обеспечить возможность чтения файла `/var/log/syslog` кластера MicroK8S.
3. Продемонстрировать возможность чтения файла изнутри пода.
4. Предоставить манифесты Deployment, а также скриншоты или вывод команды из п. 2.

------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.

------

## Ответ

Проверим установку MicroK8S

```bash
kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
13-kubernetes   Ready    <none>   16m   v1.26.3
microk8s kubectl get pod -A
```

## Задание 1

### 1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool

### 2. Сделать так, чтобы busybox писал каждые пять секунд в некий файл в общей директории

### 3. Обеспечить возможность чтения файла контейнером multitool

Создадим `deploy1.yml` с развёртыванием двух контейнеров _busybox_ и _multitool_ с общим volume

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy1
  labels:
    app: deploy1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: deploy1
  template:
    metadata:
      labels:
        app: deploy1
    spec:
      containers:
        - name: busybox
          image: busybox
          command: ['sh', '-c', 'count=1; datafile="/out/out.txt"; while true; do echo "$((count++))||Process ID $$||$(date)" >> $datafile; sleep 10; done']
          volumeMounts:
            - name: vol1
              mountPath: /out
        - name: multitool
          image: wbitt/network-multitool
          volumeMounts:
            - name: vol1
              mountPath: /in
      volumes:
        - name: vol1
          emptyDir: {}
```

</details>

[Ссылка на deploy1.yml](deploy1.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk/12.6# kubectl create -f deploy1.yml
deployment.apps/deploy1 created
```

Проверим состояние подов

```bash
root@vkvm:/home/vk/12.6# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
deploy1-758f6c8b8c-zd4g4   2/2     Running   0          9s

root@vkvm:/home/vk/12.6# kubectl describe pod deploy1-758f6c8b8c-zd4g4 | grep -iE '(Mounts|Volumes)' -A2
    Mounts:
      /out from vol1 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fmc2n (ro)
--
    Mounts:
      /in from vol1 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fmc2n (ro)
--
Volumes:
  vol1:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
```

### 4. Продемонстрировать, что multitool может читать файл, который периодоически обновляется

Проверим чтение файла из контейнера _multitool_

```bash
root@vkvm:/home/vk/12.6# kubectl exec -it deploy1-758f6c8b8c-zd4g4 -c multitool -- tail -f /in/out.txt
1 --- Wed May  3 16:05:50 UTC 2023
2 --- Wed May  3 16:06:00 UTC 2023
3 --- Wed May  3 16:06:10 UTC 2023
4 --- Wed May  3 16:06:20 UTC 2023
5 --- Wed May  3 16:06:30 UTC 2023
6 --- Wed May  3 16:06:40 UTC 2023
7 --- Wed May  3 16:06:50 UTC 2023
8 --- Wed May  3 16:07:00 UTC 2023
9 --- Wed May  3 16:07:10 UTC 2023
10 --- Wed May  3 16:07:20 UTC 2023
```

Убедились, что файл, который был записан в одном контейнере, доступен на чтение из другого контейнера

### 5. Предоставить манифесты Deployment в решении, а также скриншоты или вывод команды из п. 4

Удалим развёрнутые ресурсы

```bash
root@vkvm:/home/vk/12.6# kubectl delete -f deploy1.yml
deployment.apps "deploy1" deleted
```

## Задание 2

### 1. Создать DaemonSet приложения, состоящего из multitool

### 2. Обеспечить возможность чтения файла `/var/log/syslog` кластера MicroK8S

Создадим `deploy2.yml` с развёртыванием контейнера с доступом к локальному файлу.

<details><summary>Посмотреть YAML...</summary>

```yml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: deploy2
  labels:
    app: deploy2
  namespace: default
spec:
  selector:
    matchLabels:
      app: deploy2
  template:
    metadata:
      labels:
        app: deploy2
    spec:
      containers:
        - name: multitool
          image: wbitt/network-multitool
          volumeMounts:
            - name: vol2
              mountPath: /host/bootlog
      volumes:
        - name: vol2
          hostPath:
            path: /var/log/bootlog
```

</details>

[Ссылка на deploy2.yml](deploy2.yml)

Запустим развёртывание

```bash
root@vkvm:/home/vk/12.6# kubectl create -f deploy2.yml
daemonset.apps/deploy2 created
```

Проверим состояние подов 

```bash
root@vkvm:/home/vk/12.6# kubectl get pods
NAME            READY   STATUS    RESTARTS   AGE
deploy2-c7ss5   1/1     Running   0          10s
root@vkvm:/home/vk/12.6# kubectl get daemonset
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
deploy2   1         1         1       1            1           <none>          17s
root@vkvm:/home/vk/12.6# kubectl describe pod deploy2-c7ss5 | grep -iE '(Mounts|Volumes)' -A2
    Mounts:
      /host/boot.log from vol2 (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-j4j2l (ro)
--
Volumes:
  vol2:
    Type:          HostPath (bare host directory volume)
```

### 3. Продемонстрировать возможность чтения файла изнутри пода

Проверим чтение файла из пода

```bash
kubectl exec -it deploy2-c7ss5 -- tail /host/boot.log
root@vkvm:/home/vk/12.6# kubectl exec -it deploy2-c7ss5 -- tail /host/boot.log
[  OK  ] Finished Permit User Sessions.
[  OK  ] Started Authorization Manager.
[  OK  ] Started Power Profiles daemon.
[  OK  ] Reached target Sound Card.
         Starting Modem Manager...
         Starting GNOME Display Manager...
         Starting Hold until boot process finishes up...
         Starting Hostname Service...
[  OK  ] Started Dispatcher daemon for systemd-networkd.
[  OK  ] Started Hostname Service.
```

Убедились, что файл на локальной машине доступен на чтение из пода

### 4. Предоставить манифесты Deployment, а также скриншоты или вывод команды из п. 2

Удалим развёрнутые ресурсы

```bash
root@vkvm:/home/vk/12.6# kubectl delete -f deploy2.yml
daemonset.apps "deploy2" deleted
```
