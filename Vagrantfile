domain = "kubernetes.lab"
worker_node_count = 1
control_plane_endpoint = "k8s-master." + domain + ":6443"
pod_network_cidr = "10.244.0.0/16"
pod_network_type = "calico" # choose between calico and flannel
K8S_POD_NETWORK_TYPE = "pod_network_type"
master_node_ip = "192.168.57.100"

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false
    config.vm.provision :shell, path: "kubeadm/bootstrap.sh"
    config.vm.define "master" do |master|
      master.vm.box = "ubuntu/focal64"
      master.vm.hostname = "k8s-master.#{domain}"
      master.vm.network "private_network", ip: "#{master_node_ip}"
      master.vm.provision "shell", env: {"DOMAIN" => domain, "MASTER_NODE_IP" => master_node_ip} ,inline: <<-SHELL 
      echo "$MASTER_NODE_IP k8s-master.$DOMAIN k8s-master" >> /etc/hosts 
      SHELL
      (1..worker_node_count).each do |nodeIndex|
        master.vm.provision "shell", env: {"DOMAIN" => domain, "NODE_INDEX" => nodeIndex}, inline: <<-SHELL 
        echo "192.168.57.10$NODE_INDEX k8s-worker-$NODE_INDEX.$DOMAIN k8s-worker-$NODE_INDEX" >> /etc/hosts 
        SHELL
      end
      master.vm.provision "shell", path:"kubeadm/init-master.sh", env: {"K8S_CONTROL_PLANE_ENDPOINT" => control_plane_endpoint, "K8S_POD_NETWORK_CIDR" => pod_network_cidr, "K8S_POD_NETWORK_TYPE" => pod_network_type, "MASTER_NODE_IP" => master_node_ip}
    end
    (1..worker_node_count).each do |nodeIndex|
      config.vm.define "worker-#{nodeIndex}" do |worker|
        worker.vm.box = "ubuntu/focal64"
        worker.vm.hostname = "k8s-worker-#{nodeIndex}.#{domain}"
        worker.vm.network "private_network", ip: "192.168.57.10#{nodeIndex}"
        worker.vm.provision "shell", env: {"DOMAIN" => domain, "MASTER_NODE_IP" => master_node_ip} ,inline: <<-SHELL 
        echo "$MASTER_NODE_IP k8s-master.$DOMAIN k8s-master" >> /etc/hosts 
        SHELL
        (1..worker_node_count).each do |hostIndex|
            worker.vm.provision "shell", env: {"DOMAIN" => domain, "NODE_INDEX" => hostIndex}, inline: <<-SHELL 
            echo "192.168.57.10$NODE_INDEX k8s-worker-$NODE_INDEX.$DOMAIN k8s-worker-$NODE_INDEX" >> /etc/hosts 
            SHELL
        end
        worker.vm.provision "shell", path:"kubeadm/init-worker.sh"
        worker.vm.provision "shell", env: { "NODE_INDEX" => nodeIndex}, inline: <<-SHELL 
            echo ">>> FIX KUBELET NODE IP"
            echo "Environment=\"KUBELET_EXTRA_ARGS=--node-ip=192.168.57.10$NODE_INDEX\"" | sudo tee -a /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
            sudo systemctl daemon-reload
            sudo systemctl restart kubelet
            SHELL
      end
    end
    config.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
      vb.cpus = "2"
      vb.customize ["modifyvm", :id, "--nic1", "nat"]
    end
  end


  
