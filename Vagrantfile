# -*- mode: ruby -*-
# vi: set ft=ruby :


$centos_script = <<-SCRIPT

# set timezone
timedatectl set-timezone Asia/Shanghai

yum install -y yum-utils

yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum install -y docker-ce docker-ce-cli containerd.io vim

systemctl enable docker
systemctl start docker

# install docker-compose
if [ ! -d "/usr/local/bin/docker-compose" ];then
    curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
fi

SCRIPT
Vagrant.configure(2) do |config|
   config.vm.box = "generic/rhel7"
   config.vm.box_url = "http://files.saas.hand-china.com/vagrant/bento_centos-7.8.box"
   config.vm.hostname = "docker-node-0"
   config.vm.network "private_network", ip: "192.168.56.100"
   config.vm.synced_folder "/Users/xingminwang/data/docker", "/data/docker"
   config.vm.provision "file", source: "/Users/xingminwang/.ssh/owntest", destination: "/home/vagrant/.ssh/id_rsa"
   config.vm.provision "shell", inline: $centos_script
   config.vm.provider "virtualbox" do |v|
       v.memory = 4048
       v.cpus = 2
   end
end
