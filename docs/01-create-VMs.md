# Prerequisites

## Vagrant and VirtualBox

You will need to install Vagrant and VirtualBox.  VirtualBox will run the VMs used in this tutorial and Vagrant will be used to create the VMs in a programmatic manner.

https://www.vagrantup.com/intro/getting-started/install.html

https://www.virtualbox.org/wiki/Downloads

## Download the VM image

We will run Ubuntu for this tutorial so the first thing to do is download the 18.04 box.  We'll be using the official ubuntu release.  In order to download the box, run:

```bash
vagrant box add "ubuntu/bionic64"
```

This will take some time depending upon your connection.  After a while you should see it finish:

```bash
==> k8s-load-balancer: Loading metadata for box 'ubuntu/bionic64'
    k8s-load-balancer: URL: https://vagrantcloud.com/ubuntu/bionic64
==> k8s-load-balancer: Adding box 'ubuntu/bionic64' (v20200206.0.0) for provider: virtualbox
    k8s-load-balancer: Downloading: https://vagrantcloud.com/ubuntu/boxes/bionic64/versions/20200206.0.0/providers/virtualbox.box
    k8s-load-balancer: Download redirected to host: cloud-images.ubuntu.com
==> k8s-load-balancer: Successfully added box 'ubuntu/bionic64' (v20200206.0.0) for 'virtualbox'!
``` 

Now that we have our image downloaded, we can start provisioning our VMs.  Note that if we didn't download the box, we would see the box downloaded the first time we try to run `vagrant up`.

## The Vagrantfile, creating our first box.

There are a lot of ways to write Vagrant files.  If you have never used Vagrant before you can find further documentation here: https://www.vagrantup.com/docs/

For our purposes we'll keep the file as simple as possible.  Our goal is to have 7 like hosts with:

* 2 CPUs
* 2 GB RAM
* 15 GB disk

We will have:

* 3 controller VMs
* 3 worker VMs
* 1 load balancer VM

Let's create the base `Vagrantfile` and have it create one VM so we can see how that looks.  Run:

```bash
{

cat > Vagrantfile <<EOF
API_VERSION = 2

box_image = "ubuntu/bionic64"

Vagrant.configure("2") do |config|
	config.vm.define "k8s-load-balancer" do |node|
		node.vm.box = box_image
		lb_vm_name = "k8s-load-balancer"
		node.vm.hostname = lb_vm_name
		node.vm.network :private_network, ip: "10.0.0.10"
		node.vm.provider "virtualbox" do |v|
			v.name = lb_vm_name
			v.memory = 2048
			v.cpus = 2
		end
	end
end
EOF

vagrant up

}
```

You will see some output that should look something like this:

```bash
Bringing machine 'k8s-load-balancer' up with 'virtualbox' provider...
==> k8s-load-balancer: Importing base box 'ubuntu/bionic64'...
==> k8s-load-balancer: Matching MAC address for NAT networking...
==> k8s-load-balancer: Checking if box 'ubuntu/bionic64' version '20200206.0.0' is up to date...
==> k8s-load-balancer: Setting the name of the VM: k8s-load-balancer
==> k8s-load-balancer: Clearing any previously set network interfaces...
==> k8s-load-balancer: Preparing network interfaces based on configuration...
    k8s-load-balancer: Adapter 1: nat
    k8s-load-balancer: Adapter 2: hostonly
==> k8s-load-balancer: Forwarding ports...
    k8s-load-balancer: 22 (guest) => 2222 (host) (adapter 1)
==> k8s-load-balancer: Running 'pre-boot' VM customizations...
==> k8s-load-balancer: Booting VM...
==> k8s-load-balancer: Waiting for machine to boot. This may take a few minutes...
    k8s-load-balancer: SSH address: 127.0.0.1:2222
    k8s-load-balancer: SSH username: vagrant
    k8s-load-balancer: SSH auth method: private key
    k8s-load-balancer: 
    k8s-load-balancer: Vagrant insecure key detected. Vagrant will automatically replace
    k8s-load-balancer: this with a newly generated keypair for better security.
    k8s-load-balancer: 
    k8s-load-balancer: Inserting generated public key within guest...
    k8s-load-balancer: Removing insecure key from the guest if it's present...
    k8s-load-balancer: Key inserted! Disconnecting and reconnecting using new SSH key...
==> k8s-load-balancer: Machine booted and ready!
==> k8s-load-balancer: Checking for guest additions in VM...
==> k8s-load-balancer: Setting hostname...
==> k8s-load-balancer: Configuring and enabling network interfaces...
==> k8s-load-balancer: Mounting shared folders...
    k8s-load-balancer: /vagrant => /home/adam/dev/yak8shwr
```

