# Домашнее задание к занятию «Установка Kubernetes» - `Мурчин Артем`

### Цель задания

Установить кластер K8s.

### Чеклист готовности к домашнему заданию

1. Развёрнутые ВМ с ОС Ubuntu 20.04-lts.


### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция по установке kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/).
2. [Документация kubespray](https://kubespray.io/).

-----

### Задание 1. Установить кластер k8s с 1 master node

1. Подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды.
2. В качестве CRI — containerd.
3. Запуск etcd производить на мастере.
4. Способ установки выбрать самостоятельно.

### Решение 1. Установка с kubeadm

Создал 5 виртмашин для развертывания кластера. node1-2004-compute-vm-4-4-50-hdd - это мастер нода. node2-node5-compute-vm-2-2-100-hdd - воркер ноды. Виртмашину node1-2004-compute-vm-4-4-60-hdd пришлось создать для демонстрации скриншотов успешной установки kubernetes, т.к. в истории мастер ноды node1-2004-compute-vm-4-4-50-hdd выполнения этих команд уже не было из-за большого количетва команд связанных с подключением 4-х рабочих нод.

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-01.png)

Первым делом скопировал приватный ключ на мастер ноду и задал права 600 на ключ.

    scp id_ed25519 artem@84.201.154.76:.ssh/id_ed25519
    ~/.ssh$ chmod 600 id_ed25519

Далее сделал порт-форвардинг

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    
    sudo modprobe overlay
    sudo modprobe br_netfilter
    
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF
    
    sudo sysctl --system

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-01.png)

Затем были добавлены сертификат и ключ

    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    sudo mkdir -p -m 755 /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-02.png)

Установил containerd не сразу. Когда дошел до установки кубернетес, вышла ошибка. После просмотра лекции и дз понял, что необходимо установить containerd.

    sudo apt-get install containerd

После этого выполнил команду по установке kubernetes. Указал внутренний ip адрес, внешний ip адрес, а также сеть для подов.

    sudo kubeadm init --apiserver-advertise-address=10.129.0.16 --apiserver-cert-extra-sans=84.201.154.76 --pod-network-cidr=10.244.0.0/16

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-03.png)

Kubernetes успешно установлен!

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-04.png)

Далее поочередно подключался к каждой ноде и устанавливал kubelet, kubeadm, kubectl и containerd с помошью тех же команд, что и на мастер ноде, кроме команды sudo kubeadm init ...

Затем сделал присоединение рабочей ноды к мастер ноде. И так с каждой рабочей нодой:

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-05.png)

Вывод команды kubectl get nodes на мастер-ноде. Ноды не готовы. Установил сетевой плагин из лекции:

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-06.png)

Все ноды готовы и поды запущены:

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-02-07.png)

### Установка с kubespray

Сперва пробовал установить кубернетес с kubespray. На мастер ноде ubuntu2004 не устанавливалось виртуальное окружение. Поэтому ноду развернул на 2404 и нормально команды сработали.

Ансибл все команды завершил успешно. Но после выполнения команды kubectl get nodes ноды не отобразились. Оказалось, что мастер ноду выбрал другую. А виртмашину, с которой запускал команды, не добавил ни в одну из нод. Время дальше тратить не стал и пошел более быстрым способом через kubeadm.

Список использованных команд через kubespray:

    scp id_ed25519 artem@84.201.154.76:.ssh/id_ed25519
    ~/.ssh$ chmod 600 id_ed25519
    
    sudo apt update
    sudo apt install git python3 python3-pip -y
    git clone https://github.com/kubernetes-incubator/kubespray.git
    git clone https://github.com/kubernetes-sigs/kubespray - для ubuntu 20.04
    cd kubespray
    sudo apt install python3.12-venv - создаем виртуальное окружение
    
    python3 -m venv .venv
    source .venv/bin/activate
    
    python3 -m pip install -r requirements.txt
    cp -rfp inventory/sample inventory/mycluster
    declare -a IPS=(10.129.0.27 10.129.0.7 10.129.0.14) - задаем воркер ноды в кластер
    
    pip3 install ruamel.yaml
    CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
    
    ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml -b -v &
    
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

Скриншоты:

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-01-01.png)

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-01-02.png)

![](https://github.com/artmur1/22-3.2-K8S/blob/main/img/22-3_2-01-03.png)

## Дополнительные задания (со звёздочкой)

**Настоятельно рекомендуем выполнять все задания под звёздочкой.** Их выполнение поможет глубже разобраться в материале.   
Задания под звёздочкой необязательные к выполнению и не повлияют на получение зачёта по этому домашнему заданию. 

------
### Задание 2*. Установить HA кластер

1. Установить кластер в режиме HA.
2. Использовать нечётное количество Master-node.
3. Для cluster ip использовать keepalived или другой способ.

### Правила приёма работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl get nodes`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
