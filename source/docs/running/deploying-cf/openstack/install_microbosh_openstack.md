#Installing Micro BOSH on a VM

## <a id="prerequisites"></a>Prerequisites ##

* [Install OpenStack](http://docs.openstack.org/grizzly/basic-install/apt/content/)  
or 
* [DevStack install](http://devstack.org/) for testing environments


## <a id="step1"></a>Step 1: Prepare OpenStack and create an Inception VM ##


### <a id="step1.1"></a>Step 1.1: With Dashboard
1. Log in to dashboard (Horizon) as privileged user.
2. Select Project tab - > your created CF project.
3. Click on Access & Security - > Click on Keypairs tab and the Create Keypair button.
    - Enter the keypair name as admin-keypair and click on Create Keypair.
    - Save the keypair to some location like: `/home/<username>/openstack/admin-keypair.pem`
	- **Note:** Remember the keypair location. You would use this pair many times later.
4. On Access & Security - > Click on Security Groups tab and the Create Security Group button
	- Enter the Security Group name as *bosh*
	- **Note** We use this group for all instances
5. Edit your new Security Group
	- Click on Edit Rules button and adding some rules
		- `Proto: ICMP ; Open: Port ; From: -1 ; To: -1 ; Source: CIDR ; CIDR: 0.0.0.0/0`
		- `Proto: TCP  ; Open: Port ; From: 22 ; To: 22 ; Source: CIDR ; CIDR: 0.0.0.0/0`
		- `Proto: TCP  ; Open: Port ; From: 80 ; To: 80 ; Source: CIDR ; CIDR: 0.0.0.0/0`
		- `Proto: TCP  ; Open: Port ; From: 443; To: 443; Source: CIDR ; CIDR: 0.0.0.0/0`
		- `Proto: TCP  ; Open: PortRange ; From: 1 ; To: 65535 ; Source: CIDR ; CIDR: [Private-Network-Range]` - > this will open all ports for communication between the VM's
		- `Proto: UDP  ; Open: PortRange ; From: 1 ; To: 65535 ; Source: CIDR ; CIDR: [Private-Network-Range]`
6. Click on Images & Snapshots and make sure you have an Ubuntu 12.04 LTS image ready to use
	- If you don't see any images you need to [upload](http://docs.openstack.org/trunk/openstack-compute/admin/content/adding-images.html) a new image with CLI or Horizon
7. Click on Instances and launch your Inception VM with your created admin-keypair, Ubuntu image and bosh rule
	- **Note**: You'r Inception VM need enough disk space a good choice is more then 20GB

### <a id="step1.2"></a>Step 1.2: With CLI (Optional)
Use your admin credentials and create a rc file

    ~> vi adminrc
    export OS_TENANT_NAME=cf-testing 
    export OS_USERNAME=admin
    export OS_PASSWORD=<password>
    export OS_AUTH_URL="https://<KeystoneIP>:5000/v2.0/"

load your credentials

    ~> source adminrc
create your own keypair

    ~> nova keypair-add admin-keypair > /home/<username>/openstack/admin-keypair.pem
    ~> chmod 0600 /home/<username>/openstack/admin-keypair.pem

create security group and rules  
**Note** Since Grizzly you should use Quantum network manager instead of nova-network (deprecated).

    ~> quantum security-group-create –tenant-id <your tenant-id> bosh
	~> quantum security-group-rule-create –tenant-id <your tenant-id> –protocol TCP –remote-ip-prefix 0.0.0.0/0 –port-range-min 22 –port-range-max 22 <your group-id>
    ~> quantum security-group-rule-create –tenant-id <your tenant-id> –protocol TCP –remote-ip-prefix <your Network> –port-range-min 1 –port-range-max 65535 <your group-id>
    ~> quantum security-group-rule-create –tenant-id <your tenant-id> –protocol UDP –remote-ip-prefix <your Network> –port-range-min 1 –port-range-max 65535 <your group-id>
    ~> quantum security-group-rule-create –tenant-id <your tenant-id> –protocol TCP –remote-ip-prefix 0.0.0.0/0 –port-range-min 443 –port-range-max 443 <your group-id>
    ~> quantum security-group-rule-create –tenant-id <your tenant-id> –protocol TCP –remote-ip-prefix 0.0.0.0/0 –port-range-min 80 –port-range-max 80 <your group-id>
    ~> quantum security-group-rule-create –tenant-id <your tenant-id> –protocol ICMP –remote-ip-prefix 0.0.0.0/0 –port-range-min -1 –port-range-max -1 <your group-id>

checking images if you have a clear Ubuntu 12.04 LTS image

    ~> nova image-list
if you don't see any image please [check this doc](http://docs.openstack.org/trunk/openstack-compute/admin/content/adding-images.html) to upload a new one.

create a flavor for your bosh vm's

	~> nova flavor-create --ephemeral 10 --swap 0 --is-public false bosh.small <free id> 2048 10 4
You can use this flavor for all images types in a test or dev environment. In production you need to increase this values specially for dea's.  
In Addition make sure you have only two of three disks otherwise you will get trouble with bosh and the automatically attached volumes.

start your inception vm
    
    ~> nova boot --image <image-name> --flavor bosh.small --key-name admin-keypair --security-groups bosh --nic <net-id=net-uuid (priv)> inception
You should use your private network explicit.

## <a id="step2"></a>Step 2: Log in to the VM ##

    sudo su
    (Enter password and hit Enter)

Check whether SSH is installed:

    /etc/init.d/ssh status

If necessary, install SSH:

    apt-get install ssh


Log in to vm:

    ssh -i /home/<username>/openstack/admin-keypair.pem ubuntu@[VM-IP]


**Note:** VM-IP is the Inception Public IP Address or if you have access to the fixed network you can use the fixed IP Address


## <a id="step3"></a>Step 3: Install Ruby ##

Install Ruby and RubyGems following the instructions on the [Installing Ruby](/docs/common/install_ruby.html) page.
 

## <a id="step4"></a>Step 4: Install Bosh CLI ##

Install the BOSH command line interface.


    gem install bosh_deployer --no-ri --no-rdoc



## <a id="step5"></a>Step 5: Create Custom Micro Bosh Stemcell ##

Install the dependencies before you run the commands that follow.
    
    apt-get install build-essential libsqlite3-dev curl rsync git-core \
                    libmysqlclient-dev libxml2-dev libxslt-dev libpq-dev genisoimage mkpasswd \
                    libreadline6-dev libyaml-dev sqlite3 autoconf libgdbm-dev libncurses5-dev \
                    automake libtool bison pkg-config libffi-dev debootstrap kpartx qemu -y

Download the BOSH release and build it

    mkdir -p releases
    cd  releases
    git clone git://github.com/frodenas/bosh-release.git
    cd bosh-release
    git submodule update --init


####Build the BOSH release


    bosh create release --with-tarball

If this is the first time you have run Bosh, create the release in the release repo. You are asked to name the release, for example, "bosh".

    Output will be like this:
    Release version: x.y-dev
    Release manifest: /home/ubuntu/releases/bosh-release/dev_releases/bosh-x.y-dev.yml
    Release tarball (95.2M): /home/ubuntu/releases/bosh-release/dev_releases/bosh-x.y-dev.tgz

####Install BOSH Agent

    cd /home/ubuntu/
    git clone git://github.com/frodenas/bosh.git
    cd bosh/agent/
    bundle install --without=development test

    apt-get install libpq-dev

####Install OpenStack Registry

    cd /home/ubuntu/bosh/openstack_registry
    bundle install --without=development test
    bundle exec rake install

####Build Custom Stemcell

    root@inception-vm:/home/ubuntu/bosh/openstack_registry/# cd /home/ubuntu/bosh/agent
    root@inception-vm:/home/ubuntu/bosh/agent/# rake stemcell2:micro["openstack",/home/ubuntu/releases/bosh-release/micro/openstack.yml,/home/ubuntu/releases/bosh-release/dev_releases/bosh-x.y-dev.tgz]

**Note:** Replace x.y with actual bosh version numbers. For example: bosh-0.6-dev.tgz


Output will be like this:

    Generated stemcell: 
    /var/tmp/bosh/agent-x.y.z-nnnnn/work/work/micro-bosh-stemcell-openstack-x.y.z.tgz

####Copy the generated stemcell to a safe location

    cd /home/ubuntu/
    mkdir -p stemcells
    cd stemcells
    cp /var/tmp/bosh/agent-x.y.z-nnnnn/work/work/micro-bosh-stemcell-openstack-x.y.z.tgz .


## <a id="step6"></a>Step 6: Deploy Micro Bosh Stemcell to Glance ##

This creates the Micro Bosh VM and it shows up in Horizon 

    mkdir -p deployments/microbosh-openstack
    cd deployments/microbosh-openstack

####Create Manifest File

    vim micro_bosh.yml

Copy the below content and paste it in `micro_bosh.yml`


    name: microbosh-openstack

    env:
     bosh:
        password: $6$u/dxDdk4Z4Q3$MRHBPQRsU83i18FRB6CdLX0KdZtT2ZZV7BLXLFwa5tyVZbWp72v2wp.ytmY3KyBZzmdkPgx9D3j3oHaDZxe6F.


     level: DEBUG

    network:
     name: default
     type: dynamic
     label: private
     ip: 192.168.22.34


    resources:
     persistent_disk: 4096
     cloud_properties:
        instance_type: m1.small

    cloud:
      plugin: openstack
      properties:
       openstack:
           auth_url: http://10.0.0.2:5000/v2.0/tokens
           username: admin
           api_key: f00bar
           tenant: admin
           default_key_name: admin-keypair
           default_security_groups: ["default"]
           private_key: /root/.ssh/admin-keypair.pem



**Note:**

    1. Replace Only the red colored values with actual ones.
    2. Generate hashed password for f00bar
    3. Replace the password with hashed password.
 
----

    cd ..
    bosh micro deployment microbosh-openstack

**Output of the command is listed below:**

    $ WARNING! Your target has been changed to `http://microbosh-openstack:25555'!
    Deployment set to '/home/ubuntu/deployments/microbosh-openstack/micro_bosh.yml'


####Deploy the deployment using the custom stemcell image

    root@inception-vm:/home/ubuntu/deployments/# bosh micro deploy /home/ubuntu/stemcells/micro-bosh-stemcell-openstack-x.y.z.tgz

**Output of the command is listed below:**

    Deploying new micro BOSH instance `microbosh-openstack/micro_bosh.yml' to `http://microbosh-openstack:25555' (type 'yes' to continue): yes

    Verifying stemcell...
    File exists and readable                                     OK
    Manifest not found in cache, verifying tarball...
    Extract tarball                                              OK
    Manifest exists                                              OK
    Stemcell image file                                          OK
    Writing manifest to cache...
    Stemcell properties                                          OK

    Stemcell info
    -------------
    Name:    micro-bosh-stemcell
    Version: 0.6.4


    Deploy Micro BOSH
      unpacking stemcell (00:00:43)
      uploading stemcell (00:32:25)
      creating VM from 5aa08232-e53b-4efe-abee-385a7afb9421 (00:04:38)
      waiting for the agent (00:02:19)
      create disk (00:00:15)
      mount disk (00:00:07)
      stopping agent services (00:00:01)
      applying micro BOSH spec (00:01:20)
      starting agent services (00:00:00)
      waiting for the director (00:02:21)
    Done             11/11 00:44:30
    WARNING! Your target has been changed to `http://192.168.22.34:25555'!
    Deployment set to '/home/ubuntu/deployments/microbosh-openstack/micro_bosh.yml'
    Deployed `microbosh-openstack/micro_bosh.yml' to `http://microbosh-openstack:25555', took 00:44:30 to complete


####Test Micro BOSH deployment

    bosh target http://192.168.22.34

**Output of the command is listed below:**

    Target set to `microbosh-openstack (http://192.168.22.34:25555) Ver: 0.6 (release:ce0274ec bosh:0d9ac4d4)'
    Your username: admin
    Enter password: *****
    Logged in as `admin'

**Note:** You are prompted for the username and password; enter admin for both. Also, create a new BOSH user - the 'admin' is automatically deleted.
