# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'tsort'
require 'yaml'

$config_filename = 'config.yaml'

class ConfigurationError < StandardError
end

def deep_copy(obj)
  return Marshal.load(Marshal.dump(obj))
end

def topological_sort_roles(roles)
  each_role = lambda { |&b|
    roles.each_key(&b)
  }

  each_role_child = lambda { |node, &b|
    unless roles.has_key?(node)
      raise ConfigurationError.new, "role \"#{node}\" could not be found"
    end
    roles[node].fetch("roles", []).each(&b)
  }
  return TSort.tsort(each_role, each_role_child)
end

class VirtualMachine
  attr_accessor :name, :ansible
  def initialize(name, config)
    @config = apply_args(config)
    @name = name
    @ansible = {
      "groups" => [],
      "host_vars" => {},
    }
  end

  def get_config(key)
    return @config[key]
  end

  def create(vm_obj, ansible_config)
    create_meta(vm_obj)
    create_provider(vm_obj)
    create_networks(vm_obj)
    create_provision(vm_obj, ansible_config)
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

  def create_provision(vm_obj, ansible_config)
    ansible_groups = get_config("ansible_groups")
    if ansible_groups.is_a?(Array)
      @ansible["groups"] = ansible_groups
    else
      @ansible["groups"] = [ansible_groups, ]
    end

    v = get_config("ansible_host_vars")
    @ansible["host_vars"] = get_config("ansible_host_vars")
    if @ansible["host_vars"] == nil
      @ansible["host_vars"] = {}
    end
    ansible_config.add_machine(@name, @ansible)
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

class Config
  attr_accessor :machines, :roles, :global
  def initialize(config_filename)
    @config_filename = config_filename
    @config = {}
    @roles = {}
    @machines = {}
    @global = {}
  end

  def read_config()
    begin
      @config = YAML.load_file(@config_filename)
    rescue
      raise ConfigurationError.new "unable to parse #{@config_filename}"
    end
    unless @config.has_key?("machines")
        raise ConfigurationError.new, "unable to find machines subsection in config"
    end
  end

  def parse_config()
    if @config.has_key?("global")
      @global = @config["global"]
    end
    if @config.has_key?("roles")
      parse_roles(@config["roles"])
    end
    if @config.has_key?("machines")
      parse_machines(@config["machines"])
    end
  end

  def apply_sub_roles(object)
    res = {}
    sub_roles = object.fetch("roles", [])
    sub_roles.each { |sub_role|
      res.update(@roles[sub_role])
    }
    res.update(object)
    res.delete("roles")
    return res
  end

  def parse_role(role)
    return apply_sub_roles(role)
  end

  def parse_roles(roles)
    roles_by_order = topological_sort_roles(roles)
    roles_by_order.each { |role_key|
      role_value = roles[role_key]
      @roles[role_key] = parse_role(role_value)
    }
  end

  def expand_range(range)
    start = range.delete("~start")
    count = range.delete("~count")
    expanded = {}
    for idx in Range.new(start, start + count)
      fmt_map = { :item => idx }
      range.each { |k, v|
        expanded_v = deep_copy(v)
        k = k % fmt_map
        unless expanded_v.has_key?("args")
          expanded_v["args"] = {}
        end
        expanded_v["args"]["item"] = idx
        expanded[k] = expanded_v
      }
    end
    return expanded
  end

  def expand_ranges(machines)
    ranges = machines.select { |k, v|
      k.start_with?("~range-")
    }

    machines.delete_if { |k, v|
      k.start_with?("~range-")
    }

    ranges.each { |range_key, range_value|
      expanded_range = expand_range(range_value)
      expanded_range.each { |machine_key, machine_value|
        machines[machine_key] = machine_value
      }
    }
  end

  def parse_machine(machine_name, machine)
    parsed_machine = apply_sub_roles(machine)
    return VirtualMachine.new(machine_name, parsed_machine)
  end

  def parse_machines(machines)
    expand_ranges(machines)
    machines.each{ |machine_name, machine|
      @machines[machine_name] = parse_machine(machine_name, machine)
    }
  end
end

class AnsibleConfig
  def initialize(config)
    @playbook = config.global["ansible-playbook"]
    @verbose = config.global.fetch("ansible-verbose", "vvvv")
    @bastion_user = config.global["bastion_user"]
    @bastion_host = config.global["bastion_host"]
    @groups = {}
    @host_vars = {}
  end

  def add_machine(vm_name, vm_ansible)
    vm_ansible.fetch("groups", []).each{ |group|
      if @groups.has_key?(group)
        @groups[group].push(vm_name)
      else
        @groups[group] = [vm_name, ]
      end
    }
    @host_vars[vm_name] = vm_ansible.fetch("host_vars", {})
  end

  def set(ansible)
    ansible.verbose = @verbose
    ansible.playbook = @playbook
    ansible.groups = @groups
    ansible.host_vars = @host_vars
    if @bastion_user.is_a?(String) and @bastion_host.is_a?(String)
      ansible.raw_ssh_args = "-o ProxyCommand=\"ssh -vvv -W %h:%p #{@bastion_user}@#{@bastion_host}\""
    end
    ansible.force_remote_user = true
  end
end

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
    mapping = obj.map { |k, v| [k, apply_machine_args_(args, v)] }.to_h
    return mapping
  end
  return obj
end

def apply_args(config)
  args = config.delete("args")
  if args == nil
    return config
  end
  args = args.each_with_object({}){|(k,v), h| h[k.to_sym] = v}
  applied = apply_machine_args_(args, config)
  return applied
end


Vagrant.configure(2) do |config|

  config.ssh.insert_key = false
  config.ssh.username = "root"

  c = Config.new($config_filename)
  c.read_config()
  c.parse_config()

  ansible_config = AnsibleConfig.new(c)

  c.machines.each_value { |vm|
    config.vm.define vm.name() do |vm_obj|
      vm.create(vm_obj, ansible_config)
    end
  }

  config.vm.provision "ansible" do |ansible|
    ansible_config.set(ansible)
  end

end
