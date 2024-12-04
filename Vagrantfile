Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64" # Ubuntu 22.04
  config.vm.network "forwarded_port", guest: 30000, host: 30000 # Porta do Kubernetes Dashboard (NodePort)
  config.vm.network "forwarded_port", guest: 32000, host: 32000 # Porta do Jenkins no Kubernetes
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 2
  end
 
  config.vm.provision "shell", inline: <<-SHELL
    # Atualizar pacotes
    apt-get update
    apt-get install -y apt-transport-https curl jq

    # Instalar Docker
    curl -fsSL https://get.docker.com -o get-docker.sh
    sh get-docker.sh
    usermod -aG docker vagrant

    # Adicionar repositório Kubernetes
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    # Ajustar containerd
    sudo su
    CONFIG_FILE=/etc/containerd/config.toml
    sed -i 's/^disabled_plugins = \\["cri"\\]/#disabled_plugins = \\["cri"\\]/' "$CONFIG_FILE"
    echo '[plugins."io.containerd.grpc.v1.cri"]' >> "$CONFIG_FILE"
    echo 'sandbox_image = "registry.k8s.io/pause:3.10"' >> "$CONFIG_FILE"
    systemctl restart containerd.service
    
    # Pull da imagem pause:3.10
    sudo ctr images pull registry.k8s.io/pause:3.10
    sudo kubeadm config images pull

    # Instalar Kubernetes
    apt-get update
    apt-get install -y kubelet=1.31.3-1.1 kubeadm=1.31.3-1.1 kubectl=1.31.3-1.1
    apt-mark hold kubelet kubeadm kubectl

    # Iniciar cluster Kubernetes
    kubeadm init --pod-network-cidr=192.168.0.0/16
    mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown vagrant:vagrant /home/vagrant/.kube/config
    export KUBECONFIG=/etc/kubernetes/admin.conf

    # Instalar Calico
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
    while ! kubectl get pods -n calico-system | grep -q Running; do sleep 5; done

    # Remover Taint
    kubectl taint nodes ubuntu-focal node-role.kubernetes.io/control-plane-

    # Instalar Jenkins
    # kubectl create namespace jenkins
    # kubectl apply -n jenkins -f https://raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/ci/values.yaml
    # kubectl apply -f https://raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/templates/jenkins-controller.yaml

    # Configurar Kubernetes Dashboard
    # kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
    # kubectl -n kubernetes-dashboard create token admin-user | jq -r .status.token > /home/vagrant/dashboard-token.txt

    # Expor Jenkins e Dashboard
    # kubectl expose deployment jenkins --type=NodePort --name=jenkins-service -n jenkins
    # kubectl -n kubernetes-dashboard edit service/kubernetes-dashboard # Configurar NodePort manualmente para 30000

    # Notificar o usuário sobre o token
    # echo "O token do Dashboard Kubernetes foi salvo em /home/vagrant/dashboard-token.txt"
  SHELL
end
