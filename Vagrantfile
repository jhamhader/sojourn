# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

begin
  $config = YAML.load_file('config.yaml')
rescue
  puts 'Error reading config.yaml'
  return
end

class ConfigurationError < StandardError
end

$initial_config = {
  "provider_host" => "localhost",
  "provider_username" => "root",
  "playbook" => "ansible/playbook.yaml",
  "box" => "fedora/24-cloud-base",
  "memory" => 8192,
  "cpus" => 1,
  "storage_pool_name" => nil,
  "bastion_user" => nil,
  "bastion_host" => nil,
  "public_networks": [],
  "private_networks": [],
}

$default_config = $config.fetch("default", {})

unless $config.has_key?("machines")
    raise ConfigurationError.new, "unable to find machines subsection in config"
end

$machine_config = $config["machines"]
def get_machine_config(machine, key)
  unless $machine_config.has_key?(machine)
    raise ConfigurationError.new, "unable to find #{machine} in config"
  end
  return $machine_config[machine].fetch(
    key,
    $default_config.fetch(key, $initial_config[key])
  )
end

$machines = $machine_config.keys

def apply_machine_args_(args, obj)
  if obj.is_a?(String)
    begin
      return obj % args
    rescue KeyError => e
      raise ConfigurationError.new, "unable to find arg: #{e.message}"
    end
  end
  if obj.is_a?(Array)
    return obj.map { |k| apply_machine_args_(args, k) }
  end
  if obj.is_a?(Hash)
    return obj.map { |k| apply_machine_args_(args, k[1]) }
  end
  return obj
end

def apply_machine_args(machine, obj)
  args = get_machine_config(machine, "args")
  args = args.each_with_object({}){|(k,v), h| h[k.to_sym] = v}
  if args == nil
    return obj
  end
  return apply_machine_args_(args, obj)
end

class VirtualMachine
  def initialize(machine_id)
    @machine_id = machine_id
  end

  def get_config_raw(key)
    return get_machine_config(@machine_id, key)
  end

  def apply_args(obj)
    apply_machine_args(@machine_id, obj)
  end

  def get_config(key)
    return apply_args(get_config_raw(key))
  end

  def create(vm_obj)
    create_meta(vm_obj)
    create_provider(vm_obj)
    create_networks(vm_obj)
    create_provision(vm_obj)
  end

  def create_meta(vm_obj)
    vm_obj.vm.hostname = @machine_id
    vm_obj.vm.box = get_config("box")
  end

  def create_provider(vm_obj)
    vm_obj.vm.provider :libvirt do |libvirt|
      libvirt.host = get_config("provider_host")
      libvirt.connect_via_ssh = true
      libvirt.username = get_config("provider_username")
      libvirt.memory = get_config("memory")
      libvirt.cpus = get_config("cpus")

      storage_pool_name = get_config("storage_pool_name")
      if storage_pool_name.instance_of?(String)
        libvirt.storage_pool_name = storage_pool_name
      end
    end
  end

  def create_provision(vm_obj)
    ansible_playbook = get_config("ansible_playbook")
    unless ansible_playbook.instance_of?(String)
      return
    end

    bastion_user = get_config("bastion_user")
    bastion_host = get_config("bastion_host")
    ssh_args = nil

    if bastion_user.is_a?(String) and bastion_host.is_a?(String)
      ssh_args = "-o ProxyCommand=\"ssh -vvv -W %h:%p #{bastion_user}@#{bastion_host}\""
    end

    vm_obj.vm.provision "ansible" do |ansible|
      ansible.verbose = "vvvv"
      ansible.playbook = ansible_playbook
      ansible.groups = {
        "devstack" => $machines
      }
      ansible.raw_ssh_args = ssh_args
      ansible.force_remote_user = true
    end
  end

  def create_networks_(vm_obj, type)
    networks = get_config(type)
    if networks.instance_of?(Hash)
      networks = [networks]
    end
    if networks.instance_of?(Array)
      networks.each { |network|
        vm_obj.vm.network type, network
      }
    end
  end

  def create_networks(vm_obj)
    create_networks_(vm_obj, :private_network)
    create_networks_(vm_obj, :public_network)
  end

end

Vagrant.configure(2) do |config|

  config.ssh.insert_key = false
  config.ssh.username = "root"

  $machines.each { |machine_id|
    config.vm.define machine_id do |vm_obj|
      vm = VirtualMachine.new(machine_id)
      vm.create(vm_obj)
    end
  }

end
