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
    jenkinspass=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 8) # default jenkins-user: jenkins-debian-glue
    # credentials for additional custom user
    username='myuser'
    password='my-secret-password'
    # set location
    echo "Europe/Vienna" > /etc/timezone
    ln -fs /usr/share/zoneinfo/Europe/Vienna /etc/localtime
    # set apt sources.list to local mirror
    sed -i '/deb.debian.org/d' /etc/apt/sources.list
    sed -i 's/ftp.us.debian.org/ftp.at.debian.org/' /etc/apt/sources.list

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
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} install-plugin pipeline-milestone-step lockable-resources cloudbees-folder pipeline-model-definition workflow-cps workflow-multibranch pipeline-stage-step pipeline-stage-view workflow-cps-global-lib jackson2-api pipeline-build-step
    echo "create custom jenkins-user ..."
    echo 'jenkins.model.Jenkins.instance.securityRealm.createAccount("'${username}'","'${password}'")' | java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth groovy =
    #echo "set jenkins url ..."
    #java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} 
    #echo "set 16 build processors ..."
    sed -i "s|.*numExecutors.*|<numExecutors>16<\/numExecutors>|" /var/lib/jenkins/config.xml
    # This is needed since j-d-g needs to pass certain pbuilder configurations dynamically during runtime.
    # https://github.com/mika/jenkins-debian-glue/issues/201
    sudo echo 'PBUILDER_CONFIG=/etc/jenkins/pbuilderrc' >> /etc/jenkins/debian_glue
    sudo echo 'export PBUILDERSATISFYDEPENDSCMD=/usr/lib/pbuilder/pbuilder-satisfydepends-apt' >> /etc/jenkins/pbuilderrc

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
    echo "generate gpg key for user jenkins ..."
    sudo -u jenkins gpg --no-default-keyring --keyring /var/lib/jenkins/.gnupg/trustedkeys.kbx --batch --generate-key foo
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

    #
    # create olsrd and olsrd2 jobs:
    #
    echo "save olsrd jenkins jobs to /tmp/vagrant/ ..."
    mkdir -p /tmp/vagrant/
cat << 'EOF' > /tmp/vagrant/olsrd-source_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <actions/>
  <description>Build Debian source package of olsrd</description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.plugins.git.GitSCM" plugin="git@3.9.1">
    <configVersion>2</configVersion>
    <userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://salsa.debian.org/vchrizz-guest/olsrd.git</url>
      </hudson.plugins.git.UserRemoteConfig>
    </userRemoteConfigs>
    <branches>
      <hudson.plugins.git.BranchSpec>
        <name>master</name>
      </hudson.plugins.git.BranchSpec>
    </branches>
    <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
    <submoduleCfg class="list"/>
    <extensions>
      <hudson.plugins.git.extensions.impl.RelativeTargetDirectory>
        <relativeTargetDir>source</relativeTargetDir>
      </hudson.plugins.git.extensions.impl.RelativeTargetDirectory>
      <hudson.plugins.git.extensions.impl.PerBuildTag/>
    </extensions>
  </scm>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.tasks.Shell>
      <command>rm -f ./* || true</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command># when using git:
/usr/bin/generate-git-snapshot

# when using subversion:
# /usr/bin/generate-svn-snapshot</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>mkdir -p report
/usr/bin/lintian-junit-report *.dsc &gt; report/lintian.xml</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>*.gz,*.bz2,*.xz,*.deb,*.dsc,*.git,*.changes,*.buildinfo,lintian.txt</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
    <hudson.tasks.BuildTrigger>
      <childProjects>olsrd-binaries</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal>0</ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.26.1">
      <testResults>**/lintian.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
      <allowEmptyResults>false</allowEmptyResults>
    </hudson.tasks.junit.JUnitResultArchiver>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
</project>
EOF

cat << 'EOF' > /tmp/vagrant/olsrd-binaries_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<matrix-project plugin="matrix-project@1.13">
  <actions/>
  <description>&lt;p&gt;Build Debian binary packages of olsrd&lt;/p&gt;&#xd;
