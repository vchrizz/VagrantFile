# -*- mode: ruby -*-
# vi: set ft=ruby :

# automated setup of virtual machine:
# *) debian 9 (stretch) x86_64 installation
# *) debian-jenkins-glue installation and configuration with required plugins
# *) olsrd and olsrd2 job creation for building debian packages
#
# how to use this Vagrantfile:
# *) setup hyper-v
# *) install vagrant (v2.2.0)
# *) copy this Vagrantfile into a new folder/directory
# *) run in cmd as admin:
#    cd <to-new-folder>
#    vagrant box add generic/debian9
#    vagrant up

require 'yaml'
current_dir = File.dirname(File.expand_path(__FILE__))
configs     = YAML.load_file("#{current_dir}/config.yaml")
yamlconfig  = configs['configs'][configs['configs']['use']]

Vagrant.configure("2") do |config|
  #config.vm.box = "debian/stretch64" # does not support provider hyperv
  config.vm.box = "generic/debian9"
  config.vm.network yamlconfig['network'], bridge: yamlconfig['bridge'], mac: yamlconfig['mac'], type: yamlconfig['type'], ip: yamlconfig['ip']
  config.vm.define yamlconfig['vmname']
  config.vm.hostname = yamlconfig['hostname']
  config.vm.provider :hyperv do |hv|
    hv.vmname = yamlconfig['vmname']
    hv.cpus = yamlconfig['cpu']
    hv.memory = yamlconfig['mem']
    hv.maxmemory = yamlconfig['maxmem']
    hv.vlan_id = yamlconfig['vlan']
    hv.enable_checkpoints = false
    hv.linked_clone = true
    hv.auto_start_action = "StartIfRunning"
    hv.auto_stop_action = "Save"
  end
  config.vm.provision "shell", inline: <<-SHELL
    #
    # customizations
    #
    # default jenkins-debian-glue password
    jenkinspass=$(date +%N | sha256sum | base64 | head -c 8) # default jenkins-user: jenkins-debian-glue
    # credentials for additional custom user
    username='my-user'
    password='my-secret-password'
    # set location
    echo "Europe/Vienna" > /etc/timezone
    ln -fs /usr/share/zoneinfo/Europe/Vienna /etc/localtime
    # set apt sources.list to local mirror
    sed -i '/deb.debian.org/d' /etc/apt/sources.list
    sed -i 's/ftp.us.debian.org/ftp.nl.debian.org/' /etc/apt/sources.list

    #
    # setup system with jenkins-debian-glue
    #
    # install ssh server and apache2 webserver for apt repository access
    apt-get update && apt-get install -y ssh sudo apache2
    # create link for webserver to debian repo built by jenkins
    ln -s /srv/repository /var/www/html/debian
    # add custom system user
    useradd -m -s /bin/bash ${username}
    echo "${username}:${password}" | chpasswd
    echo "${username}   ALL=NOPASSWD: ALL" >> /etc/sudoers
    # install jenkins-debian-glue
    wget https://raw.github.com/mika/jenkins-debian-glue/master/puppet/apply.sh
    sudo bash ./apply.sh ${jenkinspass}
    echo "wait until jenkins is fully up and running ..."
    ( tail -f -n0 /var/log/jenkins/jenkins.log & ) | grep -q "Jenkins is fully up and running"
    # access http://<ip-of-vm>:8080 - user/pass: jenkins-debian-glue / ${jenkinspass}

    #
    # configuration
    #
    echo "installing additional required jenkins plugins ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} install-plugin ansible pipeline-milestone-step lockable-resources cloudbees-folder pipeline-model-definition workflow-cps workflow-multibranch pipeline-stage-step pipeline-stage-view workflow-cps-global-lib jackson2-api pipeline-build-step
    echo "create custom jenkins-user ..."
    echo 'jenkins.model.Jenkins.instance.securityRealm.createAccount("'${username}'","'${password}'")' | java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth groovy =
    #echo "set jenkins url ..."
    #java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} 
    echo "set 16 build processors ..."
    sed -i "s|.*numExecutors.*|<numExecutors>16<\/numExecutors>|" /var/lib/jenkins/config.xml
    # This is needed since j-d-g needs to pass certain pbuilder configurations dynamically during runtime.
    # https://github.com/mika/jenkins-debian-glue/issues/201
    sudo echo 'PBUILDER_CONFIG=/etc/jenkins/pbuilderrc' >> /etc/jenkins/debian_glue
    sudo echo 'export PBUILDERSATISFYDEPENDSCMD=/usr/lib/pbuilder/pbuilder-satisfydepends-apt' >> /etc/jenkins/pbuilderrc
    # fix for verifying debian packages with gpgv
    # https://github.com/mika/jenkins-debian-glue/issues/203
    #sudo echo 'export GNUPGHOME=/srv/repository/.gnupg/' >> /etc/jenkins/debian_glue
    #sudo echo 'BINDMOUNTS=/srv/repository/.gnupg/' >> /etc/jenkins/debian_glue

    # Generate a key executing 'gpg --gen-key' as user jenkins
    echo "GNUPGHOME: $GNUPGHOME"
    export GNUPGHOME="~/.gnupg/"
    sudo -u jenkins cat >foo <<EOF
      %echo Generating a basic OpenPGP key
      # Debian wants stronger RSA keys that are at least 4096 bits and preferring SHA2.
      Key-Type: RSA
      Key-Length: 4096
      Subkey-Type: RSA
      Subkey-Length: 4096
      Name-Real: Jenkins Debian Glue
      #Name-Comment: autocreated
      Name-Email: jenkins@$HOSTNAME
      Expire-Date: 0
      #Passphrase: ${jenkinspass}
      %no-protection
      # Do a commit here, so that we can later print "done" :-)
      %commit
      %echo done
