These instructions are a combination of "Boot an AWS instance" and "Install BOSH via its chef_deployer"

**NOTE: This instructions are for review only. I'll write them up as a formal blog post soon. Thanks for your help!**


* Create a VM
* Run prepare_instance.sh inside instance
* Use chef_deployer to setup the VM as BOSH

## Setup

Install fog, \~/.fog credentials (for AWS), and \~/.ssh/id_rsa(.pub) keys

Install fog

```
gem install fog
```

Example \~/.fog credentials:

```
 :default:
  :aws_access_key_id:     PERSONAL_ACCESS_KEY
  :aws_secret_access_key: PERSONAL_SECRET
```
To create id_rsa keys:

```
$ ssh-keygen
```

## Boot instance

From Wesley's [fog blog post|http://www.engineyard.com/blog/2011/spinning-up-cloud-compute-instances/], boot a vanilla Ubunutu 64-bit image:

```
$ fog
  Welcome to fog interactive!
  :default provides AWS and VirtualBox
connection = Fog::Compute.new({
  :provider => 'AWS', 
  :region => 'us-east-1'
})
server = connection.servers.bootstrap({
  :public_key_path => '~/.ssh/id_rsa.pub',
  :private_key_path => '~/.ssh/id_rsa',
  :flavor_id => 'm1.large', # 64 bit, normal large
  :username => 'ubuntu'
})
```

Check that SSH key credentials are setup. The following should return "ubuntu", and shouldn't timeout.

```
server.ssh "whoami"
```

Now create an elastic IP and associate it with the instance. (I did this via the console).

```
address = connection.addresses.create
address.server = server
server.reload
address.public_ip
"10.2.3.4"
server.dns_name
"ec2-10-2-3-4.compute-1.amazonaws.com"
```

**The public DNS name will be used in the remainder of the tutorials to reference the BOSH VM.**

## Firewall/Security Group

Go to the [AWS console](https://console.aws.amazon.com/ec2/home?region=us-east-1#s=SecurityGroups) and set your Security Group "Inbound" to include the 25555 port: (the default BOSH director port)

![security groups](https://img.skitch.com/20120414-m9g6ndg3gfjs7kdqhbp2y9a6y.png)

## Install Cloud Foundry

These commands below can take a long time. If it terminates early, re-run it until completion.

Alternately, run it inside screen or tmux so you don't have to fear early termination:

```
$ ssh ubuntu@107.21.120.243
# sudo apt-get install screen -y
# screen
sudo apt-get update
sudo apt-get install git-core build-essential libsqlite3-dev curl \
libmysqlclient-dev libxml2-dev libxslt-dev libpq-dev -y

git clone git://github.com/sstephenson/rbenv.git .rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.2-p290.tar.gz
tar xvfz ruby-1.9.2-p290.tar.gz
cd ruby-1.9.2-p290
./configure --prefix=$HOME/.rbenv/versions/1.9.2-p290
make
make install

cd
source ~/.bashrc
rbenv global 1.9.2-p290
gem update --system
gem install bundler rake
rbenv rehash

   
git clone https://github.com/cloudfoundry/bosh.git

sudo groupadd vcap 
sudo useradd vcap -m -g vcap

cd bosh/release/template/instance
sudo ./prepare_instance.sh


```

Create a `/path/to/your/bosh/release/project`. Yeah, that's a bit magical at the moment. FIXME

From another terminal on your local machine:

```
git clone https://github.com/cloudfoundry/bosh.git
cd bosh/chef_deployer
bundle
ruby bin/chef_deployer deploy /path/to/your/bosh/release/project
...lots of chef...
```

We can now connect to our MicroBOSH!

```
$ bosh target ec2-23-23-203-54.compute-1.amazonaws.com:25555
Target set to 'yourboshname (http://ec2-23-23-203-54.compute-1.amazonaws.com:25555) Ver: 0.4 (1e5bed5c)'
Your username: admin
Enter password: *****
Logged in as 'admin'
```

Username/password is admin/admin

Yay!