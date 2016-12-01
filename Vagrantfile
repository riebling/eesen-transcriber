# -*- mode: ruby -*-
# vi: set ft=ruby

Vagrant.configure("2") do |config|

    # Default provider is VirtualBox!
    # If you want AWS, you need to populate and run e.g.
    #   . aws.sh; vagrant up --provider aws
    # Make sure you don't check in aws.sh (maybe make a copy with your "secret" data)
    # Before that, do
    #   vagrant plugin install vagrant-aws; vagrant plugin install vagrant-sshfs

#    config.vm.box = "bento/ubuntu-16.04"

    config.vm.box = "https://cloud-images.ubuntu.com/releases/xenial/release/ubuntu-16.04-server-cloudimg-amd64-vagrant.box"
    config.vm.synced_folder ".", "/vagrant", :mount_options => ["dmode=777", "fmode=777"]

    config.vm.provider "virtualbox" do |vbox|
      config.ssh.forward_x11 = true

      # enable (uncomment) this for debugging output
      #vbox.gui = true

      # fix Virtualbox timeout bug (older version)
      vbox.customize ["modifyvm", :id, "--cableconnected1", "on"]

      # host-only network on which web browser serves files
      config.vm.network "private_network", ip: "192.168.56.101"

      vbox.cpus = 2
      vbox.memory = 8192
    end

    config.vm.provider "aws" do |aws, override|

      aws.tags["Name"] = "Eesen Transcriber"
