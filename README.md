# Домашнее задание №10: Установка Kubernetes (kubeadm и Kubespray)

## Цель работы
Освоить два подхода к развертыванию Kubernetes-кластеров: ручной с помощью kubeadm и автоматизированный с помощью Kubespray, включая конфигурацию высокой доступности (HA).

---

## Задание 1. Установка кластера с 1 мастер-узлом (kubeadm)

### 1.1. Подготовка узлов
Было подготовлено 5 узлов с Ubuntu 22.04:
- 1 мастер (control plane)
- 4 рабочих ноды

На всех узлах выполнены:
- отключение swap,
- настройка модулей ядра и параметров сети,
- установка containerd,
- установка kubeadm, kubelet, kubectl.

### 1.2. Инициализация master-узла
На мастер-узле выполнена команда:
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
Лог инициализации сохранён в task1/kubeadm-init.log.

Сгенерированная команда для присоединения рабочих нод сохранена в task1/kubeadm-join-command.txt.

1.3. Настройка kubectl и установка CNI
Настроен доступ для обычного пользователя:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
Установлен сетевой плагин Calico:

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28/manifests/calico.yaml
1.4. Присоединение worker-узлов
На каждом из 4 рабочих узлов выполнена команда kubeadm join.

1.5. Проверка состояния кластера

kubectl get nodes
Вывод сохранён в task1/nodes.txt:

NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   10m   v1.31.0
worker1  Ready    <none>          5m    v1.31.0
worker2  Ready    <none>          5m    v1.31.0
worker3  Ready    <none>          5m    v1.31.0
worker4  Ready    <none>          5m    v1.31.0
bash
kubectl get pods -A
Вывод системных подов сохранён в task1/pods.txt. Все поды в статусе Running.

рис.1_00, рис.1_01, рис.1_03, рис.1_04, рис.1_06, рис.1_07

Задание 2*. Установка HA-кластера (Kubespray)
2.1. Подготовка узлов и управляющей машины
Для HA-кластера подготовлено:

3 мастер-ноды (control plane)

2 рабочие ноды

отдельная управляющая машина с Kubespray

На всех узлах выполнены базовые настройки (аналогично заданию 1).

2.2. Конфигурация Kubespray
Создан инвентарный файл inventory.ini (копия в task2/inventory.ini):

ini
[all]
master1 ansible_host=10.0.0.1 ip=10.0.0.1
master2 ansible_host=10.0.0.2 ip=10.0.0.2
master3 ansible_host=10.0.0.3 ip=10.0.0.3
worker1 ansible_host=10.0.0.4 ip=10.0.0.4
worker2 ansible_host=10.0.0.5 ip=10.0.0.5

[kube_control_plane]
master1
master2
master3

[kube_node]
worker1
worker2

[etcd]
master1
master2
master3
В group_vars/all/all.yml настроен виртуальный IP для доступа к API:

apiserver_loadbalancer_domain_name: "my-cluster-endpoint"
loadbalancer_apiserver:
  address: 10.0.0.100   # VIP
  port: 6443
Также включена поддержка Keepalived для обеспечения доступности VIP. Соответствующие конфигурации приложены:

task2/all.yml

task2/k8s-cluster.yml

2.3. Запуск развертывания
На управляющей машине выполнен плейбук:

ansible-playbook -i inventory/mycluster/inventory.ini --become --become-user=root cluster.yml
Лог выполнения сохранён в task2/deployment.log. Установка заняла ~15 минут.

2.4. Проверка HA-кластера
После завершения получен файл admin.conf для доступа к кластеру. Проверка узлов:

kubectl --kubeconfig=artifacts/admin.conf get nodes
Вывод сохранён в task2/nodes.txt:

NAME      STATUS   ROLES           AGE   VERSION
master1   Ready    control-plane   12m   v1.31.0
master2   Ready    control-plane   12m   v1.31.0
master3   Ready    control-plane   12m   v1.31.0
worker1   Ready    <none>          10m   v1.31.0
worker2   Ready    <none>          10m   v1.31.0
Проверка системных подов:

kubectl --kubeconfig=artifacts/admin.conf get pods -A
Все поды запущены (файл task2/pods.txt).

2.5. Тестирование отказоустойчивости
Была остановлена одна из мастер-нод (master1). API кластера оставался доступен через VIP. Команды kubectl продолжали выполняться. После восстановления ноды она успешно вернулась в кластер.

рис.2_01,рис.2_02, рис.2_02_01,,рис.2_02_02, рис.2_03, рис.2_03+, рис.2_03++, рис.2_03+++, рис.2_04, рис.2_04+, рис.2_05 

Выводы
В ходе работы были успешно выполнены:

Установка кластера из 5 нод с помощью kubeadm (1 master, 4 worker) и настройка сетевого плагина Calico.

Развертывание production-ready HA-кластера с 3 master-нодами, 2 worker-нодами и виртуальным IP с использованием Kubespray.

Проверка работоспособности и отказоустойчивости кластера.

Оба подхода показали свою эффективность: kubeadm даёт полный контроль над процессом, Kubespray позволяет быстро развернуть сложную конфигурацию. Полученные навыки являются основой для администрирования Kubernetes в production-среде.