&#xd;
&lt;h2&gt;Usage instructions how to remotely access and use the repository&lt;/h2&gt;&#xd;
&#xd;
&lt;p&gt;Install apache webserver:&lt;/p&gt;&#xd;
&#xd;
&lt;pre&gt;&#xd;
  sudo apt-get install apache2&#xd;
  sudo ln -s /srv/repository /var/www/debian&#xd;
&lt;/pre&gt;&#xd;
&#xd;
&lt;p&gt;Then access to this repository is available using the following sources.list entry:&lt;/p&gt;&#xd;
&#xd;
&lt;pre&gt;&#xd;
  deb     http://dev.onetrix.net/debian/ olsrd main&#xd;
  deb-src http://dev.onetrix.net/debian/ olsrd main&#xd;
&lt;/pre&gt;</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <jenkins.model.BuildDiscarderProperty>
      <strategy class="hudson.tasks.LogRotator">
        <daysToKeep>-1</daysToKeep>
        <numToKeep>10</numToKeep>
        <artifactDaysToKeep>-1</artifactDaysToKeep>
        <artifactNumToKeep>-1</artifactNumToKeep>
      </strategy>
    </jenkins.model.BuildDiscarderProperty>
  </properties>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>true</concurrentBuild>
  <axes>
    <hudson.matrix.TextAxis>
      <name>architecture</name>
      <values>
        <string>amd64</string>
        <string>i386</string>
        <string>mips</string>
        <string>mipsel</string>
        <string>mips64el</string>
        <string>armhf</string>
        <string>armel</string>
      </values>
    </hudson.matrix.TextAxis>
  </axes>
  <builders>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.41">
      <project>olsrd-source</project>
      <filter>*</filter>
      <target></target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <fallbackToLastSuccessful>true</fallbackToLastSuccessful>
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
        <allowUpstreamDependencies>false</allowUpstreamDependencies>
      </selector>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
    <hudson.tasks.Shell>
      <command>export ARCHITECTURES="source amd64 i386 mips mipsel mips64el armhf armel"
export POST_BUILD_HOOK=/usr/bin/jdg-debc
/usr/bin/build-and-provide-package</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo &quot;Listing packages inside the olsrd repository:&quot;
/usr/bin/repository_checker --list-repos olsrd</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>mkdir -p report
/usr/bin/lintian-junit-report *.dsc &gt; report/lintian.xml</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>*.gz,*.bz2,*.xz,*.deb,*.dsc,*.git,*.changes,*.buildinfo,lintian.txt</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.26.1">
      <testResults>**/lintian.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
      <allowEmptyResults>false</allowEmptyResults>
    </hudson.tasks.junit.JUnitResultArchiver>
    <hudson.tasks.BuildTrigger>
      <childProjects>olsrd-piuparts</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal>0</ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
  <executionStrategy class="hudson.matrix.DefaultMatrixExecutionStrategyImpl">
    <runSequentially>false</runSequentially>
  </executionStrategy>
</matrix-project>
EOF

cat << 'EOF' > /tmp/vagrant/olsrd-piuparts_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <actions/>
  <description>Installation and upgrade tests for olsrd Debian packages</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <hudson.model.ParametersDefinitionProperty>
      <parameterDefinitions>
        <hudson.model.StringParameterDefinition>
          <name>architecture</name>
          <description></description>
          <defaultValue>amd64</defaultValue>
          <trim>false</trim>
        </hudson.model.StringParameterDefinition>
      </parameterDefinitions>
    </hudson.model.ParametersDefinitionProperty>
  </properties>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.41">
      <project>olsrd-binaries/architecture=$architecture</project>
      <filter>*.deb</filter>
      <target>artifacts/</target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <fallbackToLastSuccessful>true</fallbackToLastSuccessful>
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
        <allowUpstreamDependencies>false</allowUpstreamDependencies>
      </selector>
      <flatten>true</flatten>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
    <hudson.tasks.Shell>
      <command># sadly piuparts always returns with exit code 1 :((
