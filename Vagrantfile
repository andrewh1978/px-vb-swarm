vb_dir = "#{ENV['HOME']}/VirtualBox\ VMs"
nodes = 3
disk_size = 20480
name = "px-test-cluster"
version = "2.0"

open("hosts", "w") do |f|
  f << "192.168.99.99 master\n"
  (1..nodes).each do |n|
    f << "192.168.99.10#{n} node#{n}\n"
  end
end

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = true
  config.vm.provision "shell", inline: <<-SHELL
    setenforce 0
    sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
    swapoff -a
    rm -f /swapfile
    rpm -e linux-firmware
    sed -i /swap/d /etc/fstab
    sed -i s/enabled=1/enabled=0/ /etc/yum/pluginconf.d/fastestmirror.conf
    mkdir -p /root/.ssh /etc/docker
    cp /vagrant/hosts /etc
    cp /vagrant/CentOS-Base.repo /etc/yum.repos.d
    cp /vagrant/id_rsa /root/.ssh
    cp /vagrant/id_rsa.pub /root/.ssh/authorized_keys
    cp /vagrant/docker /etc/sysconfig
    chmod 600 /root/.ssh/id_rsa
    echo '{"insecure-registries":["registry:5000"]}' >/etc/docker/daemon.json
  SHELL

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
  end

  config.vm.define "master" do |master|
    master.vm.hostname = "master"
    master.vm.network "private_network", ip: "192.168.99.99", virtualbox__intnet: true
    master.vm.provider :virtualbox do |vb| 
      vb.customize ["modifyvm", :id, "--name", "master"]
      vb.memory = 2048
    end
    master.vm.provision "shell", inline: <<-SHELL
      ( yum install -y docker
        systemctl start docker
        docker run -p 5000:5000 -d --restart=always --name registry -e REGISTRY_PROXY_REMOTEURL=http://registry-1.docker.io -v /opt/shared/docker_registry_cache:/var/lib/registry registry:2
        systemctl enable docker
        systemctl restart docker
        docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 4001:4001 -p 2380:2380 -p 2379:2379 --name etcd quay.io/coreos/etcd:v2.3.8 -name etcd0 -advertise-client-urls http://192.168.99.99:2379,http://192.168.99.99:4001 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 -initial-advertise-peer-urls http://192.168.99.99:2380 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster-1 -initial-cluster etcd0=http://192.168.99.99:2380 -initial-cluster-state new
        docker swarm init --advertise-addr 192.168.99.99
        echo End
      ) &>/var/log/vagrant.bootstrap &
    SHELL
  end

  (1..nodes).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      node.vm.network "private_network", ip: "192.168.99.10#{i}", virtualbox__intnet: true
      if i === 1
        node.vm.network "forwarded_port", guest: 32678, host: 32678
      end
      node.vm.provider "virtualbox" do |vb| 
        vb.memory = 3072
        vb.customize ["modifyvm", :id, "--name", "node#{i}"]
        if File.exist?("#{vb_dir}/disk#{i}.vdi")
          vb.customize ['closemedium', "#{vb_dir}/disk#{i}.vdi", "--delete"]
        end
        vb.customize ['createmedium', 'disk', '--filename', "#{vb_dir}/disk#{i}.vdi", '--size', disk_size]
        vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', "#{vb_dir}/disk#{i}.vdi"]
      end
      node.vm.provision "shell", inline: <<-SHELL
        ( yum install -y docker
          systemctl enable docker
          systemctl start docker
          latest_stable=$(curl -fsSL "https://install.portworx.com/#{version}/?type=dock&stork=false" | awk '/image: / {print $2}')
          while : ; do
            token=$(ssh -oConnectTimeout=1 -oStrictHostKeyChecking=no master docker swarm join-token worker -q)
            [ $? -eq 0 ] && break
            sleep 5
          done
          docker swarm join --token $token master:2377
          if [ $HOSTNAME != node1 ]; then
            while : ; do
              ssh -oConnectTimeout=1 -oStrictHostKeyChecking=no node1 docker images | grep -q px-enterprise
              [ $? -eq 0 ] && break
              echo waiting for px-enterprise to be cached
              sleep 1
            done
          fi
          docker run --entrypoint /runc-entry-point.sh --rm -i --privileged=true -v /opt/pwx:/opt/pwx -v /etc/pwx:/etc/pwx $latest_stable
          /opt/pwx/bin/px-runc install -c #{name} -k etcd://master:2379 -s /dev/sdb -m eth1 -d eth1
          systemctl daemon-reload
          systemctl enable portworx
          systemctl start portworx
          echo End
        ) &>/var/log/vagrant.bootstrap &
      SHELL
    end
  end

end
