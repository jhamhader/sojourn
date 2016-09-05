# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

begin
  $config = YAML.load_file('config.yaml')
rescue
  puts 'Error reading config.yaml'
  return
end

$default_config = {
  "host" => "localhost",
  "playbook" => "ansible/playbook.yaml",
  "box" => "fedora/24-cloud-base",
  "memory" => 8192,
  "cpus" => 1,
  "storage_pool_name" => nil,
  "bastion_user" => nil,
  "bastion_host" => nil,
  "basename" => "devstack",
}

def get_config(key)
  $config.fetch(key, $default_config[key])
end

libvirt_host = get_config("host")
ansible_playbook = get_config("playbook")
box = get_config("box")
basename = get_config("basename")
bastion_user = get_config("bastion_user")
bastion_host = get_config("bastion_host")
memory = get_config("memory")
cpus = get_config("cpus")
storage_pool_name = get_config("storage_pool_name")
id_range = (111..120)
bridge_dev = "bridge"

Vagrant.configure(2) do |config|

  config.vm.box = box
  config.vm.synced_folder '~/vagrant/common', '/vagrant_common', type: "rsync"
  config.ssh.insert_key = false
  config.ssh.username = "root"

  config.vm.provider :libvirt do |libvirt|
    libvirt.host = libvirt_host
    libvirt.connect_via_ssh = true
    libvirt.username = "root"
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

end
