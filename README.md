![image](https://github.com/user-attachments/assets/a9ab53b4-8c6d-40ae-a0d5-a29fd3a9adf3)**Задание 1**

Создать Deployment приложения, использующего локальный PV, созданный вручную.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.

2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.

3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.

4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.

5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV. Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.

6. Предоставить манифесты, а также скриншоты или вывод необходимых команд.



**Решение 1**


Создадим деплоймент со следующим содержимым


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-test
  namespace: default
  labels:
    app: dep-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dep-test
  template:
    metadata:
      labels:
        app: dep-test
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'while true; do echo Hello world! >> /output/output.txt; sleep 5; done']
        volumeMounts:
        - name: pv-test
          mountPath: /output
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 80
        env:
        - name: HTTP_PORT
          value: "80"
        volumeMounts:
        - name: pv-test
          mountPath: /input
      volumes:
      - name: pv-test
        persistentVolumeClaim:
          claimName: pvc-test
```

![Image alt](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/1.jpg)


Запускаем, и видим что статус пода Pending, изз атого что не готов PVC


Создадим сначала PV со следующим содержимым

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-test
spec:
  storageClassName: host-path
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /root/Kubernetes7-Storage/data/pv-test
  persistentVolumeReclaimPolicy: Retain
```

Затем PVC запрос на получения места из ранее созданного PV

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: host-path
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```


Создаем PV и проверяем наличие

![Image alt](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/2.jpg)


И тперь для этого PV создаем PVC


![Image alt](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/3.jpg)



Продемонстрируем, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.


![Image alt](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/4.jpg)



Удаляем Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.

![Image alt]([скрин5](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/5.jpg)


После удаления PVC, PV сменил статус с Bound на Released, так как больше не свзан с PVC


Смотрим, что файл сохранился на локальном диске ноды. теперь удалим PV. посмотрим что произошло с файлом после удаления PV. Пояснить, почему.


![Image alt](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/6.jpg)



Удаляем PV. Так как Reclaim Policy было установлено Retain, соответственно данные сохранились

![Image alt](https://github.com/mezhibo/kubernetes7/blob/89ec7c7bbb61bd885186361b53ef11a19b6666e8/IMG/7.jpg)





**Задание 2**


Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

1. Включить и настроить NFS-сервер на MicroK8S.

2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.

3. Продемонстрировать возможность чтения и записи файла изнутри пода.

4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.


**Решение 2**


Установим nsf в k8s


![Image alt](https://github.com/mezhibo/kubernetes7/blob/626ba8e7df7105094f4916ad78bcf2a498fd8c50/IMG/8.jpg)


Проверим состояние службы nfs


![Image alt](https://github.com/mezhibo/kubernetes7/blob/626ba8e7df7105094f4916ad78bcf2a498fd8c50/IMG/9.jpg)



Создадим деплоймент 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nfs
  labels:
    app: app-nfs
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-nfs
  template:
    metadata:
      labels:
        app: app-nfs
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "80"
        ports:
        - containerPort: 80
        volumeMounts:
        - name: vol-nfs
          mountPath: /nfs-share
      volumes:
      - name: vol-nfs
        persistentVolumeClaim:
          claimName: nfs-share
```


Далее создадим pvc-nfs

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-share
spec:
  storageClassName: "nfs-share"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```


Создадим sc_nfs.yaml

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 127.0.0.1
  share: /srv/nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
  - nfsvers=4.1
```


Проверяем что у нас создался под и pv и pvc

![Image alt](https://github.com/mezhibo/kubernetes7/blob/626ba8e7df7105094f4916ad78bcf2a498fd8c50/IMG/10.jpg)


Проверим возможность чтения и записи файла изнутри пода:

Создадим файл на ноде:

![Image alt](https://github.com/mezhibo/kubernetes7/blob/626ba8e7df7105094f4916ad78bcf2a498fd8c50/IMG/11.jpg)



Теперь провалимся внутрь пода нашего приложения, и в примонтированной шаре создадим файл readme.txt

![Image alt](https://github.com/mezhibo/kubernetes7/blob/626ba8e7df7105094f4916ad78bcf2a498fd8c50/IMG/12.jpg)

Теперь выйдем из пода и проверим наличие этого файла в нашей nfs


![Image alt](https://github.com/mezhibo/kubernetes7/blob/626ba8e7df7105094f4916ad78bcf2a498fd8c50/IMG/13.jpg)


И теперь видим что файл появился