sudo piuparts_wrapper ${PWD}/artifacts/*.deb || true</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>piuparts_tap piuparts.txt &gt; piuparts.tap</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <org.tap4j.plugin.TapPublisher plugin="tap@2.2.1">
      <testResults>piuparts.tap</testResults>
      <failIfNoResults>false</failIfNoResults>
      <failedTestsMarkBuildAsFailure>false</failedTestsMarkBuildAsFailure>
      <outputTapToConsole>false</outputTapToConsole>
      <enableSubtests>true</enableSubtests>
      <discardOldReports>false</discardOldReports>
      <todoIsFailure>false</todoIsFailure>
      <includeCommentDiagnostics>true</includeCommentDiagnostics>
      <validateNumberOfTests>false</validateNumberOfTests>
      <planRequired>true</planRequired>
      <verbose>true</verbose>
      <showOnlyFailures>false</showOnlyFailures>
      <stripSingleParents>false</stripSingleParents>
      <flattenTapResult>false</flattenTapResult>
      <skipIfBuildNotOk>false</skipIfBuildNotOk>
    </org.tap4j.plugin.TapPublisher>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>piuparts.*</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
</project>
EOF

cat << 'EOF' > /tmp/vagrant/olsrd2-source_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <actions/>
  <description>Build Debian source package of olsrd2</description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.plugins.git.GitSCM" plugin="git@3.9.1">
    <configVersion>2</configVersion>
    <userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://salsa.debian.org/vchrizz-guest/olsrd2.git</url>
      </hudson.plugins.git.UserRemoteConfig>
    </userRemoteConfigs>
    <branches>
      <hudson.plugins.git.BranchSpec>
        <name>master</name>
      </hudson.plugins.git.BranchSpec>
    </branches>
    <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
    <submoduleCfg class="list"/>
    <extensions>
      <hudson.plugins.git.extensions.impl.RelativeTargetDirectory>
        <relativeTargetDir>source</relativeTargetDir>
      </hudson.plugins.git.extensions.impl.RelativeTargetDirectory>
      <hudson.plugins.git.extensions.impl.PerBuildTag/>
    </extensions>
  </scm>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.tasks.Shell>
      <command>rm -f ./* || true</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command># when using git:
/usr/bin/generate-git-snapshot

# when using subversion:
# /usr/bin/generate-svn-snapshot</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>mkdir -p report
/usr/bin/lintian-junit-report *.dsc &gt; report/lintian.xml</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>*.gz,*.bz2,*.xz,*.deb,*.dsc,*.git,*.changes,*.buildinfo,lintian.txt</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
    <hudson.tasks.BuildTrigger>
      <childProjects>olsrd2-binaries</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal>0</ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.26.1">
      <testResults>**/lintian.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
      <allowEmptyResults>false</allowEmptyResults>
    </hudson.tasks.junit.JUnitResultArchiver>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
</project>
EOF

cat << 'EOF' > /tmp/vagrant/olsrd2-binaries_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<matrix-project plugin="matrix-project@1.13">
  <actions/>
  <description>&lt;p&gt;Build Debian binary packages of olsrd2&lt;/p&gt;&#xd;
&#xd;
&lt;h2&gt;Usage instructions how to remotely access and use the repository&lt;/h2&gt;&#xd;
&#xd;
&lt;p&gt;Install apache webserver:&lt;/p&gt;&#xd;
&#xd;
&lt;pre&gt;&#xd;
  sudo apt-get install apache2&#xd;
  sudo ln -s /srv/repository /var/www/debian&#xd;
&lt;/pre&gt;&#xd;
&#xd;
&lt;p&gt;Then access to this repository is available using the following sources.list entry:&lt;/p&gt;&#xd;
&#xd;
&lt;pre&gt;&#xd;
  deb     http://dev.onetrix.net/debian/ olsrd2 main&#xd;
  deb-src http://dev.onetrix.net/debian/ olsrd2 main&#xd;
&lt;/pre&gt;</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <jenkins.model.BuildDiscarderProperty>
      <strategy class="hudson.tasks.LogRotator">
        <daysToKeep>-1</daysToKeep>
        <numToKeep>10</numToKeep>
        <artifactDaysToKeep>-1</artifactDaysToKeep>
        <artifactNumToKeep>-1</artifactNumToKeep>
      </strategy>
    </jenkins.model.BuildDiscarderProperty>
  </properties>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>true</concurrentBuild>
  <axes>
    <hudson.matrix.TextAxis>
      <name>architecture</name>
      <values>
        <string>amd64</string>
        <string>i386</string>
        <string>mips</string>
        <string>mipsel</string>
        <string>mips64el</string>
        <string>armhf</string>
        <string>armel</string>
      </values>
    </hudson.matrix.TextAxis>
  </axes>
  <builders>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.41">
      <project>olsrd2-source</project>
      <filter>*</filter>
      <target></target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <fallbackToLastSuccessful>true</fallbackToLastSuccessful>
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
        <allowUpstreamDependencies>false</allowUpstreamDependencies>
      </selector>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
    <hudson.tasks.Shell>
      <command>export ARCHITECTURES="source amd64 i386 mips mipsel mips64el armhf armel"
