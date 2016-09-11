# -*- mode: ruby -*-
# vi: set ft=ruby :

libvirt_host = "host"
base_name = "devstack"
id_range = (111..120)
box = "fedora/24-cloud-base"
memory = 8192
cpus = 1
storage_pool_name = "volumes"
bridge_dev = "bridge"
ansible_playbook = "ansible/playbook.yaml"
bastion_user = "user"
bastion_host = "host"

Vagrant.configure(2) do |config|


  config.vm.box = box

  config.vm.synced_folder '~/vagrant/common', '/vagrant_common', type: "rsync"
  config.ssh.insert_key = false
  config.ssh.username = "root"

  config.vm.provider :libvirt do |libvirt|
    libvirt.host = libvirt_host
    libvirt.connect_via_ssh = true
    libvirt.username = "root"
    # libvirt.password = "secrete"
    libvirt.memory = memory
    libvirt.cpus = cpus

    if defined? storage_pool_name
      libvirt.storage_pool_name = storage_pool_name
    end

  end

  machines = []

  first = true
  id_range.each { |id|
    machine_id = "#{base_name}-#{id}"
    public_ip = "10.0.0.#{id}"
    private_ip = "10.1.#{id}.2"
    machines.push(machine_id)
    config.vm.define machine_id, primary: first do |vm_obj|
      first = false
      vm_obj.vm.hostname = machine_id
      vm_obj.vm.network :public_network,
        :type => "bridge",
        :dev => bridge_dev,
        :ip => public_ip
      vm_obj.vm.network :private_network,
        :ip => private_ip
    end
  }
  ssh_args = "-o ProxyCommand=\"ssh -vvv -W %h:%p #{bastion_user}@#{bastion_host}\""

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "vvvv"
    ansible.playbook = ansible_playbook
    ansible.groups = {
      "devstack" => machines
    }
    ansible.raw_ssh_args = ssh_args
    ansible.force_remote_user = true
  end

  # config.vm.provision "shell" do |s|
  #   s.inline = "sudo -u stack /vagrant_common/guest_scripts/devstack.sh"
  #   s.args = "/vagrant/local.conf"
  # end
end
