# Networking options
cni_plugin = 'flannel'    # options are weave or flannel
flannel_backend = 'ipsec' # options are vxlan or ipsec

# If you're using ipsec, you need to create a Private Shared Key.  This is not production safe,
# it's only suitable for experimentation.
# To generate a new PSK, you could use: dd if=/dev/urandom count=48 bs=1 status=none | xxd -p -c 48
flannel_ipsec_psk = 'ee9b0c24c5d015f4ed23e84fb1473314a8baa8bee65fcefd659981d59423c6ee718eee681f579cccf18fd5d7f759f736'

# Set the image to 'master:5000/coreos/flannel' if using ipsec.  You'll also have to build it, see README.
# Or use 'quay.io/coreos/flannel:v0.10.0-amd64' to get it from upstream, if you only want vxlan.
#flannel_docker_image = 'quay.io/coreos/flannel:v0.10.0-amd64'
flannel_docker_image = 'master:5000/coreos/flannel'

# Versions
k8s_pkg_version = '1.11.3-00'    # this is the dpkg version for http://apt.kubernetes.io/

# Change these if you need to run multiple clusters
host_kube_port = 6444            # Port forward for Kubernetes API server
host_prom_port = 31000           # Port forward for Prometheus
node_domain = 'ikube'            # domain name of vm hosts

# You probably don't need to change anything below, but you can if you like.
network_prefix  = '10.0.0.'  # ip prefix of vm hosts, remaining octet provided by :id attribute in nodes, below:
nodes = [
  { :hostname => 'master', :id => '10', :memory => 1500 },  # master needs 1.5GB
  { :hostname => 'node1',  :id => '11', :memory => 500 },   # nodes need at least 500MB, adjust for workload
  { :hostname => 'node2',  :id => '12', :memory => 500 },
  { :hostname => 'node3',  :id => '13', :memory => 500 },
]

# --------------------------------
# No user-serviceable parts below.
# --------------------------------
overlay_network = "10.244.0.0"
overlay_network_bits = "16"

$script = <<SCRIPT
apt-get -y update
apt-get -y install python-minimal python-apt
SCRIPT

Vagrant.configure("2") do |config|
  # Use a single ssh key for all hosts, makes it easier to ssh between them.
  config.ssh.insert_key = false
  # hostmanager enabled (installed via vagrant plugin install vagrant-hostmanager):
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = false
  config.hostmanager.manage_guest = true

  nodes.each do |node|
    config.vm.define node[:hostname] do |nodeconfig|
      nodeconfig.vm.box = "ubuntu/xenial64"
      nodeconfig.vm.hostname = node[:hostname]
      nodeconfig.vm.network :private_network, ip: network_prefix + node[:id], virtualbox__intnet: node_domain
      if node[:hostname] == 'master'
        nodeconfig.vm.network "forwarded_port", guest: 31000, host: host_prom_port
        nodeconfig.vm.network "forwarded_port", guest: 6443, host: host_kube_port
      end
      nodeconfig.vm.provider :virtualbox do |vb|
        # Use virtualbox linked clones to reduce disk usage
        vb.linked_clone = true
        vb.name = node[:hostname]+"."+node_domain
        vb.memory = node[:memory]
        vb.cpus = 1
        vb.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
        vb.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
        vb.customize ['modifyvm', :id, '--macaddress1', "5CA1AB1E00"+node[:id]]
        vb.customize ['modifyvm', :id, '--natnet1', "192.168/16"]
      end
      nodeconfig.vm.provision "file", source: "~/.vagrant.d/insecure_private_key", destination: "/home/vagrant/.ssh/id_rsa"

      nodeconfig.vm.provision "shell", inline: $script
      if node[:hostname] == nodes[-1][:hostname]
        nodeconfig.vm.provision "ansible" do |ansible|
          ansible.config_file = "ansible/ansible.cfg"
          # Disable default limit to connect to all the machines.  Add localhost so we can run some commands
          # locally.
          ansible.limit = "all,localhost"
          # Setup ssh-config file for future ansible use
          ansible.playbook = "ansible/site.yml"
          ansible.groups = {
            'all:vars' => {
              'flannel_docker_image': flannel_docker_image,
              'host_prom_port': host_prom_port,
              'host_kube_port': host_kube_port,
              'k8s_pkg_version': k8s_pkg_version,
              'cni_plugin': cni_plugin,
              'flannel_backend': flannel_backend,
              'flannel_ipsec_psk': flannel_ipsec_psk,
              'overlay_network': overlay_network,
              'overlay_network_bits': overlay_network_bits,
            },
            'masters' => ['master'],
            'nodes' => ['node1', 'node2', 'node3']
          }
        end
      end
    end
  end
end
