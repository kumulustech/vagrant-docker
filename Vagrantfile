# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Require a recent version of vagrant otherwise some have reported errors setting host names on boxes
Vagrant.require_version ">= 1.9"


# ip configuration
$docker_ip = ENV['MASTER_IP'] || "192.168.56.10"
$kube_os = ENV['KUBERNETES_OS'] || "ubuntu"

# Determine whether vagrant should use nfs to sync folders
$use_nfs = ENV['KUBERNETES_VAGRANT_USE_NFS'] == 'true'
# Determine whether vagrant should use rsync to sync folders
$use_rsync = ENV['KUBERNETES_VAGRANT_USE_RSYNC'] == 'true'

# To override the vagrant provider, use (e.g.):
#   KUBERNETES_PROVIDER=vagrant VAGRANT_DEFAULT_PROVIDER=... .../cluster/kube-up.sh
# To override the box, use (e.g.):
#   KUBERNETES_PROVIDER=vagrant KUBERNETES_BOX_NAME=... .../cluster/kube-up.sh
# You can specify a box version:
#   KUBERNETES_PROVIDER=vagrant KUBERNETES_BOX_NAME=... KUBERNETES_BOX_VERSION=... .../cluster/kube-up.sh
# You can specify a box location:
#   KUBERNETES_PROVIDER=vagrant KUBERNETES_BOX_NAME=... KUBERNETES_BOX_URL=... .../cluster/kube-up.sh
# KUBERNETES_BOX_URL and KUBERNETES_BOX_VERSION will be ignored unless
# KUBERNETES_BOX_NAME is set

# Default OS platform to provider/box information
$kube_provider_boxes = {
  :virtualbox => {
    'ubuntu' => {
      :box_name => 'ubuntu/xenial64',
    }
  },
}

# Give access to all physical cpu cores
# Previously cargo-culted from here:
# http://www.stefanwrobel.com/how-to-make-vagrant-performance-not-suck
# Rewritten to actually determine the number of hardware cores instead of assuming
# that the host has hyperthreading enabled.
host = RbConfig::CONFIG['host_os']
if host =~ /darwin/
  $vm_cpus = `sysctl -n hw.physicalcpu`.to_i
elsif host =~ /linux/
  #This should work on most processors, however it will fail on ones without the core id field.
  #So far i have only seen this on a raspberry pi. which you probably don't want to run vagrant on anyhow...
  #But just in case we'll default to the result of nproc if we get 0 just to be safe.
  $vm_cpus = `cat /proc/cpuinfo | grep 'core id' | sort -u | wc -l`.to_i
  if $vm_cpus < 1
      $vm_cpus = `nproc`.to_i
  end
else # sorry Windows folks, I can't help you
  $vm_cpus = 2
end

# Give VM 1024MB of RAM by default
# In Fedora VM, tmpfs device is mapped to /tmp.  tmpfs is given 50% of RAM allocation.
# When doing Salt provisioning, we copy approximately 200MB of content in /tmp before anything else happens.
# This causes problems if anything else was in /tmp or the other directories that are bound to tmpfs device (i.e /run, etc.)
$vm_docker_mem = (ENV['KUBERNETES_MASTER_MEMORY'] || ENV['KUBERNETES_MEMORY'] || 2048).to_i

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  def setvmboxandurl(config, provider)
    if ENV['KUBERNETES_BOX_NAME'] then
      config.vm.box = ENV['KUBERNETES_BOX_NAME']

      if ENV['KUBERNETES_BOX_URL'] then
        config.vm.box_url = ENV['KUBERNETES_BOX_URL']
      end

      if ENV['KUBERNETES_BOX_VERSION'] then
        config.vm.box_version = ENV['KUBERNETES_BOX_VERSION']
      end
    else
      config.vm.box = $kube_provider_boxes[provider][$kube_os][:box_name]

      if $kube_provider_boxes[provider][$kube_os][:box_url] then
        config.vm.box_url = $kube_provider_boxes[provider][$kube_os][:box_url]
      end

      if $kube_provider_boxes[provider][$kube_os][:box_version] then
        config.vm.box_version = $kube_provider_boxes[provider][$kube_os][:box_version]
      end
    end
  end

  def customize_vm(config, vm_mem)

    if $use_nfs then
      config.vm.synced_folder ".", "/vagrant", nfs: true
    elsif $use_rsync then
      opts = {}
      if ENV['KUBERNETES_VAGRANT_RSYNC_ARGS'] then
        opts[:rsync__args] = ENV['KUBERNETES_VAGRANT_RSYNC_ARGS'].split(" ")
      end
      if ENV['KUBERNETES_VAGRANT_RSYNC_EXCLUDE'] then
        opts[:rsync__exclude] = ENV['KUBERNETES_VAGRANT_RSYNC_EXCLUDE'].split(" ")
      end
      config.vm.synced_folder ".", "/vagrant", opts
    end

    # Don't attempt to update Virtualbox Guest Additions (requires gcc)
    if Vagrant.has_plugin?("vagrant-vbguest") then
      config.vbguest.auto_update = false
    end
    # Finally, fall back to VirtualBox
    config.vm.provider :virtualbox do |v, override|
      setvmboxandurl(override, :virtualbox)
      v.memory = vm_mem # v.customize ["modifyvm", :id, "--memory", vm_mem]
      v.cpus = $vm_cpus # v.customize ["modifyvm", :id, "--cpus", $vm_cpus]

      # Use faster paravirtualized networking
      v.customize ["modifyvm", :id, "--nictype1", "virtio"]
      v.customize ["modifyvm", :id, "--nictype2", "virtio"]
    end
  end

  # master node
  config.vm.define "docker" do |c|
    customize_vm c, $vm_docker_mem
    config.vm.provision "shell", path: "docker-ubuntu.sh"
    c.vm.hostname = 'docker'
    c.vm.network "private_network", ip: "#{$docker_ip}"
  end
end
