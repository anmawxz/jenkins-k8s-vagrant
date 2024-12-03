Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64" # Ubuntu 22.04
  config.vm.network "forwarded_port", guest: 8080, host: 8080 # Porta do Jenkins
  config.vm.network "forwarded_port", guest: 30000, host: 30000 # Porta do Kubernetes Dashboard (NodePort)
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "4096"
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: <<-SHELL
    # Atualizar pacotes
    apt-get update
    apt-get install -y apt-transport-https curl

    # Instalar Docker
    curl -fsSL https://get.docker.com -o get-docker.sh
    sh get-docker.sh
    usermod -aG docker vagrant

    # Adicionar repositório para Kubernetes
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list


    # Comentar disabled_plugins line
    if grep -q 'disabled_plugins = \\["cri"\\]' /etc/containerd/config.toml; then
      sed -i 's/^disabled_plugins = \\["cri"\\]/#disabled_plugins = \\["cri"\\]/' /etc/containerd/config.toml
      systemctl restart containerd.service
    fi

    # Adicionar linhas ao final do arquivo
    echo '[plugins."io.containerd.grpc.v1.cri"]' >> "$CONFIG_FILE"
    echo 'sandbox_image = "registry.k8s.io/pause:3.10"' >> "$CONFIG_FILE"
    systemctl restart containerd.service


    # Instalar versão específica do kubelet, kubeadm e kubectl
    apt-get update
    apt-get install -y kubelet=1.31.3-1.1 kubeadm=1.31.3-1.1 kubectl=1.31.3-1.1
    apt-mark hold kubelet kubeadm kubectl

    # Iniciar o cluster Kubernetes
    kubeadm init --pod-network-cidr=192.168.0.0/16
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    export KUBECONFIG=/etc/kubernetes/admin.conf
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /home/vagrant/.bashrc

    # Instalar plugin de rede (Calico)
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
    watch kubectl get pods -n calico-system
    #kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

    # Configurar permissões para o usuário 'vagrant'
    mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown vagrant:vagrant /home/vagrant/.kube/config
  SHELL
end
