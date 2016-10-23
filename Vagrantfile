# A dummy plugin for Barge to set hostname and network correctly at the very first `vagrant up`
module VagrantPlugins
  module GuestLinux
    class Plugin < Vagrant.plugin("2")
      guest_capability("linux", "change_host_name") { Cap::ChangeHostName }
      guest_capability("linux", "configure_networks") { Cap::ConfigureNetworks }
    end
  end
end

REXRAY_VERSION = "0.6.0"

require "yaml"
rexray_config  = YAML.load_file("assets/config.yml")
vbox_config    = rexray_config['libstorage']['server']['services']['virtualbox']
volumePath     = File.expand_path "#{vbox_config['virtualbox']['volumePath']}"
controllerName = "#{vbox_config['virtualbox']['controllerName']}"

Vagrant.configure(2) do |config|
  config.vm.define "rexray-barge"

  config.vm.box = "ailispaw/barge"
  config.vm.box_version = ">= 2.2.1"
  config.vm.base_mac = "auto"

  config.vm.provider :virtualbox do |vb|
    vb.customize ["storagectl", :id, "--name", "#{controllerName}", "--portcount", 30]
    Dir["#{volumePath}/*"].each do | file |
      vb.customize [
        "storageattach", :id,
        "--storagectl", "#{controllerName}",
        "--port", 2,
        "--device", 0,
        "--type", "hdd",
        "--medium", file,
      ]
      vb.customize [
        "storageattach", :id,
        "--storagectl", "#{controllerName}",
        "--port", 2,
        "--device", 0,
        "--type", "hdd",
        "--medium", "none",
      ]
    end
  end

  if Vagrant.has_plugin?("vagrant-triggers") then
    config.trigger.before [:up, :resume] do
      info "Start vboxwebsrv"
      run "vboxwebsrv -b -H 0.0.0.0 -A null -P /tmp/vboxwebsrv.pid"
    end

    config.trigger.after [:destroy, :suspend, :halt] do
      info "Stop vboxwebsrv"
      run <<-EOT
        sh -c "kill $(cat /tmp/vboxwebsrv.pid) && rm /tmp/vboxwebsrv.pid || true"
      EOT
    end
  end

  config.vm.provision :file, source: "assets", destination: "/tmp/"

  config.vm.provision :shell do |sh|
    sh.inline = <<-EOT
      # Install REX-Ray
      wget -qO- https://dl.bintray.com/emccode/rexray/stable/#{REXRAY_VERSION}/rexray-Linux-x86_64-#{REXRAY_VERSION}.tar.gz | \
        tar zxf - -C /opt/bin

      # Setup REX-Ray
      sed -i 's|volumePath:.*$|volumePath: #{volumePath}|g' /tmp/assets/config.yml
      mkdir -p /etc/rexray
      mv /tmp/assets/config.yml /etc/rexray/config.yml

      # Patch eudev rules in Buildroot to create /dev/disk/by-id/* required by REX-Ray
      mkdir -p /etc/udev/rules.d
      mv /tmp/assets/60-persistent-storage.rules /etc/udev/rules.d/60-persistent-storage.rules
    EOT
  end

  config.vm.provision :shell, run: "always" do |sh|
    sh.inline = <<-EOT
      # Install eudev on every boot, because it's not persistent
      pkg install eudev
      pkg install dmidecode

      # Restart udev
      udevadm control --stop-exec-queue || true
      killall udevd || true
      udevd --daemon --resolve-names=never
      udevadm trigger --action=add --property-match=DEVTYPE=disk
      udevadm settle

      # Restart rexray
      rexray service stop || true
      rexray service start
    EOT
  end
end