export POST_BUILD_HOOK=/usr/bin/jdg-debc
/usr/bin/build-and-provide-package</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo &quot;Listing packages inside the olsrd repository:&quot;
/usr/bin/repository_checker --list-repos olsrd2</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>mkdir -p report
/usr/bin/lintian-junit-report *.dsc &gt; report/lintian.xml</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>*.gz,*.bz2,*.xz,*.deb,*.dsc,*.git,*.changes,*.buildinfo,lintian.txt</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.26.1">
      <testResults>**/lintian.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
      <allowEmptyResults>false</allowEmptyResults>
    </hudson.tasks.junit.JUnitResultArchiver>
    <hudson.tasks.BuildTrigger>
      <childProjects>olsrd2-piuparts</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal>0</ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
  <executionStrategy class="hudson.matrix.DefaultMatrixExecutionStrategyImpl">
    <runSequentially>false</runSequentially>
  </executionStrategy>
</matrix-project>
EOF

cat << 'EOF' > /tmp/vagrant/olsrd2-piuparts_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <actions/>
  <description>Installation and upgrade tests for olsrd2 Debian packages</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <hudson.model.ParametersDefinitionProperty>
      <parameterDefinitions>
        <hudson.model.StringParameterDefinition>
          <name>architecture</name>
          <description></description>
          <defaultValue>amd64</defaultValue>
          <trim>false</trim>
        </hudson.model.StringParameterDefinition>
      </parameterDefinitions>
    </hudson.model.ParametersDefinitionProperty>
  </properties>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.41">
      <project>olsrd2-binaries/architecture=$architecture</project>
      <filter>*.deb</filter>
      <target>artifacts/</target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <fallbackToLastSuccessful>true</fallbackToLastSuccessful>
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
        <allowUpstreamDependencies>false</allowUpstreamDependencies>
      </selector>
      <flatten>true</flatten>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
    <hudson.tasks.Shell>
      <command># sadly piuparts always returns with exit code 1 :((
sudo piuparts_wrapper ${PWD}/artifacts/*.deb || true</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>piuparts_tap piuparts.txt &gt; piuparts.tap</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <org.tap4j.plugin.TapPublisher plugin="tap@2.2.1">
      <testResults>piuparts.tap</testResults>
      <failIfNoResults>false</failIfNoResults>
      <failedTestsMarkBuildAsFailure>false</failedTestsMarkBuildAsFailure>
      <outputTapToConsole>false</outputTapToConsole>
      <enableSubtests>true</enableSubtests>
      <discardOldReports>false</discardOldReports>
      <todoIsFailure>false</todoIsFailure>
      <includeCommentDiagnostics>true</includeCommentDiagnostics>
      <validateNumberOfTests>false</validateNumberOfTests>
      <planRequired>true</planRequired>
      <verbose>true</verbose>
      <showOnlyFailures>false</showOnlyFailures>
      <stripSingleParents>false</stripSingleParents>
      <flattenTapResult>false</flattenTapResult>
      <skipIfBuildNotOk>false</skipIfBuildNotOk>
    </org.tap4j.plugin.TapPublisher>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>piuparts.*</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
</project>
EOF

    #
    # create icinga2 jobs:
    #
    echo "save icinga2 jenkins jobs to /tmp/vagrant/ ..."
    mkdir -p /tmp/vagrant/
cat << 'EOF' > /tmp/vagrant/icinga2-source_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <actions/>
  <description>Build Debian source package of icinga2</description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.plugins.git.GitSCM" plugin="git@3.9.1">
    <configVersion>2</configVersion>
    <userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://salsa.debian.org/nagios-team/pkg-icinga2.git</url>
      </hudson.plugins.git.UserRemoteConfig>
    </userRemoteConfigs>
    <branches>
      <hudson.plugins.git.BranchSpec>
        <name>debian/2.10.1-3</name>
      </hudson.plugins.git.BranchSpec>
    </branches>
    <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
    <submoduleCfg class="list"/>
    <extensions>
      <hudson.plugins.git.extensions.impl.RelativeTargetDirectory>
        <relativeTargetDir>source</relativeTargetDir>
      </hudson.plugins.git.extensions.impl.RelativeTargetDirectory>
      <hudson.plugins.git.extensions.impl.PerBuildTag/>
    </extensions>
  </scm>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.tasks.Shell>
      <command>rm -f ./* || true</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command># when using git:
/usr/bin/generate-git-snapshot

# when using subversion:
# /usr/bin/generate-svn-snapshot</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>mkdir -p report
/usr/bin/lintian-junit-report *.dsc &gt; report/lintian.xml</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>*.gz,*.bz2,*.xz,*.deb,*.dsc,*.git,*.changes,*.buildinfo,lintian.txt</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>true</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
    <hudson.tasks.BuildTrigger>
      <childProjects>icinga2-binaries</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal>0</ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.26.1">
      <testResults>**/lintian.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
      <allowEmptyResults>false</allowEmptyResults>
    </hudson.tasks.junit.JUnitResultArchiver>
  </publishers>
  <buildWrappers/>