#      aws.ami = "ami-45b69e52" # Ubuntu ("Xenial") Server 16.04 LTS AMI - US-East region hvm:ebs-ssd
#      aws.ami = "ami-fdb69eea" # Ubuntu ("Xenial") Server 16.04 LTS AMI - US-East region ebs-ssd
      aws.ami = "ami-abbc94bc"  # release 20161115 hvm-instance
      aws.instance_type = "m3.xlarge"

      override.vm.synced_folder ".", "/vagrant", type: "sshfs", ssh_username: ENV['USER'], ssh_port: "22", prompt_for_password: "true"

      override.vm.box_url = "http://speechkitchen.org/dummy.box"

      # it is assumed these environment variables were set by ". env.sh"
      aws.access_key_id = ENV['AWS_KEY']
      aws.secret_access_key = ENV['AWS_SECRETKEY']
      aws.keypair_name = ENV['AWS_KEYPAIR']
      override.ssh.username = "ubuntu"
      override.ssh.private_key_path = ENV['AWS_PEM']

      aws.terminate_on_shutdown = "true"
      aws.region = ENV['AWS_REGION']

      # https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#SecurityGroups
      # Edit the security group on AWS Console; Inbound tab, add the HTTP rule
    aws.security_groups = ENV['AWS_SECURITYGROUP']

      #aws.subnet_id = "vpc-666c9a02"
      aws.region_config "us-east-1" do |region|
        #region.spot_instance = true
        region.spot_max_price = "0.1"
      end

      # this works around the error from AWS AMI vm on 'vagrant up':
      #   No host IP was given to the Vagrant core NFS helper. This is
      #   an internal error that should be reported as a bug.
      #override.nfs.functional = false
    end

  config.vm.provision "shell", inline: <<-SHELL
    apt-get update

    apt-mark hold grub-legacy-ec2
    apt-get upgrade -y

    if grep --quiet vagrant /etc/passwd
    then
      user="vagrant"
    else
      user="ubuntu"
    fi

    apt-get install -y git make automake libtool-bin autoconf patch subversion fuse\
       libatlas-base-dev libatlas-dev liblapack-dev sox openjdk-8-jre libav-tools g++\
       zlib1g-dev libsox-fmt-all apache2 sshfs python2.7

    update-alternatives --install /usr/bin/python python /usr/bin/python2.7 1

    # fix dash-as-bash
    rm /bin/sh
    ln -s /bin/bash /bin/sh

    # If you wish to train EESEN with a GPU machine, uncomment this section to install CUDA
    # also uncomment the line that mentions cudatk-dir in the EESEN install section below
    #cd /home/${user}
    #wget -nv http://speechkitchen.org/vms/Data/cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
    #dpkg -i cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
    #rm cuda-repo-ubuntu1404-7-5-local_7.5-18_amd64.deb
    #apt-get update                                                                  
    #apt-get remove --purge xserver-xorg-video-nouveau                           
    #apt-get install -y cuda

    # install srvk EESEN (does not require CUDA)
    git clone https://github.com/srvk/eesen
    cd eesen
    git reset --hard 698ee6f

    cd tools
    make -j `lscpu -p|grep -v "#"|wc -l`
    # remove a parameter from scoring script
    sed -i 's/\ lur//g' sctk/bin/hubscr.pl
    cd ../src
    ./configure --shared #--cudatk-dir=/opt/nvidia/cuda
    make -j `lscpu -p|grep -v "#"|wc -l`

    # install language model building toolkit
    cd /home/${user}/eesen/asr_egs/tedlium/v2-30ms
    git clone http://github.com/srvk/lm_build

    # get eesen-offline-transcriber
    mkdir -p /home/${user}/tools
    cd /home/${user}/tools
    git clone https://github.com/srvk/srvk-eesen-offline-transcriber
    mv srvk-eesen-offline-transcriber eesen-offline-transcriber
    # make links to EESEN
    cd eesen-offline-transcriber
    ln -s /home/${user}/eesen/asr_egs/tedlium/v2-30ms/steps .
    ln -s /home/${user}/eesen/asr_egs/tedlium/v2-30ms/utils .

    # get models
    cd /home/${user}/eesen/asr_egs/tedlium
    wget -nv http://speechkitchen.org/vms/Data/v2-30ms.tgz
    tar zxvf v2-30ms.tgz --dereference
    rm v2-30ms.tgz
    # optionally get 8khz models
    if [ -f /vagrant/swbd-v1-pitch.tgz ]
    then
       cd /home/${user}/eesen/asr_egs/swbd
       tar zxvf /vagrant/swbd-v1-pitch.tgz
    fi

    # Uncomment for optional large language model rescoring
    # produces generally 2% better Word Error Rates at the epense of longer
    # decoding time and memory requirements (Requires guest VM setting
    # of at least vbox.memory = 15360, just barely fitting in a 16GB host
    # computer - with warnings)   Substitute "make -f Makefile.rescore"
    # for "make" in run scripts (speech2text.sh and friends) to use this.
    # 
    # cd /home/${user}
    # wget http://speechkitchen.org/vms/Data/rescore-eesen.tgz
    # tar zxvf rescore-eesen.tgz
    # rm rescore-eesen.tgz    

    # Results (and intermediate files) are placed on the shared host folder
    mkdir -p /vagrant/{build,log,transcribe_me,src-audio}

    ln -s /vagrant/build /home/${user}/tools/eesen-offline-transcriber/build
    ln -s /vagrant/src-audio /home/${user}/tools/eesen-offline-transcriber/src-audio

    # get XFCE, xterm if we want guest VM to open windows /menus on host
    #apt-get install -y xfce4-panel xterm

    # Apache: set up web content
    cd /vagrant
    git clone http://github.com/srvk/www

    # set the shared folder to be (mounted as a shared folder in the VM) "www"
    sed -i 's|/var/www/html|/vagrant/www|g' /etc/apache2/sites-enabled/000-default.conf
    sed -i 's|/var/www/|/vagrant/www/|g' /etc/apache2/apache2.conf
    service apache2 restart

    # shorten paths used by vagrant ssh -c <command> commands
    # by symlinking ~/bin to here
    ln -s /home/${user}/tools/eesen-offline-transcriber /home/${user}/bin

    chmod g-w /var/log

    # get SLURM stuff
    apt-get install -y --no-install-recommends slurm-llnl < /usr/bin/yes
    /usr/sbin/create-munge-key -f
    mkdir -p /var/run/munge /var/run/slurm-llnl
    chown munge:root /var/run/munge
    chown slurm:slurm /var/run/slurm-llnl
    echo 'OPTIONS="--force --syslog"' >> /etc/default/munge
    cp /vagrant/conf/slurm.conf /etc/slurm-llnl/slurm.conf
    cp /vagrant/conf/reconf-slurm.sh /root/

    # 
    # Supervisor stuff needed by slurm
    # copy config first so it gets picked up
    cp /vagrant/conf/supervisor.conf /etc/supervisor.conf
    mkdir -p /etc/supervisor/conf.d
    cp /vagrant/conf/slurm.sv.conf /etc/supervisor/conf.d/
    # Now start service
    apt-get install -y supervisor
    
    # Turn off release upgrade messages
    sed -i s/Prompt=lts/Prompt=never/ /etc/update-manager/release-upgrades
    rm -f /var/lib/ubuntu-release-upgrader/*
    /usr/lib/ubuntu-release-upgrader/release-upgrade-motd
    
    # Silence error message from missing file
    touch /home/${user}/.Xauthority 

    # Provisioning runs as root; we want files to belong to '${user}'
    chown -R ${user}:${user} /home/${user}

    # Handy info
    echo ""
    echo "------------------------------------------------------------"
    echo ""
    echo "  Watching folder [...]/eesen-transcriber/transcribe_me/"
    echo "    for new files to transcribe. Output *.ctm files"
    echo "    will appear alongside the original audio files"
    echo "    logs are in [...]/eesen-transcriber/log/"
    echo ""
    echo "  Point your Chrome or Safari browser to "
    if [ ${user} == vagrant ] 
    then
      echo "  http://192.168.56.101/ to view transcription results"
    else
      publicIP=`curl -s http://169.254.169.254/latest/meta-data/public-ipv4`
      echo "  http://${publicIP}/ to view transcription results"
    fi
    echo ""
    echo "------------------------------------------------------------"
  SHELL
end

# always monitor watched folder
  Vagrant.configure("2") do |config|
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    if grep --quiet vagrant /etc/passwd
    then
      user="vagrant"
    else
      user="ubuntu"
    fi

    rm -rf /var/run/motd.dynamic

    service slurmctld start
    service slurmd start

    su ${user} -c "cd /home/${user}/tools/eesen-offline-transcriber && ./watch.sh >& /vagrant/log/watched.log &"
SHELL
  end
