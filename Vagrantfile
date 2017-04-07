# encoding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :

CHEF_WORKSTATION_SCRIPT = <<EOF.freeze
apt-get update
apt-get install -y curl git tree vim

# ensure the time is up to date
echo "Synchronizing time..."
apt-get -y install ntp
service ntp stop
ntpdate -s time.nist.gov
service ntp start

curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -P chefdk -c stable -v 1.2.22
EOF

CHEF_SERVER_SCRIPT = <<EOF.freeze
apt-get update
apt-get -y install curl

# ensure the time is up to date
echo "Synchronizing time..."
apt-get -y install ntp
service ntp stop
ntpdate -s time.nist.gov
service ntp start

# download the Chef server package
echo "Downloading the Chef server package..."
if [ ! -f /downloads/chef-server-core_12.14.0-1_amd64.deb ]; then
  wget -nv -P /downloads https://packages.chef.io/files/stable/chef-server/12.14.0/ubuntu/14.04/chef-server-core_12.14.0-1_amd64.deb
fi

# install the package
echo "Installing Chef server..."
sudo dpkg -i /downloads/chef-server-core_12.14.0-1_amd64.deb

# reconfigure and restart services
echo "Reconfiguring Chef server..."
sudo chef-server-ctl reconfigure
echo "Restarting Chef server..."
sudo chef-server-ctl restart

# wait for services to be fully available
echo "Waiting for services..."
until (curl -D - http://localhost:8000/_status) | grep "200 OK"; do sleep 15s; done
while (curl http://localhost:8000/_status) | grep "fail"; do sleep 15s; done

# create admin user
echo "Creating a user and organization..."
sudo chef-server-ctl user-create admin Bob Admin admin@4thcoffee.com insecurepassword --filename admin.pem
sudo chef-server-ctl org-create 4thcoffee "Fourth Coffee, Inc." --association_user admin --filename 4thcoffee-validator.pem

# copy admin RSA private key to share
echo "Copying admin key to /vagrant/secrets..."
mkdir -p /vagrant/secrets
cp -f /home/vagrant/admin.pem /vagrant/secrets

echo "Your Chef server is ready!"
EOF

CHEF_NODE_SCRIPT = <<EOF.freeze
echo "Preparing node..."

# ensure the time is up to date
apt-get update
apt-get -y install ntp
service ntp stop
ntpdate -s time.nist.gov
service ntp start

echo "10.1.1.33 chef-server.test" | tee -a /etc/hosts
EOF

def set_hostname(server)
  server.vm.provision 'shell', inline: "hostname #{server.vm.hostname}"
end

Vagrant.configure(2) do |config|
  config.vm.define 'chef-workstation' do |cw|
    cw.vm.box = "bento/ubuntu-14.04"
    cw.vm.box_version = '2.3.4'
    cw.vm.hostname = 'chef-workstation.test'
    cw.vm.network "private_network", ip: "10.1.1.32"
    cw.vm.provision "shell", inline: CHEF_WORKSTATION_SCRIPT.dup
    set_hostname(cw)
  end

  config.vm.define 'chef-server' do |cs|
    cs.vm.box = 'bento/ubuntu-14.04'
    cs.vm.box_version = '2.3.4'
    cs.vm.hostname = 'chef-server.test'
    cs.vm.network 'private_network', ip: '10.1.1.33'
    cs.vm.provision 'shell', inline: CHEF_SERVER_SCRIPT.dup
    set_hostname(cs)

    cs.vm.provider 'virtualbox' do |v|
      v.memory = 2048
      v.cpus = 2
    end
  end

  config.vm.define 'chef-node-1' do |cn1|
    cn1.vm.box = 'bento/ubuntu-14.04'
    cn1.vm.box_version = '2.3.4'
    cn1.vm.hostname = 'chef-node-1.test'
    cn1.vm.network 'private_network', ip: '10.1.1.34'
    cn1.vm.provision :shell, inline: CHEF_NODE_SCRIPT.dup
    set_hostname(cn1)
  end

  # config.vm.define 'chef-node-2' do |cn2|
  #   cn2.vm.box = 'bento/ubuntu-14.04'
  #   cn2.vm.box_version = '2.3.4'
  #   cn2.vm.hostname = 'chef-node-2.test'
  #   cn2.vm.network 'private_network', ip: '10.1.1.35'
  #   cn2.vm.provision :shell, inline: CHEF_NODE_SCRIPT.dup
  #   set_hostname(cn2)
  # end
end
