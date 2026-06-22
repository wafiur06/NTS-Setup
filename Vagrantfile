Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # NTS Server
  config.vm.define "nts-server" do |server|
    server.vm.hostname = "nts-server.local"
    server.vm.network "private_network", ip: "192.168.56.10"
    server.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
  end

  # NTS Client
  config.vm.define "nts-client" do |client|
    client.vm.hostname = "nts-client.local"
    client.vm.network "private_network", ip: "192.168.56.20"
    client.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
  end
end
