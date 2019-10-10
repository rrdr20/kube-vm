DOMAIN_NAME = "dwkube.net"

# Defines the specifications for the nodes being deployed.
nodes = [
    # Deploy a server for running services such as ansible for configuration deployment.
    {:base_name => "srv", 
     :num_vms => "1", 
     :box => "ubuntu/bionic64", 
     :cpu => "1",
     :ram => "1024",
     :networks => [{:name => "kube_net01", :base_ip => "192.100.100.10"}]},
    
    # Deploy Load Balancer servers for the Kubernetes cluster.
    {:base_name => "lbs", 
     :num_vms => "2", 
     :box => "ubuntu/bionic64", 
     :cpu => "1",
     :ram => "1024",
     :networks => [{:name => "kube_net01", :base_ip => "192.168.100.20"}]},
    
    # Deploy ETCD servers for the Kubernetes cluster.
    {:base_name => "etc", 
     :num_vms => "3", 
     :box => "ubuntu/bionic64", 
     :cpu => "1",
     :ram => "2048",
     :networks => [{:name => "kube_net01", :base_ip => "192.168.100.30"}]},
    
    # Deploy Master servers for the Kubernetes cluster.
    {:base_name => "k8m", 
     :num_vms => "3", 
     :box => "ubuntu/bionic64", 
     :cpu => "2",
     :ram => "4096",
     :networks => [{:name => "kube_net01", :base_ip => "192.168.100.40"}]},
    
    # Deploy Worker servers for the Kubernetes cluster.
    {:base_name => "k8w", 
     :num_vms => "5", 
     :box => "ubuntu/bionic64", 
     :cpu => "2",
     :ram => "4096",
     :networks => [{:name => "kube_net01", :base_ip => "192.168.100.50"}]}
]

Vagrant.configure("2") do |config|
    nodes.each do |node|
        (1..node[:num_vms].to_i).each do |srv_count|
            node_name = node[:base_name] + "%02d" % srv_count

	    config.vm.define node_name do |nodecfg|
	        nodecfg.vm.box = node[:box]
            nodecfg.vm.hostname = node_name + ".#{DOMAIN_NAME}"

            # Only the server needs a synced folder
            unless node_name.include?("srv01")
                nodecfg.vm.synced_folder ".", "/vagrant",
                                         ShareFolderEnableSymlinksCreate: false,
                                         disabled: true
            end

            # Some of the servers will have more than one network interface defined
            node[:networks].each do |nodeint|
                int_ip = nodeint[:base_ip][0..-2] + "#{srv_count}"
                nodecfg.vm.network "private_network",
                                   ip: int_ip,
                                   nic_type: "82540EM",
                                   virtualbox__intnet: nodeint[:name]
            end

            nodecfg.vm.provider "virtualbox" do |vb|
                vb.cpus = node[:cpu]
                vb.memory = node[:ram]

                # Disable the audio controller, not needed
                vb.customize ["modifyvm", :id, "--audio", "none"]
            end

            nodecfg.ssh.insert_key = false
            nodecfg.ssh.private_key_path = ["keys/id_rsa", "~/.vagrant.d/insecure_private_key"]
            if node_name.include?("srv01")
                nodecfg.vm.provision :shell, inline: <<-EOC
                sudo systemctl stop systemd-resolved.service
                sudo systemctl disable systemd-resolved.service
                sudo rm -f /etc/resolv.conf
                sudo echo "nameserver 8.8.8.8" > /etc/resolv.conf
                sudo apt-get update
                sudo apt-get install dnsmasq
                sudo cp /vagrant/dnsmasq.conf /etc/dnsmasq.conf
                sudo chown root:root /etc/dnsmasq.conf
                sudo chmod 644 /etc/dnsmasq.conf
                sudo systemctl enable dnsmasq.service
                sudo systemctl restart dnsmasq.service
                EOC
            end
            nodecfg.vm.provision :file,
                source: "keys/id_rsa",
                destination: "/home/vagrant/.ssh/id_rsa"
            nodecfg.vm.provision :file,
                source: "keys/id_rsa.pub",
                destination: "/home/vagrant/.ssh/id_rsa.pub"
            nodecfg.vm.provision :file,
                source: "keys/id_rsa.pub",
                destination: "/home/vagrant/.ssh/authorized_keys"
            nodecfg.vm.provision :shell, inline: <<-EOC
                chmod 600 /home/vagrant/.ssh/id_rsa
                chmod 644 /home/vagrant/.ssh/id_rsa.pub
                sudo rm -f /etc/resolv.conf
                sudo echo "search dwkube.net" >> /etc/resolv.conf
                sudo echo "nameserver 192.168.100.11" >> /etc/resolv.conf
                sudo chmod 777 /etc/resolv.conf
                EOC
            end
        end
    end
end

