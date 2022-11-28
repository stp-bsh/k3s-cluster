default_box = 'ubuntu/jammy64'
node_num = 3
master_mem = '2048'
agent_mem = '4096'

Vagrant.configure(2) do |config|
  config.vm.define 'master' do |master|
    master.vm.box = default_box
    master.vm.hostname = "master"
    master.vm.synced_folder ".", "/vagrant", type:"virtualbox"
    master.vm.network "private_network", ip: "192.168.0.200", netmask: "255.255.255.0", auto_config: true
    master.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", disabled: true
    master.vm.network "forwarded_port", guest: 22, host: 2200 # SSH TO MASTER/NODE
    master.vm.network "forwarded_port", guest: 6443, host: 6443 # ACCESS K8S API
    for p in 30000..30100 # PORTS DEFINED FOR K8S TYPE-NODE-PORT ACCESS
      master.vm.network "forwarded_port", guest: p, host: p, protocol: "tcp"
      end
    master.vm.provider "virtualbox" do |v|
      v.memory = master_mem
      v.name = "master"
      end

    master.vm.provision "shell", inline: <<-SHELL
      sudo apt update
      sudo apt install etcd-server etcd-client -y
      IPADDR=$(ip a show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f1)
      # curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--node-ip=${IPADDR} --flannel-iface=enp0s8 --write-kubeconfig-mode 644 --kube-apiserver-arg="service-node-port-range=30000-30100" --no-deploy=servicelb --no-deploy=traefik" K3S_STORAGE_BACKEND=etcd3 K3S_STORAGE_ENDPOINT="http://127.0.0.1:2379" sh -
      curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--node-ip=${IPADDR} --flannel-iface=enp0s8 --write-kubeconfig-mode 644 --kube-apiserver-arg="service-node-port-range=30000-30100 K3S_STORAGE_BACKEND=etcd3 K3S_STORAGE_ENDPOINT="http://127.0.0.1:2379" sh -
      hostnamectl set-hostname master
      NODE_TOKEN="/var/lib/rancher/k3s/server/node-token"
      while [ ! -e ${NODE_TOKEN} ]
      do
          sleep 1
      done
      cat ${NODE_TOKEN}
      cp ${NODE_TOKEN} /vagrant/
      sudo sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config
      sudo systemctl reload ssh
      KUBE_CONFIG="/etc/rancher/k3s/k3s.yaml"
      cp ${KUBE_CONFIG} /vagrant/ #copy contents of "k3s.yaml" to ".kube/config" to 'kubectl' from local-machine
    SHELL
  end

  (1..node_num).each do |i|
    vm_name = "agent-#{i-1}"
    config.vm.define vm_name do |node|
      node.vm.box = default_box
      node.vm.hostname = "#{vm_name}"
      node.vm.synced_folder ".", "/vagrant", type:"virtualbox"
      private_ip = "192.168.0.#{i+210}"
      ssh_port = "#{i+2200}"
      node.vm.network "private_network", ip: private_ip, netmask: "255.255.255.0", auto_config: true
      node.vm.network "forwarded_port", guest: 22, host: 2222, id: "ssh", disabled: true
      node.vm.network "forwarded_port", guest: 22, host: ssh_port
      node.vm.provider "virtualbox" do |v|
        v.memory = agent_mem
        v.name = "#{vm_name}"
        end
    
      node.vm.provision "shell", inline: <<-SHELL
        IPADDR=$(ip a show enp0s8 | grep "inet " | awk '{print $2}' | cut -d / -f1)
        curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--node-ip=${IPADDR} --flannel-iface=enp0s8" K3S_URL=https://192.168.0.200:6443 K3S_TOKEN=$(cat /vagrant/node-token) sh -
        sudo sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config
        sudo systemctl reload ssh
      SHELL
    end
  end
end
