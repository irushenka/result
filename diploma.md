
### Создание облачной инфраструктуры

С помощью Terraform создана инфраструктура:
- Три виртуальные машины для использования Kubernetes
- Одна виртуальная машина для установки Jenkins
- Для постоянного ip-адреса балансировщик Network Load Balaner.
- Сеть с подсетями в разных зонах
- Реестр
- Сервисный аккаунт с необходимыми ролями

[Ссылка на репозиторий с конфигами Terraform](https://github.com/irushenka/terraform_configs)


### Создание Kubernetes кластера

С помощью Kubespray задеплоена Kubernetes на три созданные ранее виртуальные машины:
- Изменен файл hosts.yaml - добавлены ip созданных виртуальных машин
- Выполнена команда для установки кластера:

```
ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v
```

- Заходим на Control plane копируем конфиги в $HOME/.kube/config, проверяем, что все запустилось:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-11%2019-47-15.png)

### Создание тестового приложения

В качестве тестового приложения использован nginx, в котором изменена стартовая страница.
В git репозиторий добавлены следующие файлы:
- Dockerfile:

```
FROM nginx:latest
COPY index.html /usr/share/nginx/html

//EXPOSE 8080
```

- index.html:

```
<html><body><p>Hello World!</p></body></html>
```
Yandex Container Registry создан на первом шаге с помощью Terraform:
![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-11%2023-18-37.png)

### Подготовка cистемы мониторинга и деплой приложения

Для деплоя prometheus, grafana, alertmanager, экспортер в кластер был использован пакет kube-prometheus.
Настройка согласно описанию:

```
git clone https://github.com/prometheus-operator/kube-prometheus
cd kube-prometheus

kubectl apply --server-side -f manifests/setup
kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
kubectl apply -f manifests/

kubectl --namespace monitoring port-forward svc/grafana 3000
```

Проверяем сервисы:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-13%2016-30-25.png)


Проверяем доступ к Grafana:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-13%2016-26-49.png)


### Установка и настройка CI/CD

В качестве инструмента CI/CD использовался Jenkins.
Для автоматизации сборки были выполнены следующие шаги:
- В  Jenkins установлены плагины Docker Pipeline, SSH Agent
- В  Jenkins созданы credentials для доступа к Yandex Container Registry и для доступа по SSH на вирутальную машину
- На виртуальной машине с Jenkins был установлен docker
- В git репозитории с приложением включен webhook для запуска автоматической сборки в Jenkins
- Создан Pipeline:

```
pipeline {
    
    agent any
    
    environment {
      imagename = "crpa0g9it2tedrcl22gh/app"
      registryCredential = 'yandex'
      dockerImage = ''
    } 
    
    stages {
        stage('Cloning Git repository') {
            steps{
                checkout([$class: 'GitSCM', branches: [[name: '*']], extensions: [], userRemoteConfigs: [[credentialsId: 'bb32435a-45d6-45fd-a459-b25b8218cdf2', url: 'https://github.com/irushenka/test-server-app']]])
            }
        }
        stage('Build and deploy image') {
            steps{
                script {
                    docker.withRegistry('http://cr.yandex/crpa0g9it2tedrcl22gh', registryCredential) {
                        dockerImage = docker.build imagename
                        dockerImage.push()
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps{
                sshagent (credentials: ['ssh']) {
                    sh 'ssh -o StrictHostKeyChecking=no -l irushenka 51.250.10.14 uname -a'
                    sh 'scp service.yaml irushenka@51.250.10.14:'
                    script {
                        try{
                            sh 'ssh irushenka@51.250.10.14 kubectl apply -f service.yaml'
                        }catch(error){}
                    }
                }
            }
        }
    }
}
```

Файл service.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app
  name: app
  namespace: default
spec:
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - image: cr.yandex/crpa0g9it2tedrcl22gh/app:latest
        imagePullPolicy: IfNotPresent
        name: nginx
            
---
apiVersion: v1
kind: Service
metadata:
  name: app
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: app
  type: NodePort
```

Docker-образ попал в репозиторий:
![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-11%2023-18-45.png)

Для получения порта на Control plane выполнена команда:

```
kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
app          NodePort    10.233.7.207   <none>        80:32627/TCP   24s
```
Этот порт добавляем вручную в обработчик на балансировщике.

Смотрим ip-адрес балансировщика:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-25%2002-34-42.png)


Открывает ip-адрес балансировщика и порт и проверяем, что все работает:

![alt text](https://github.com/irushenka/screens/blob/main/Screenshot%20from%202022-12-25%2002-33-49.png)
