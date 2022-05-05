# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

props = YAML.load_file('vagrant-configuration.yaml')

vagrant_box = props["vm"]["box"]
memory = props["vm"]["memory"]
maxmemory = props["vm"]["maxmemory"]
cpus = props["vm"]["cpus"]

host_name = props["vm"]["network"]["hostname"]
static_ip = props["vm"]["network"]["ip"]
sub_domains = props["vm"]["network"]["services"].map! {|prefix| prefix + "." + host_name};
hostmanager_enabled = props["vm"]["network"]["hostmanager"]["enabled"]


docker_compose_version = props["docker"]["compose"]["version"]
docker_image_version = props["docker"]["image"]["version"]
docker_compose_files = props["docker"]["compose"]["files"]

compose_env = Hash.new
if File.file?(".env")
  lines = File.read(".env").split("\n")
  lines.each do |line|
    line.strip!
    unless line.start_with?("#") || line.empty?
      split = line.split("=")
      compose_env[split[0]] = split[1]
    end
  end
end

compose_env["EXTERNAL_HOSTNAME"] = host_name

# see https://stackoverflow.com/questions/42230536/docker-compose-up-times-out-with-unixhttpconnectionpool
# see https://www.codegrepper.com/code-examples/whatever/UnixHTTPConnectionPool%28host%3D%27localhost%27%2C+port%3DNone%29%3A+Read+timed+out.+%28read+timeout%3D60
compose_env["COMPOSE_HTTP_TIMEOUT"] = "400"
compose_env["DOCKER_CLIENT_TIMEOUT"] = "400"

Vagrant.configure("2") do |config|

  config.vm.box = vagrant_box
  
  if static_ip.nil? 
    config.vm.network "private_network", type: "dhcp"
  else 
    config.vm.network "private_network", ip: static_ip
  end

  config.hostmanager.enabled = hostmanager_enabled
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = false 
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  
  config.vm.provider "virtualbox" do |vb|
    vb.name = "docker-stack"
    vb.memory = memory
    vb.cpus = cpus
  end

  # we need a different box to use it
  config.vm.provider "hyperv" do |h|
    h.vmname = "docker-stack"
    h.memory = memory
    h.maxmemory = maxmemory
    h.cpus = cpus
  end
  
  config.vagrant.plugins = [
    # require plugin https://github.com/leighmcculloch/vagrant-docker-compose
    "vagrant-docker-compose",
    # https://github.com/devopsgroup-io/vagrant-hostmanager
    "vagrant-hostmanager"
    ] 
  

  config.vm.provision :docker

  docker_compose_files.each do |entry|
    project = entry["project"]
    location = entry["location"].to_s

    config.vm.provision project,  type: "docker_compose", rebuild: false, 
      compose_version: docker_compose_version, 
      project_name: project,
      yml: [ location ], 
      command_options: { rm: "-f", up: "-d --timeout 400"},
      env: compose_env        

    config.vm.provision project + "_echo", type: "shell", inline: "echo " + location
    config.vm.provision project + "_mem", type: "shell", inline: "free -m"
  end

  config.vm.hostname = host_name
  config.hostmanager.aliases = sub_domains

  config.trigger.after :provision do |trigger|
    trigger.ignore = [:destroy, :halt, :suspend]
    trigger.name = "Message"
    trigger.info = "Please visit http://" + host_name
  end

end
