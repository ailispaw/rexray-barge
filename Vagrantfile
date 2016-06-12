# A dummy plugin for Barge to set hostname and network correctly at the very first `vagrant up`
module VagrantPlugins
  module GuestLinux
    class Plugin < Vagrant.plugin("2")
      guest_capability("linux", "change_host_name") { Cap::ChangeHostName }
      guest_capability("linux", "configure_networks") { Cap::ConfigureNetworks }
    end
  end
end

require "yaml"
rexray_config  = YAML.load_file("assets/config.yml")
volumePath     = File.expand_path "#{rexray_config['virtualbox']['volumePath']}"
controllerName = "#{rexray_config['virtualbox']['controllerName']}"

Vagrant.configure(2) do |config|
  config.vm.define "rexray-barge"

  config.vm.box = "ailispaw/barge"

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

  config.vm.provision :file, source: "assets", destination: '/tmp/assets'

  config.vm.provision :shell do |sh|
    sh.inline = <<-EOT
      # Use Docker v1.10.3 for REX-Ray
      /etc/init.d/docker restart v1.10.3

      # Install REX-Ray
      wget -qO- https://dl.bintray.com/emccode/rexray/stable/latest/rexray-Linux-x86_64.tar.gz | \
        tar zxf - -C /opt/bin

      # Setup REX-Ray
      sed -i 's|volumePath:.*$|volumePath: #{volumePath}|g' /tmp/assets/config.yml
      mkdir -p /etc/rexray
      mv /tmp/assets/config.yml /etc/rexray/config.yml

      # Patch eudev rules in Buildroot to create /dev/disk/by-id/* required by REX-Ray
      mkdir -p /etc/udev/rules.d
      mv /tmp/assets/60-persistent-storage.rules /etc/udev/rules.d/60-persistent-storage.rules

      rm -rf /tmp/assets
    EOT
  end

  config.vm.provision :shell, run: "always" do |sh|
    sh.inline = <<-EOT
      # Install eudev on every boot, because it's not persistent 
      sudo pkg install eudev

      # Restart udev
      udevadm control --stop-exec-queue || true
      killall udevd || true
      udevd --daemon --resolve-names=never
      udevadm trigger --action=add --property-match=DEVTYPE=disk
      udevadm settle

      # Restart rexray
      rexray stop || true
      rexray start
    EOT
  end
end