</project>
EOF

cat << 'EOF' > /tmp/vagrant/icinga2-binaries_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<matrix-project plugin="matrix-project@1.13">
  <actions/>
  <description>&lt;p&gt;Build Debian binary packages of icinga2&lt;/p&gt;&#xd;
&#xd;
&lt;h2&gt;Usage instructions how to remotely access and use the repository&lt;/h2&gt;&#xd;
&#xd;
&lt;p&gt;Install apache webserver:&lt;/p&gt;&#xd;
&#xd;
&lt;pre&gt;&#xd;
  sudo apt-get install apache2&#xd;
  sudo ln -s /srv/repository /var/www/debian&#xd;
&lt;/pre&gt;&#xd;
&#xd;
&lt;p&gt;Then access to this repository is available using the following sources.list entry:&lt;/p&gt;&#xd;
&#xd;
&lt;pre&gt;&#xd;
  deb     http://dev.onetrix.net/debian/ icinga2 main&#xd;
  deb-src http://dev.onetrix.net/debian/ icinga2 main&#xd;
&lt;/pre&gt;</description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <axes>
    <hudson.matrix.TextAxis>
      <name>architecture</name>
      <values>
        <string>amd64</string>
        <string>i386</string>
        <string>mips</string>
        <string>mipsel</string>
        <string>mips64el</string>
        <string>armhf</string>
        <string>armel</string>
      </values>
    </hudson.matrix.TextAxis>
  </axes>
  <builders>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.41">
      <project>icinga2-source</project>
      <filter>*</filter>
      <target></target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <fallbackToLastSuccessful>true</fallbackToLastSuccessful>
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
        <allowUpstreamDependencies>false</allowUpstreamDependencies>
      </selector>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
    <hudson.tasks.Shell>
      <command>export POST_BUILD_HOOK=/usr/bin/jdg-debc
/usr/bin/build-and-provide-package</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>echo &quot;Listing packages inside the icinga2 repository:&quot;
/usr/bin/repository_checker --list-repos icinga2</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>mkdir -p report
/usr/bin/lintian-junit-report *.dsc &gt; report/lintian.xml</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>*.gz,*.bz2,*.xz,*.deb,*.dsc,*.git,*.changes,*.buildinfo,lintian.txt</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>true</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.26.1">
      <testResults>**/lintian.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
      <allowEmptyResults>false</allowEmptyResults>
    </hudson.tasks.junit.JUnitResultArchiver>
    <hudson.tasks.BuildTrigger>
      <childProjects>icinga2-piuparts</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal>0</ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
  <executionStrategy class="hudson.matrix.DefaultMatrixExecutionStrategyImpl">
    <runSequentially>true</runSequentially>
  </executionStrategy>