Now we can connect to the box.  Let's test our connection:

```bash
vagrant ssh k8s-load-balancer`
```

Great, it worked!

```bash
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-76-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Feb 16 16:30:59 UTC 2020

  System load:  0.05              Processes:             99
  Usage of /:   10.0% of 9.63GB   Users logged in:       0
  Memory usage: 6%                IP address for enp0s3: 10.0.2.15
  Swap usage:   0%                IP address for enp0s8: 10.0.0.10


0 packages can be updated.
0 updates are security updates.


Last login: Sun Feb 16 16:30:43 2020 from 10.0.2.2
vagrant@k8s-load-balancer:~$ 
```

Now, let's get rid of this box so we can create the full set of 7.  Exit the ssh session and run:

```bash
vagrant destroy -f
```

## The Vagrantfile, creating multiple boxes.

We will run vagrant up again, this time with a new Vagrantfile which will spin up 7 hosts.  Run:

```bash
{

cat > Vagrantfile <<EOF
API_VERSION = 2

box_image 					= "ubuntu/bionic64"
controller_vm_name_prefix 	= "k8s-controller-"
worker_vm_name_prefix 		= "k8s-worker-"
lb_vm_name 					= "k8s-load-balancer"

Vagrant.configure("2") do |config|

	# Define controllers
	(0..3- 1).each do |i|
		config.vm.define controller_vm_name_prefix + "#{i}" do |node|
			node.vm.box = box_image
			node.vm.hostname = controller_vm_name_prefix + "#{i}"
            node.vm.network :private_network, ip: "10.0.0.#{i + 10}"
			node.vm.provider "virtualbox" do |v|
				v.name = controller_vm_name_prefix + "#{i}"
				v.memory = 1024
				v.cpus = 1
			end
		end
	end

	# Define workers
	(0..3- 1).each do |i|
		config.vm.define worker_vm_name_prefix + "#{i}" do |node|
			node.vm.box = box_image
			node.vm.hostname = worker_vm_name_prefix + "#{i}"
            node.vm.network :private_network, ip: "10.0.0.#{i + 20}"
			node.vm.provider "virtualbox" do |v|
				v.name = worker_vm_name_prefix + "#{i}"
				v.memory = 2048
				v.cpus = 2
			end
		end
	end

	# Define load balancer
	config.vm.define lb_vm_name do |node|
		node.vm.box = box_image
		node.vm.hostname = lb_vm_name
		node.vm.network :private_network, ip: "10.0.0.30"
		node.vm.provider "virtualbox" do |v|
			v.name = lb_vm_name
			v.memory = 256
			v.cpus = 1
		end
	end

	# Allow password auth
	config.vm.provision 'shell', inline: 'sed -i "s/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g" /etc/ssh/sshd_config'
	config.vm.provision 'shell', inline: 'service ssh restart'
end
EOF

}
```

Vagrant will only bring up one host at a time which takes a while with 7 hosts.  Instead of running `vagrant up` this time we will start them all in parallel:

***WARNING*** If you don't have a machine with enough cores, memory, or disk this will hang your computer.

```bash
{

HOSTS=(k8s-controller-0 k8s-controller-1 k8s-controller-2 k8s-worker-0 k8s-worker-1 k8s-worker-2 k8s-load-balancer)

printf "%s\n" ${HOSTS[@]} | xargs -P7 -I {} vagrant up {}

}
```

Run `vagrant status` to confirm all the hosts are up

```bash
Current machine states:

k8s-controller-0          running (virtualbox)
k8s-controller-1          running (virtualbox)
k8s-controller-2          running (virtualbox)
k8s-worker-0              running (virtualbox)
k8s-worker-1              running (virtualbox)
k8s-worker-2              running (virtualbox)
k8s-load-balancer         running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

We need a couple of packages installed on our controllers and workers. Normally, we would use a Vagrant provisioner but for this exercise it is important to understand every step so we will run this ourselves:


```bash
{

HOSTS=(k8s-controller-0 k8s-controller-1 k8s-controller-2 k8s-worker-0 k8s-worker-1 k8s-worker-2)

printf "%s\n" ${HOSTS[@]} | xargs -P7 -I {} vagrant ssh -c "sudo apt install conntrack socat -y" {}

}
```

Next: [Installing the Client Tools](02-install-tools.md)