Vagrant.configure("2") do |config|
  config.vm.define :servidor do |servidor|
    servidor.vm.box = "generic/ubuntu2004"
    servidor.vm.hostname = "servidor1"
    servidor.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 2
      libvirt.memory = 2048
      libvirt.uri = 'qemu:///system'
    end
  end
  config.vm.define :host1 do |host1|
    host1.vm.box = "generic/ubuntu2004"
    host1.vm.hostname = "host1"
    host1.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 1
      libvirt.memory = 1024
      libvirt.uri = 'qemu:///system'
    end
  end
  config.vm.define :host2 do |host2|
    host2.vm.box = "generic/ubuntu2004"
    host2.vm.hostname = "host2"
    host2.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 1
      libvirt.memory = 512
      libvirt.uri = 'qemu:///system'
    end
  end
  config.vm.define :host3 do |host3|
    host3.vm.box = "generic/ubuntu2004"
    host3.vm.hostname = "host3"
    host3.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 1
      libvirt.memory = 512
      libvirt.uri = 'qemu:///system'
    end
  end
end