EOF
#     fix
#    mkdir -p /srv/repository/.gnupg/
#    chmod 700 /srv/repository/.gnupg/
#    echo "generate gpg key for user jenkins ..."
#    sudo -u jenkins gpg --no-default-keyring --keyring /srv/repository/.gnupg/trustedkeys.kbx --batch --generate-key foo
#    cp /srv/repository/.gnupg/trustedkeys.kbx /srv/repository/.gnupg/trustedkeys.gpg
#    chown -R jenkins:jenkins /srv/repository/.gnupg/

    echo "generate gpg key for user jenkins ..."
    sudo -u jenkins gpg --no-default-keyring --keyring /var/lib/jenkins/.gnupg/trustedkeys.kbx --batch --generate-key foo
    cp /var/lib/jenkins/.gnupg/trustedkeys.kbx /srv/repository/.gnupg/trustedkeys.gpg
    chmod 0644 /var/lib/jenkins/.gnupg/trustedkeys.gpg
    # put key-id in variable
    gpgkey=$(sudo -u jenkins gpg --no-default-keyring --keyring /var/lib/jenkins/.gnupg/trustedkeys.kbx --list-keys --with-colons | awk -F: '/sub/ {print $5}')

    # export key for user root, import key as user root
    echo "export gpg key from user jenkins ..."
    sudo -u jenkins gpg --no-default-keyring --keyring /var/lib/jenkins/.gnupg/trustedkeys.kbx --export-secret-keys > /var/lib/jenkins/.gnupg/.private.key
    echo "import gpg key to user root ..."
    gpg --no-default-keyring --keyring /root/.gnupg/trustedkeys.kbx --import /var/lib/jenkins/.gnupg/.private.key
    #gpg --no-default-keyring --keyring /root/.gnupg/trustedkeys.kbx --list-secret-keys
    gpg --list-secret-keys
    echo "import gpg key in pubring.kbx"
    gpg --import /var/lib/jenkins/.gnupg/.private.key
    sudo -u jenkins gpg --import /var/lib/jenkins/.gnupg/.private.key
    #sudo -u jenkins gpg --no-default-keyring -a --export ${gpgkey} | gpg --no-default-keyring --keyring ~/.gnupg/trustedkeys.gpg --import -
    # and then adjust /etc/jenkins/debian_glue.
    sed -i "s/.*KEY_ID=.*/KEY_ID=${gpgkey}/" /etc/jenkins/debian_glue
    grep KEY_ID= /etc/jenkins/debian_glue
    sed -i "s|.*REPOSITORY_KEYRING=.*|REPOSITORY_KEYRING=/var/lib/jenkins/.gnupg/pubring.kbx|" /etc/jenkins/debian_glue


    echo "restart jenkins to let all changes take effect ..."
    service jenkins restart && ( tail -f -n0 /var/log/jenkins/jenkins.log & ) | grep -q "Jenkins is fully up and running"
    echo "delete default jenkins-debian-glue jobs ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} delete-job jenkins-debian-glue-source jenkins-debian-glue-binaries jenkins-debian-glue-piuparts

    # get custom jenkins jobs
    git clone https://github.com/vchrizz/jenkins-jobs.git && cp -a jenkins-jobs/*/ /var/lib/jenkins/jobs/
    rm -rf jenkins-jobs/
    chown -R jenkins. /var/lib/jenkins/jobs/
    # import custom jenkins jobs
    echo "create olsrd jobs ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd-source < /var/lib/jenkins/jobs/olsrd-source/config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd-binaries < /var/lib/jenkins/jobs/olsrd-binaries/config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd-piuparts < /var/lib/jenkins/jobs/olsrd-piuparts/config.xml
    echo "create olsrd2 jobs ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd2-source < /var/lib/jenkins/jobs/olsrd2-source/config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd2-binaries < /var/lib/jenkins/jobs/olsrd2-binaries/config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd2-piuparts < /var/lib/jenkins/jobs/olsrd2-piuparts/config.xml
    echo "create icinga2 jobs ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job icinga2-source < /var/lib/jenkins/jobs/icinga2-source/config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job icinga2-binaries < /var/lib/jenkins/jobs/icinga2-binaries/config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job icinga2-piuparts < /var/lib/jenkins/jobs/icinga2-piuparts/config.xml

    # switch ip (temp fix for #10329 until vagrant is able to properly set static ip on hyper-v)
    echo "setting ip to static 10.5.44.233/24 ..."
    sed -i '/dhcp/d' /etc/network/interfaces
    cat << 'EOF' >> /etc/network/interfaces
iface eth0 inet static
        address 10.5.44.233
        netmask 255.255.255.0
        gateway 10.5.44.101
EOF
    service networking restart && ifup eth0 &
    sleep 1
    /etc/init.d/networking restart && ifup eth0

    echo "start building job icinga2-source ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth build icinga2-source
    echo "start building job olsrd-source ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth build olsrd-source
    echo "start building job olsrd2-source ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth build olsrd2-source

    echo "all done."
    exit 0
  SHELL
end
