# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

# CoreOS doesn't support vboxsf annd guest-additions for virtualbox
# So we need to use NFS, and Vagrant NFS doesn't work without this
plugin_dependencies = [
  "vagrant-winnfsd",
  "vagrant-hostmanager"
]

needsRestart = false

# Install plugins if required
plugin_dependencies.each do |plugin_name|
  unless Vagrant.has_plugin? plugin_name
    system("vagrant plugin install #{plugin_name}")
    needsRestart = true
    puts "#{plugin_name} installed"
  end
end

# Restart vagrant if new plugins were installed
if needsRestart === true
  exec "vagrant #{ARGV.join(' ')}"
end

# Use old vb_xxx config variables when set
def vm_gui
  $vb_gui.nil? ? $vm_gui : $vb_gui
end

def vm_memory
  $vb_memory.nil? ? $vm_memory : $vb_memory
end

def vm_cpus
  $vb_cpus.nil? ? $vm_cpus : $vb_cpus
end

$vm_configs = [
  # Defaults for config options
  etcd_config: {
    num_instances: 1,
    instance_name_prefix: "etcd",
    enable_serial_logging: false,

    vm_gui: false,
    vm_memory: 512,
    vm_cpus: 1,
    vb_cpuexecutioncap: 80,

    user_home_path: "/home/core",
    forwarded_ports: [],
    shared_folders: [
      {
        host_path: "./",
        guest_path: "/vagrant"
      }
    ]
  },

  kube_master_config: {
    num_instances: 1,
    instance_name_prefix: "kube-master",
    enable_serial_logging: false,

    vm_gui: false,
    vm_memory: 2048,
    vm_cpus: 2,
    vb_cpuexecutioncap: 80,

    user_home_path: "/home/core",
    forwarded_ports: [],
    shared_folders: [
      {
        host_path: "./",
        guest_path: "/vagrant"
      }
    ]
  },

  kube_worker_config: {
    num_instances: 2,
    instance_name_prefix: "kube-worker",
    enable_serial_logging: false,

    vm_gui: false,
    vm_memory: 1024,
    vm_cpus: 2,
    vb_cpuexecutioncap: 80,

    user_home_path: "/home/core",
    forwarded_ports: [],
    shared_folders: [
      {
        host_path: "./",
        guest_path: "/vagrant"
      }
    ]
  }
]

Vagrant.configure("2") do |config|
  # always use Vagrants insecure key
  config.ssh.insert_key = true
  # forward ssh agent to easily ssh into the different machines
  config.ssh.forward_agent = false

  # Hostmanager
  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false

  config.vm.box = "jaskaranbir/coreos-ansible"
  config.vm.boot_timeout = 500

  config.vm.provider :virtualbox do |vbox|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    vbox.check_guest_additions = false
    vbox.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  # This keeps track of total number of instances in all VMs
  # It is dynamically incremented as the VM configs are iterated
  vm_num_instances_offset = 0

  # We need to know total number of instances so we run ansible
  # only once, at last instance.
  total_instances_count = 0
  $vm_configs.each do | vm_config |
    vm_config.each do |_, vc|
      total_instances_count += vc[:num_instances]
    end
  end

  # ================= VM-specific Configurations =================

  $vm_configs.each do |vm_config|
    vm_config.each do |vm_config_name, vc|
      (1..vc[:num_instances]).each do |i|
        config.vm.define vm_name = "%s-%02d" % [vc[:instance_name_prefix], i] do |config|
          vm_num_instances_offset += 1
          config.vm.hostname = vm_name

          # Serial Logging
          if vc[:enable_serial_logging]
            logdir = File.join(File.dirname(__FILE__), "log")
            FileUtils.mkdir_p(logdir)

            serialFile = File.join(logdir, "%s-%s-serial.txt" % [vm_name, vc[:instance_name_prefix]])
            FileUtils.touch(serialFile)

            config.vm.provider :virtualbox do |vb, override|
              vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
              vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
            end
          end

          # VM hardware resources configurations
          config.vm.provider :virtualbox do |vb|
            vb.gui = vc[:vm_gui]
            vb.memory = vc[:vm_memory]
            vb.cpus = vc[:vm_cpus]
            vb.customize [
              "modifyvm", :id,
              "--cpuexecutioncap", "#{vc[:vb_cpuexecutioncap]}"
            ]
          end

          ip = "172.17.8.#{vm_num_instances_offset + 100}"
          config.vm.network :private_network, ip: ip, auto_correct: true

          # Port Forwarding
          vc[:forwarded_ports].each do |port|
            config.vm.network :forwarded_port,
              host: port[:host_port],
              guest: port[:guest_port],
              auto_correct: true
          end

          # # Shared folders
          vc[:shared_folders].each_with_index do |share, i|
            config.vm.synced_folder share[:host_path], share[:guest_path],
                id: "core-share%02d" % vm_num_instances_offset,
                nfs: true,
                mount_options: ['nolock,vers=3,udp']
          end

          # Automatically set current-dir to /vagrant on vagrant ssh
          config.vm.provision :shell,
              inline: "echo 'cd /vagrant' >> #{vc[:user_home_path]}/.bashrc"

          # Ansible 2.6+ works only when SSH key is protected.
          # So we manually copy the SSH key and set its permissions.
          config.vm.provision :shell,
            privileged: true, inline: <<-EOF
              mkdir -p "#{vc[:user_home_path]}/.ssh"
              cp "/vagrant/.vagrant/machines/#{vm_name}/virtualbox/private_key" "#{vc[:user_home_path]}/.ssh/id_rsa"
              chmod 0400 "#{vc[:user_home_path]}/.ssh/id_rsa"
            EOF

          # Run Ansible provisioning when its last instance, so its only run once
          if vm_num_instances_offset === total_instances_count
            # Copy ansible directory to enable provisioning
            config.vm.provision :shell,
                inline: "mkdir -p -m777 /ansible",
                privileged: true
            config.vm.provision "file", source: "./", destination: "/ansible"
            # File-provisioner needs full permissions to copy files,
            # but ansible 2.6+ will not work unless parent dir is write-protected.
            config.vm.provision :shell,
                inline: "chmod 744 /ansible",
                privileged: true

            config.vm.provision :shell,
             inline: "cd /ansible" \
                     " && /opt/bin/active_python/bin/ansible-playbook" \
                       " kubernetes.yml -vv",
             privileged: true
          end
        end
      end
    end
  end
end