</matrix-project>
EOF
cat << 'EOF' > /tmp/vagrant/icinga2-piuparts_config.xml
<?xml version='1.1' encoding='UTF-8'?>
<project>
  <actions/>
  <description>Installation and upgrade tests for icinga2 Debian packages</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <hudson.model.ParametersDefinitionProperty>
      <parameterDefinitions>
        <hudson.model.StringParameterDefinition>
          <name>architecture</name>
          <description></description>
          <defaultValue>amd64</defaultValue>
          <trim>false</trim>
        </hudson.model.StringParameterDefinition>
      </parameterDefinitions>
    </hudson.model.ParametersDefinitionProperty>
  </properties>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.41">
      <project>icinga2-binaries/architecture=$architecture</project>
      <filter>*.deb</filter>
      <target>artifacts/</target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <fallbackToLastSuccessful>true</fallbackToLastSuccessful>
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
        <allowUpstreamDependencies>false</allowUpstreamDependencies>
      </selector>
      <flatten>true</flatten>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
    <hudson.tasks.Shell>
      <command># sadly piuparts always returns with exit code 1 :((
sudo piuparts_wrapper ${PWD}/artifacts/*.deb || true</command>
    </hudson.tasks.Shell>
    <hudson.tasks.Shell>
      <command>piuparts_tap piuparts.txt &gt; piuparts.tap</command>
    </hudson.tasks.Shell>
  </builders>
  <publishers>
    <org.tap4j.plugin.TapPublisher plugin="tap@2.2.1">
      <testResults>piuparts.tap</testResults>
      <failIfNoResults>false</failIfNoResults>
      <failedTestsMarkBuildAsFailure>false</failedTestsMarkBuildAsFailure>
      <outputTapToConsole>false</outputTapToConsole>
      <enableSubtests>true</enableSubtests>
      <discardOldReports>false</discardOldReports>
      <todoIsFailure>false</todoIsFailure>
      <includeCommentDiagnostics>true</includeCommentDiagnostics>
      <validateNumberOfTests>false</validateNumberOfTests>
      <planRequired>true</planRequired>
      <verbose>true</verbose>
      <showOnlyFailures>false</showOnlyFailures>
      <stripSingleParents>false</stripSingleParents>
      <flattenTapResult>false</flattenTapResult>
      <skipIfBuildNotOk>false</skipIfBuildNotOk>
    </org.tap4j.plugin.TapPublisher>
    <hudson.tasks.ArtifactArchiver>
      <artifacts>piuparts.*</artifacts>
      <allowEmptyArchive>false</allowEmptyArchive>
      <onlyIfSuccessful>false</onlyIfSuccessful>
      <fingerprint>false</fingerprint>
      <defaultExcludes>true</defaultExcludes>
      <caseSensitive>true</caseSensitive>
    </hudson.tasks.ArtifactArchiver>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.36">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
      <disableDeferredWipeout>false</disableDeferredWipeout>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
</project>
EOF
    echo "create olsrd jobs from /tmp/vagrant/ ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd-source < /tmp/vagrant/olsrd-source_config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd-binaries < /tmp/vagrant/olsrd-binaries_config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd-piuparts < /tmp/vagrant/olsrd-piuparts_config.xml
    echo "create olsrd2 jobs from /tmp/vagrant/ ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd2-source < /tmp/vagrant/olsrd2-source_config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd2-binaries < /tmp/vagrant/olsrd2-binaries_config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job olsrd2-piuparts < /tmp/vagrant/olsrd2-piuparts_config.xml
    echo "create icinga2 jobs from /tmp/vagrant/ ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job icinga2-source < /tmp/vagrant/icinga2-source_config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job icinga2-binaries < /tmp/vagrant/icinga2-binaries_config.xml
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} create-job icinga2-piuparts < /tmp/vagrant/icinga2-piuparts_config.xml
    echo "delete default jenkins-debian-glue jobs ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} delete-job jenkins-debian-glue-source jenkins-debian-glue-binaries jenkins-debian-glue-piuparts
    echo "start building job olsrd-source ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth build olsrd-source
    echo "start building job olsrd2-source ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth build olsrd2-source
    echo "start building job icinga2-source ..."
    java -jar /var/cache/jenkins/war/WEB-INF/jenkins-cli.jar -s "http://localhost:8080" -auth jenkins-debian-glue:${jenkinspass} -noKeyAuth build icinga2-source

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

    echo "all done."
    exit 0
  SHELL
end
