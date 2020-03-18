# Развертывание High Available Kubernetes кластера

Данный playbook разварачивает HA кластер Kubernetes. Пока что в глубокой aplpha
При развертывании можно выбрать следующие параметры
- Выбрать используемый CIDR
- Выбрать нужен ли HA IP адресс реализованный с помощью VRRP

## Последовательность развертывания:
- на машине, с которой запускается плейбук, создается временный каталог /tmp/k8s. Он понадобится для копирования сертификатов после инициализации первого мастера. Также в него будет добавлена строка инициализации сгенерированная kubeadmin
- на мастерах, рабочих нодах и нодах под ingress устанавливается kubelet, kubectl, kubeadm
- на мастерах, рабочих нодах и нодах под ingress установливается CRI
- на мастер нодах устанавливается ETCD. Настраивается ETCD кластер
- на мастер нодах устанавливается и настраивается keepalived (для этого флаг vrrp=true)
- на master1 при помощи kubeadm init создаем первый мастер кластера сертификаты, ключи и.т.д.
- с помощью сгенерированых файлов конфигурации в кластер добавляются master2 и master3.
- устанавливается CIDR (сейчас только callico)
- добавляются в кластер рабочие ноды (и ingress  ноды если указанны в в группе хостов [ingressnodes])
- устанавливается haproxy как балансировщик запросов с воркеров к API
- настраивается ingress контроллер
- устанавливается kubernetes dashboard
- устанавливается helm
- выводится информация о кластере


## Быстрый старт:

```bash
git clone https://github.com/rjeka/k8s-easy-ha-cluster.git
cd k8s-easy-ha-cluster
vim hosts
```
В файле hosts:
- в группе хостов [masters] изменить значения ansible_host на IP адреса мастеров.  
  Если Вы хотите использовать vrrp keepalivedInterface - поменять на значение интерфейса, на котором будет работать API сервер  kubernetes. На этом же интерфейсе keepalived создаст subinterface c виртуальным IP
- в группе хостов [worknodes]  изменить значения ansible_host на IP адреса рабочих нод
- в группе хостов [ingressnodes] добавить имена и ip нод для ingress контроллера. Если в группе нет хостов ingress контроллер устанавливается на work ноды с replicas: 1. Если ноды в списке есть то на данные ноды ставится label: ingress, устанавливается ingress контроллер и реплицируется на каждую ноду из группы.
- в блоке [all:vars]:
  - coreDns=true - если хотим CoreDns, иначе Kube-Dns
  - criType docker, так как с последних версий docker включает в себя containerd
  - helm если хотите установить в кластер helm то true, если helm не нужен то false
  - DynamicKubeletConfig - в версии kubernetes 1.11 появилась возможность динамического обновления конфигурации, если нужна то true, если нет то false
  - garbageCollection - сборщик мусора, если true то с master1 удялятся скаченные архивы CRI и ETCD, все не нужные конфиг файлы,  останется только строка сгенерированная kubeadm в файле /opt/config/token.txt. Если false то все архивы и конфиги останутся
- [masters:vars]
  - СIDR=192.168.0.0/16 для установки callica
  - K8SHA_TOKEN токен для kubeadmin. Можно сгенерировать kubeadm token generate
  - etcd_version=v3.2.17
  - etcdToken - токен etcd любое значение например 9489bf67bdfe1b4ae067d6fd9e7efefd
  - kepalivedPass - праоль keepalived любое значение например 6cf8dd754c90194d1600c483e10abfr

```bash
ansible-playbook -i hosts main.yaml
```
После того как плейбук отработает, на экран будет выведенна информация о кластере
![GitHub Logo](images/cluster-info.png)


