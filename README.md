# Learning BOSH - Geode Cluster Release

This is just a personal repository, used to acquire some basic [BOSH](http://bosh.io/) knowledge. It should not be used as an official reference or as the source of any truth, it's just something I'm using for learning and testing purposes, it could be used for the same goal by others as well and that's why I'm making it public. Keep in mind that, leaving aside the quotes extracted from the official documentation links, there could be errors and/or mistakes within my line of thouhgt and how I implement the release itself.

The steps, based on [Creating a Release](https://bosh.io/docs/create-release.html), show how to create a [BOSH](http://bosh.io/) release from scratch and how to deploy it to a local BOSH-Lite environment for further testing. The main idea is to have a small [Apache Geode Cluster](http://geode.apache.org/) up and running. We are using BOSH-Lite on VirtualBox for the sake of simplicity, but the release could be easily deployed to any of the supported IaaS providers since, after all, that's one of the magical things that [BOSH](http://bosh.io/) can do for us. 

For a complete reference about [BOSH](http://bosh.io/) terminology, please refer to the [Official BOSH Documentation](https://bosh.io/docs).

---

### INDEX

* [BOSH-Lite](#boshLite)
* [Getting Started](#gettingStarted)
* [Building the Packages](#packages)
* [Implementing the Jobs](#jobs)
* [Creating & Uploading the Release](#release)
* [Creating the Deployment Manifest](#manifest)
* [Deploying and Executing the Smoke Tests](#deploy)

---

### <a id="boshLite"></a>BOSH-Lite

The full (and official) set of instructions to install BOSH-Lite on Virtual Box can be found in the [bosh-deployment](https://github.com/cloudfoundry/bosh-deployment) repository.

First things first: below is the folder structure we're going to use going forward, it's not mandatory neither required, just a personal preference to keep things organized, I'll refer to it quite often within this guide.

```bash
workspace
├── config
│   └── (...)
├── downloads
│   └── (...)
└── git
    ├── (...)
    └── (...)
```

When you are getting started with something new, sometimes it is useful to have some automated script to delete everything and start from scratch (specially if you don't want to memorize all of the steps required), so I've created a simple one do so:

```bash
#!/bin/bash
set -x

WORKSPACE_DIRECTORY=/workspace
sudo route add -net 10.244.0.0/16 192.168.50.6
BOSH_DEPLOYMENT_ROOT_DIRECTORY=$WORKSPACE_DIRECTORY/git/bosh-deployment

# Clean Previous Settings
rm -Rf ~/.bosh
rm -Rf ~/VirtualBox\ VMs/
ssh-keygen -R 192.168.50.6
rm -Rf $BOSH_DEPLOYMENT_ROOT_DIRECTORY
rm -f $WORKSPACE_DIRECTORY/config/creds.yml
rm -f $WORKSPACE_DIRECTORY/config/state.json

# Install Bosh Lite
git clone https://github.com/cloudfoundry/bosh-deployment $BOSH_DEPLOYMENT_ROOT_DIRECTORY

bosh create-env \
  $BOSH_DEPLOYMENT_ROOT_DIRECTORY/bosh.yml \
  -o $BOSH_DEPLOYMENT_ROOT_DIRECTORY/virtualbox/cpi.yml \
  -o $BOSH_DEPLOYMENT_ROOT_DIRECTORY/virtualbox/outbound-network.yml \
  -o $BOSH_DEPLOYMENT_ROOT_DIRECTORY/bosh-lite.yml \
  -o $BOSH_DEPLOYMENT_ROOT_DIRECTORY/bosh-lite-runc.yml \
  -o $BOSH_DEPLOYMENT_ROOT_DIRECTORY/jumpbox-user.yml \
  --state $WORKSPACE_DIRECTORY/config/state.json \
  --vars-store $WORKSPACE_DIRECTORY/config/creds.yml \
  -v internal_ip=192.168.50.6 \
  -v internal_gw=192.168.50.1 \
  -v internal_cidr=192.168.50.0/24 \
  -v outbound_network_name=NatNetwork \
  -v director_name="Bosh Lite Director"

# Alias and Certs
bosh int $WORKSPACE_DIRECTORY/config/creds.yml --path /jumpbox_ssh/private_key > ~/.ssh/bosh-virtualbox.key
chmod 600 ~/.ssh/bosh-virtualbox.key
bosh -e 192.168.50.6 --ca-cert <(bosh int $WORKSPACE_DIRECTORY/config/creds.yml --path /director_ssl/ca) alias-env bosh-lite

# Upload Stemcell & Cloud Config
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh int $WORKSPACE_DIRECTORY/config/creds.yml --path /admin_password`
bosh -e bosh-lite update-cloud-config $BOSH_DEPLOYMENT_ROOT_DIRECTORY/warden/cloud-config.yml
bosh -e bosh-lite upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent
```

Remember that you need to pause your Virtual Box VM whenever your machine goes to sleep or gets rebooted, otherwise the VM will be halted by the OS and you will need to install your software again from scratch.

The following script could be used to do some management on the VM without manually accesing the VirtualBox UI:

```bash
#!/bin/bash
set -x

WORKSPACE_DIRECTORY=/workspace
BOSH_LITE_VM_ID=$(cat $WORKSPACE_DIRECTORY/config/state.json | python -c "import json,sys;obj=json.load(sys.stdin);print obj['current_vm_cid'];")

case $1 in
  ssh)
    ssh -i ~/.ssh/bosh-virtualbox.key jumpbox@192.168.50.6
    ;;
  pause)
    echo "Pausing Bosh_Lite VM with ID $BOSH_LITE_VM_ID..."
    VBoxManage controlvm $BOSH_LITE_VM_ID savestate
    echo "Pausing Bosh_Lite VM with ID $BOSH_LITE_VM_ID... Done!."
    ;;
  resume)
    echo "Resuming Bosh_Lite VM with ID $BOSH_LITE_VM_ID..."
    VBoxManage startvm $BOSH_LITE_VM_ID --type=headless
    echo "Resuming Bosh_Lite VM with ID $BOSH_LITE_VM_ID... Done!"
    ;;
  *)
    echo "Usage: bosh_vm {ssh|pause|resume}" ;;
esac
```

---

### <a id="gettingStarted"></a>Getting Started

We have a BOSH-Lite environment installed on our local Virtual Box that we can use to play now, so it's time to get started. 

The first step when creating a release is to initialize the release directory itself, which can be achieved by using the BOSH CLI [init-release](https://bosh.io/docs/cli-v2#init-release) command:  `$ bosh init-release --git --dir=geode-bosh-release`. Once that's done, we should have the following directory structure and we're ready to start with the actual work:

```bash
workspace
├── config
│   ├── creds.yml
│   └── state.json
├── downloads
└── git
    ├── bosh-deployment
    │   └── (...)
    └── geode-bosh-release
        ├── config
        │   ├── blobs.yml
        │   └── final.yml
        ├── jobs
        ├── packages
        └── src
```

With the release root directory created, we now want to build our dependencies graph and start defining which jobs and packages we need. For this particular example we know that we'll have a small [Apache Geode](http://geode.apache.org/) Cluster, composed of locators and servers. [Apache Geode](http://geode.apache.org/) is written in Java, so we also know that we need to have Oracle Java installed on every single VM for the member to work properly. We are also going to build [Apache Geode](http://geode.apache.org/) from its source code instead of using the available binaries, and to be able to do so we'll need to have [Gradle](https://gradle.org/) installed on the compillation VMs as well.

Once the deploy finishes we'll want to verify that the cluster and its components are up and running, so we are going to configure a [BOSH errand](https://bosh.io/docs/terminology.html#errand) and use the [Apache Geode GFSH Tool](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html) to implemente some smoke tests on the cluster.

Below are some [BOSH concepts](https://bosh.io/docs/terminology.html), extracted from the official documentation:
> A release job represents a specific chunk of work that the release performs. Jobs describe pieces of the service or application you are releasing.

> A package is a component of a BOSH release that contains a packaging spec file and a packaging script. Each package also references source code or pre-compiled software that you store in the src directory of a BOSH release directory. Packages provide source code and dependencies to jobs.

> An errand is a short-lived job that an operator can run multiple times after the deploy finishes. Examples: smoke tests, comprehensive test suites, CF service broker binding and unbinding.

Putting it all together, we have three jobs and three packages:

##### Packages
| Package | Name | Compile Dependencies | Runtime Dependencies
| --- | --- | --- | --- |
| Java | java | NONE | NONE
| Gradle | gradle | NONE | java
| Apache Geode | geode | gradle, java | java

##### Jobs
| Job | Name | Runtime Dependencies
| --- | --- | --- |
| Locator | locator | java, geode
| Server | server | java, geode
| Smoke Tests | smoke-tests | java, geode

---

### <a id="packages"></a>Building the Packages

A [package](https://bosh.io/docs/packages.html) is a component of a [BOSH](http://bosh.io/) release that contains a packaging spec file and a packaging script. Each package also references source code or pre-compiled software that you store in the src directory of a [BOSH](http://bosh.io/) release.

[BOSH](http://bosh.io/) comes with a handy command to create the default skeleton for each package, [generate-package](https://bosh.io/docs/cli-v2#generate-package), so we should execute it for each one of our packages under the release directory:

```bash
$ bosh generate-package java
$ bosh generate-package gradle
$ bosh generate-package geode
$ tree
.
├── config
│   ├── blobs.yml
│   └── final.yml
├── jobs
├── packages
│   ├── geode
│   │   ├── packaging
│   │   └── spec
│   ├── gradle
│   │   ├── packaging
│   │   └── spec
│   └── java
│       ├── packaging
│       └── spec
└── src
```

We're building a dev release for testing purposes only so we can safely use a local blobstore to store our [blobs](http://bosh.io/docs/reference/blobs.html) instead of an external one, but keep in mind that this approach doesn't work when creating a final release, for that particular scenario we would need to upload the blobs to an actual S3 blobstore.

That said, we need to download the required files that our packages are going to use into the `downloads` folder, configure our local blobstore, add the blobs and inform [BOSH](http://bosh.io/) where the blobs are through the BOSH CLI [add-blob](https://bosh.io/docs/cli-v2#add-blob) command. For this release we're going to use [apache-geode-src-1.1.1.tar.gz](http://ftp.heanet.ie/mirrors/www.apache.org/dist/geode/1.1.1/apache-geode-src-1.1.1.tar.gz), [gradle-3.5-bin.zip](https://services.gradle.org/distributions/gradle-3.5-bin.zip) and [jdk-8u131-linux-x64.tar.gz](http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.tar.gz).

```bash
tree -L 2
.
├── config
│   ├── creds.yml
│   ├── custom-cloud-config.yml
│   └── state.json
├── downloads
│   ├── apache-geode-src-1.1.1.tar.gz
│   ├── gradle-3.5-bin.zip
│   └── jdk-8u131-linux-x64.tar.gz
└── git
    ├── bosh-deployment
    └── geode-bosh-release
```

The blobstore can be configured through the `final.yml` file, and to inform [BOSH](http://bosh.io/) where the blobs are located we must issue the BOSH CLI [add-blob](https://bosh.io/docs/cli-v2#add-blob) command:

*config/final.yml*
```
---
blobstore:
  provider: local
  options:
    blobstore_path: /tmp/bosh/blobs
name: geode-bosh
```

```bash
$ export WORKSPACE_DIRECTORY=/workspace
$ bosh add-blob $WORKSPACE_DIRECTORY/downloads/gradle-3.5-bin.zip gradle/gradle-3.5-bin.zip
$ bosh add-blob $WORKSPACE_DIRECTORY/downloads/jdk-8u131-linux-x64.tar.gz java/jdk-8u131-linux-x64.tar.gz
$ bosh add-blob $WORKSPACE_DIRECTORY/downloads/apache-geode-src-1.1.1.tar.gz geode/apache-geode-src-1.1.1.tar.gz
```

Now that the blobs have been added to the blobstore, it's time to define the content of each package through the `spec` file, along with the instructions to install it using the `packaging` script. The first two (Java and Gradle) are straightforward so we'll start with them, the last one requires some extra work.

#### Java

We need to update the `spec` file with the relevant information and implement the `packaging` script, which will be used by [BOSH](http://bosh.io/) to install the package on the VMs. The implementation just needs to extract the contents of the Java binary distribution into the installation folder assigned by [BOSH](http://bosh.io/), which can be referenced from the script through the `BOSH_INSTALL_TARGET` variable.

*packages/java/spec*
```
---
name: java
dependencies: []
files:
- java/jdk-8u131-linux-x64.tar.gz # http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.tar.gz
```

*packages/java/packaging*
```bash
set -e -x
echo "Installing Java..."
tar xzf java/jdk-8u131-linux-x64.tar.gz -C ${BOSH_INSTALL_TARGET} --strip-components=1
echo "Installing Java... Done!"
```

#### Gradle

Again, and like the previous package, we have to fill the details within the `spec` file and implement the `packaging` script to uncrompress the content of the binary file into the assigned installation folder: `BOSH_INSTALL_TARGET`.

*packages/gradle/spec*
```
---
name: gradle
dependencies: []
files: []
- gradle/gradle-3.5-bin.zip # https://services.gradle.org/distributions/gradle-3.5-bin.zip
```

*packages/gradle/packaging*
```bash
set -e -x
echo "Installing Gradle..."
tar xzf gradle/gradle-3.5-bin.zip -C ${BOSH_INSTALL_TARGET} --strip-components=1
echo "Installing Gradle... Done!"
```

#### Apache Geode

Binary downloads for [Apache Geode](http://geode.apache.org/) are only provided for the convenience of the users and are not official releases, so we need to actually download and build the source code from scratch. It would be way easier to just download and uncompress the binary distribution like we did before for the other two packages, of course, but doing so would prevent us from learning how to actually write some custom scripts to compile and deploy packages instead of just uncompressing already compiled code.

Considering that [Apache Geode](http://geode.apache.org/) is entirely written in Java and that it also provides a clear way to build the framework from the source code, this could be also used as a great example when dealing with our own applications as well. In the normal case where the package is an actual application written by us, however, the source code would be placed into the `src` folder instead of treat it as a blob (we're not doing it right here because the source code is huge).

The blob with the source code is already downloaded into our local blobstore, so we just need to fill the details within the `spec` file (note that this package has two compile time dependencies) and, just like before, implement the `packaging` script. The implementation for this script will be slighty different, we need to uncrompress the source code into a temporary folder, [build Apache Geode](http://geode.apache.org/docs/guide/11/getting_started/installation/install_standalone.html) and move the result to the folder assigned by [BOSH](http://bosh.io/): `BOSH_INSTALL_TARGET`.

*packages/geode/spec*
```
---
name: geode
dependencies:
- java
- gradle
files:
- geode/apache-geode-src-1.1.1.tar.gz #http://ftp.heanet.ie/mirrors/www.apache.org/dist/geode/1.1.1/apache-geode-src-1.1.1.tar.gz
```

*packages/geode/packaging*
```bash
set -e -x
echo "Installing Apache Geode..."
export JAVA_HOME=/var/vcap/packages/java
export PATH=$PATH:$JAVA_HOME/bin:/var/vcap/packages/gradle/bin
tar xzf geode/apache-geode-src-1.1.1.tar.gz
pushd apache-geode-src-1.1.1
 ./gradlew build -Dskip.tests=true
 cp -a geode-assembly/build/install/apache-geode/* ${BOSH_INSTALL_TARGET}
popd
echo "Installing Apache Geode... Done!"
```

---

### <a id="jobs"></a>Implementing the Jobs

A [job](https://bosh.io/docs/jobs.html) represents a specific chunk of work that the release performs. It typically includes metadata that specifies available configuration options, [ERB](https://apidock.com/ruby/ERB) configuration files, a [Monit](https://mmonit.com/) file that describes how to start, stop and monitor processes, start and stop scripts for each process and additional hook scripts.

We previously identified two main jobs within our release (locator and server), and one special type of job (errand) that will run after the deploy finishes to verify that our primary jobs are working as expected in terms of functionality ([Monit](https://mmonit.com/)  already takes care of verifying the job is up and running).

The [Locator](http://geode.apache.org/docs/guide/11/configuring/running/running_the_locator.html) is a [Geode](http://geode.apache.org/) process that tells new, connecting members where running members are located and provides load balancing for server use. The [Server](http://geode.apache.org/docs/guide/11/configuring/running/running_the_cacheserver.html), on the other hand, is a process that runs as a long-lived, configurable member of a client/server system; it is used primarily for hosting long-lived data regions and running standard [Geode](http://geode.apache.org/) processes.

Considering that the focus is to show how to create a [BOSH](http://bosh.io/) release, and not how to configure and use [Apache Geode](http://geode.apache.org/), we won't go deep into the details about the configuration and functionatilies offered by this framework. Our cluster will be up and accepting client connections once we're done anyway, it'll use [Geode PDX Serialization](http://geode.apache.org/docs/guide/11/developing/data_serialization/gemfire_pdx_serialization.html) to avoid deploying custom model jars to the servers, will be persistent (data will remain there after restarts) and it'll have two [Data Regions](http://geode.apache.org/docs/guide/11/basic_config/data_regions/chapter_overview.html): one [partitioned](http://geode.apache.org/docs/guide/11/developing/partitioned_regions/chapter_overview.html) and one [replicated](http://geode.apache.org/docs/guide/11/developing/distributed_regions/chapter_overview.html).

That said, it's time for us to create and configure the three jobs that are part of our release: the Locator, the Cache Server and the Smoke-Tests. [BOSH](http://bosh.io/) comes with a handy command to create the default skeleton for each job, [generate-job](https://bosh.io/docs/cli-v2#generate-job), so we should execute it for each one of our jobs under the release directory:

```bash
$ bosh generate-job locator
$ bosh generate-job server
$ bosh generate-job smoke-tests
$ tree
.
├── config
│   ├── blobs.yml
│   └── final.yml
├── jobs
│   ├── locator
│   │   ├── monit
│   │   ├── spec
│   │   └── templates
│   └── server
│   │   ├── monit
│   │   ├── spec
│   │   └── templates
│   └── smoke-tests
│       ├── monit
│       ├── spec
│       └── templates
├── packages
│   ├── geode
│   │   ├── packaging
│   │   └── spec
│   ├── gradle
│   │   ├── packaging
│   │   └── spec
│   └── java
│       ├── packaging
│       └── spec
└── src
```

By default [BOSH](http://bosh.io/) creates two files and one directory under each job's root folder:
* The `spec` file defines job metadata (like properties, dependencies, and files used by the job).
* The `monit` file is used to monitor, start and stop the job itself through [Monit](https://mmonit.com/). There are several advanced options to monitor processes using monit, for the sake of simplicity we're going to use the most basic one.
* The `templates` directory, which will contain several files that can be used to configure, control, and manage the job.

#### Locator

The locator is a [Geode](http://geode.apache.org/) process that tells new, connecting members where running members are located and provides load balancing for server use.

We need to tell [BOSH](http://bosh.io/) how to actively monitor the locator process through the `monit` file and how to manage its state through the `ctl` script. The locator has tons of configuration options that can be used to modify its behavior, but for the sake of simplicity we'll just set specific values for the most common ones and allow the operator to manually configure some of them using the deployment manifest, which passes the instance-specific information to the VM through the agent.

Sometimes it's useful to do some work before or after the job is started, and for these scenarios where we need to hook custom logic we can make use of the [job-lifecycle](http://bosh.io/docs/job-lifecycle.html) scripts. Having that in mind, we'll define our own `post-start` script, which allows the job to execute any additional commands against a machine and/or persistent data before considering the release job as successfully started.

*jobs/locator/spec*

The Locator process requires Java and [Apache Geode](http://geode.apache.org/) to be installed on the VM so we use the `dependencies` section to let [BOSH](http://bosh.io/) know about it. The `templates` section specifies which [ERB](https://apidock.com/ruby/ERB) templates the job has, along with the path where the resulting file will be place on the destination VM.

It's recommend that every member belonging to a [Geode](http://geode.apache.org/) cluster knows about all of the available locators within that cluster, so we need to pass the list of configured locators when starting every single [Geode](http://geode.apache.org/) process. To be able to do this in a dynamic, ordered and manageable fashion we'll use [BOSH links](http://bosh.io/docs/links.html). To instruct [BOSH](http://bosh.io/) that the job needs to import some information we use the `consumes` statement, and to let [BOSH](http://bosh.io/) know that the job is actually exporting some information we use `provides`. [BOSH links](http://bosh.io/docs/links.html) automatically export some instance methods and properties by default, to also export our own defined properties we must declare them through the `properties` statement under the `provides` section.

As previously stated, there are tons of properties that can be used to modify the locator behavior and configuration, for the sake of simplicity we're going to allow the operator to modify only some of them through the `properties` section, and we'll use defaults for the rest. For a complete list of available properties and what they do please refer to the [Apache Geode Reference](http://geode.apache.org/docs/guide/11/reference/topics/gemfire_properties.html).

```
---
name: locator

templates:
  ctl.erb: bin/ctl
  post-start.erb: bin/post-start
  locator.properties.erb: config/locator.properties

packages:
- java
- geode

consumes:
- name: locator
  type: locator

provides:
- name: locator
  type: locator
  properties:
  - http.port
  - locator.port

properties:
  jmx.port:
    description: The port at which the JMX Manager will listen to for client connections.
    default: 1099
  http.port:
    description: Port at which the embedded HTTP service listens on.
    default: 7070
  locator.port:
    description: Port the locator will listen on.
    default: 10334
  log.level:
    description: Log level that will be used by the locator.
    default: config
```

*jobs/locator/monit*

The `monit` file, in its most basic form, specifies the process ID (pid) file for the job and defines wich commands should be ran to start and stop the process, along with the user required to run the job.

```bash
check process locator
  with pidfile /var/vcap/store/locator/vf.gf.locator.pid
  start program "/var/vcap/jobs/locator/bin/ctl start"
  stop program "/var/vcap/jobs/locator/bin/ctl stop"
  group vcap
```

*jobs/locator/templates/ctl.erb*

This template is used by [Monit](https://mmonit.com/) to start and stop the process on the VM when needed. [BOSH](http://bosh.io/) defines several standard folders and options to implement it, but the actual implementation is strongly tied to the job itself, along with the commands and packages required to manage it. 

Keep in mind that each template file is evaluated with [ERB](https://apidock.com/ruby/ERB) before being sent to each instance. As an example, this template will be processed by [ERB](https://apidock.com/ruby/ERB) and placed on the VM under the `/var/vcap/jobs/locator/bin/` folder (as specified by our `spec` file). Templates have access to merged job property values, built by merging default property values and operator specified property values in the deployment manifest, which gives us a huge power and configuration dynamism. For practical purposes, and as a simple example, this allow us to use the IP assigned to the VM through `<%= "#{spec.id}" %>`, custom job defined properties through `<%= p('propertyName') %>`, and several other use cases, the possibilities are endless.

Remember when I said that every single [Geode](http://geode.apache.org/) process requires the full list of locators at the start?, well, it is the perfect usage of [BOSH links](http://bosh.io/docs/links.html) links + [ERB](https://apidock.com/ruby/ERB) templates.

```bash
#!/bin/bash
set -e -x

WORK_DIR=/var/vcap/store/locator
LOG_DIR=/var/vcap/sys/log/locator
CONF_DIR=/var/vcap/jobs/locator/config
LOCATOR_NAME=locator_<%= "#{spec.id}" %>
LOCATOR_BIND_ADDRESS=<%= "#{spec.address}" %>

export JAVA_HOME=/var/vcap/packages/java
export GEODE_HOME=/var/vcap/packages/geode
export PATH=$PATH:$JAVA_HOME/bin:$GEODE_HOME/bin

mkdir -p $LOG_DIR $WORK_DIR
touch $LOG_DIR/ctl.std{out,err}.log
touch $LOG_DIR/post-start.std{out,err}.log
chown -R vcap:vcap $LOG_DIR $WORK_DIR

exec > >(tee --append "$LOG_DIR"/ctl.stdout.log )
exec 2> >(tee --append "$LOG_DIR"/ctl.stderr.log)

case $1 in

  start)
    echo [`date '+%F %T'`]: "Starting locator $LOCATOR_NAME[<%= p('locator.port') %>]..."

    gfsh start locator \
      --force=true --connect=false \
      --log-level=<%= p('log.level') %> \
      --enable-cluster-configuration=true \
      --initial-heap=256m --max-heap=256m \
      --dir=$WORK_DIR --name=$LOCATOR_NAME \
      --properties-file=$CONF_DIR/locator.properties \
      --port=<%= p('locator.port') %> --bind-address=$LOCATOR_BIND_ADDRESS --mcast-port=0 \
      --locators=<%= link('locator').instances.map { |l| "#{l.address}[#{p('locator.port')}]"}.join(",") %> \
      --J=-Dgemfire.http-service-port=<%= p('http.port') %> --J=-Dgemfire.http-service-bind-address=$LOCATOR_BIND_ADDRESS \
      --J=-Dgemfire.jmx-manager=true --J=-Dgemfire.jmx-manager-start=true --J=-Dgemfire.jmx-manager-port=<%= p('jmx.port') %> --J=-Djava.rmi.server.hostname=$LOCATOR_BIND_ADDRESS

    echo [`date '+%F %T'`]: "Starting locator $LOCATOR_NAME[<%= p('locator.port') %>]... Done!"
    ;;

  stop)
    echo [`date '+%F %T'`]: "Stopping locator $LOCATOR_NAME[<%= p('locator.port') %>]..."
    gfsh stop locator --dir=$WORK_DIR
    echo [`date '+%F %T'`]: "Stopping locator $LOCATOR_NAME[<%= p('locator.port') %>]... Done!"
    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;
esac
```

*jobs/locator/templates/locator.properties.erb*

This file is used to further configure the [Geode](http://geode.apache.org/) Locator, empty in our release, but left within it anyway as an easy way to configure the internals of the Locator without requiring a modification to the `ctl` script or the `spec` file.

```
### Geode Properties using default values ###
```

*jobs/locator/templates/post-start.erb*

Another [ERB](https://apidock.com/ruby/ERB) template that will be called, according to the [job-lifecycle](https://bosh.io/docs/job-lifecycle.html) execution order, after [Monit](https://mmonit.com/) starts our Locator job. The content shouldn't take much of our attention since it's related mainly to [Apache Geode](http://geode.apache.org/) and not to [BOSH](http://bosh.io/) itself. It basically verifies that the Locator is receiving connections by using the [GFSH Tool](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html), and that the [Pulse Web Application](http://geode.apache.org/docs/guide/11/tools_modules/pulse/chapter_overview.html) running embedded into the Locator is also accepting requests.

```bash
#!/bin/bash
set -e

export JAVA_HOME=/var/vcap/packages/java
export GEODE_HOME=/var/vcap/packages/geode
export PATH=$PATH:$JAVA_HOME/bin:$GEODE_HOME/bin

RETRIES=10
SLEEP_TIME=5
LOCATOR_NAME=locator_<%= "#{spec.id}" %>
LOCATOR_BIND_ADDRESS=<%= "#{spec.address}" %>
LOCATOR_CONNECTION_STRING=$LOCATOR_BIND_ADDRESS[<%= p('locator.port') %>]
PULSE_WEB_APP_CONNECTION_STRING=http://$LOCATOR_BIND_ADDRESS:<%= p('http.port') %>/pulse/login.html

echo [`date '+%F %T'`]: "Verifying locator $LOCATOR_CONNECTION_STRING..."

for i in $(seq 1 "$RETRIES"); do

    if gfsh -e "connect --locator=$LOCATOR_CONNECTION_STRING"; then
      sleep "$SLEEP_TIME"
      PULSE_STATUS=$(curl -s --head -w %{http_code} $PULSE_WEB_APP_CONNECTION_STRING -o /dev/null)

      if [[ $PULSE_STATUS == 200 ]]; then
        echo [`date '+%F %T'`]: "Verifying locator $LOCATOR_CONNECTION_STRING... Done!."
        exit 0
      else
        echo [`date '+%F %T'`]: "Verifying locator $LOCATOR_CONNECTION_STRING... Failure (Pulse)."
        exit 1
      fi
    else
      sleep "$SLEEP_TIME"
    fi
done

echo [`date '+%F %T'`]: "Verifying locator $LOCATOR_CONNECTION_STRING... Failure (Locator)."
exit 1
```

#### Server

The [Geode](http://geode.apache.org/) server is a process that runs as a long-lived, configurable member of a client/server system.

Just like we did with the locator, we need to tell [BOSH](http://bosh.io/) how to monitor the process through the `monit` file and how to manage its state through the `ctl` script. The server also has tons of configuration options that can be used to modify its behavior but, again and for the sake of simplicity, we'll use just the most common ones and allow the operator to manually configure some of these properties using the deployment manifest. We'll also define a `post-start` script to execute some sanity checks before the job can be considered as up and running.

*jobs/server/spec*

The Server process requires Java and [Apache Geode](http://geode.apache.org/) to be installed on the VM so we use the `dependencies` section to let [BOSH](http://bosh.io/) know about it.

As said before, we need to know the full list of configured locators within the cluster when starting the server, so we'll use [BOSH links](http://bosh.io/docs/links.html) again to get this information from the deployment manifest. The list of properties to further configure the server will be set in the `server.properties` file, empty again to use the default values. The `cache.xml` file is used to configure several functional aspects of the server (like regions, disk stores, queues, listeners, etc.), the one included is a basic example with two persistent regions, along with the [Automatic Reflection-Based PDX Serialization](http://geode.apache.org/docs/guide/11/developing/data_serialization/auto_serialization.html). We also define some properties with default values (heap memory to use, tcp port to listen for client connections, etc) that can be further overridden by the operator when implementing the deployment manifest.

```
---
name: server

templates:
  ctl.erb: bin/ctl
  post-start.erb: bin/post-start
  cache.xml.erb: config/cache.xml
  server.properties.erb: config/server.properties

packages:
- java
- geode

consumes:
- name: locator
  type: locator

provides:
- name: server
  type: server
  properties:
  - http.port
  - server.port

properties:
  http.port:
    description: Port at which the embedded HTTP service listens on.
    default: 7070
  server.port:
    description: Port the server will listen on.
    default: 40404
  log.level:
    description: Log level that will be used by the server.
    default: config
  newGen.size:
    description: Heap memory that will be assigned to the young generation space (megabyte).
    default: 128
  oldGen.size:
    description: Heap memory that will be assigned to the old generation space (megabyte).
    default: 1024
```

*jobs/server/monit*

```bash
check process server
  with pidfile /var/vcap/store/server/vf.gf.server.pid
  start program "/var/vcap/jobs/server/bin/ctl start"
  stop program "/var/vcap/jobs/server/bin/ctl stop"
  group vcap
```

*jobs/server/templates/cache.xml.erb*

The full details about how to configure this file can be found in the [Official Geode User Gudie](http://geode.apache.org/docs/guide/11/reference/topics/chapter_overview_cache_xml.html).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<cache
    xmlns="http://geode.apache.org/schema/cache"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://geode.apache.org/schema/cache http://geode.apache.org/schema/cache/cache-1.0.xsd"
    version="1.0">

  <cache-server/>

  <disk-store name="diskStore"/>

  <pdx persistent="true">
    <pdx-serializer>
      <class-name>org.apache.geode.pdx.ReflectionBasedAutoSerializer</class-name>
      <parameter name="classes">
        <string>com.company.model.*</string>
      </parameter>
    </pdx-serializer>
  </pdx>

  <region name="replicatedRegion">
    <region-attributes refid="REPLICATE_PERSISTENT" statistics-enabled="true" disk-store-name="diskStore"/>
  </region>

  <region name="partitionedRegion">
    <region-attributes refid="PARTITION_REDUNDANT_PERSISTENT" statistics-enabled="true" disk-store-name="diskStore">
      <partition-attributes redundant-copies="2"/>
    </region-attributes>
  </region>
</cache>
```

*jobs/server/templates/ctl.erb*

This template is used by [Monit](https://mmonit.com/) to start and stop the process on the VM when needed. We use [Geode GFSH](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html) to start and stop the server, pay special attention to the folders and techniques used to redirect the standard output and error streams, along with the properties extracted from the deployment manifest and the path to the configuration files used.

```bash
#!/bin/bash
set -e -x

WORK_DIR=/var/vcap/store/server
LOG_DIR=/var/vcap/sys/log/server
CONF_DIR=/var/vcap/jobs/server/config
SERVER_NAME=server_<%= "#{spec.id}" %>
SERVER_BIND_ADDRESS=<%= "#{spec.address}" %>
HEAP_OPTIONS="--J=-Xmn<%= p('newGen.size') %>m --J=-Xmx<%= p('oldGen.size') %>m --J=-Xms<%= p('oldGen.size') %>m --J=-XX:+AlwaysPreTouch"
JAVA_GC_OPTS="--J=-XX:+UseParNewGC --J=-XX:+UseConcMarkSweepGC --J=-XX:CMSInitiatingOccupancyFraction=70 --J=-XX:+UseCMSInitiatingOccupancyOnly --J=-XX:+DisableExplicitGC --J=-XX:+CMSClassUnloadingEnabled"
JAVA_GC_PRINT_OPTS="--J=-verbose:gc --J=-Xloggc:$SERVER_NAME-gc.log --J=-XX:+PrintGCDateStamps --J=-XX:+PrintGCDetails --J=-XX:+PrintTenuringDistribution --J=-XX:+PrintGCApplicationConcurrentTime --J=-XX:+PrintGCApplicationStoppedTime"

export JAVA_HOME=/var/vcap/packages/java
export GEODE_HOME=/var/vcap/packages/geode
export PATH=$PATH:$JAVA_HOME/bin:$GEODE_HOME/bin

mkdir -p $LOG_DIR $WORK_DIR
touch $LOG_DIR/ctl.std{out,err}.log
touch $LOG_DIR/post-start.std{out,err}.log
chown -R vcap:vcap $LOG_DIR $WORK_DIR

exec > >(tee --append "$LOG_DIR"/ctl.stdout.log )
exec 2> >(tee --append "$LOG_DIR"/ctl.stderr.log)

case $1 in

  start)
    echo [`date '+%F %T'`]: "Starting server $SERVER_NAME[<%= p('server.port') %>]..."

    gfsh start server \
      --force=true \
      --use-cluster-configuration=true \
      --log-level=<%= p('log.level') %> \
      --dir=$WORK_DIR --name=$SERVER_NAME \
      --properties-file=$CONF_DIR/server.properties --cache-xml-file=$CONF_DIR/cache.xml \
      --server-port=<%= p('server.port') %> --server-bind-address=$SERVER_BIND_ADDRESS --mcast-port=0 \
      --start-rest-api=true --http-service-port=<%= p('http.port') %> --http-service-bind-address=$SERVER_BIND_ADDRESS \
      --locators=<%= link('locator').instances.map { |l| "#{l.address}[#{link('locator').p('locator.port')}]"}.join(",") %> \
      --J=-Dgemfire.statistic-sample-rate=1000 --J=-Dgemfire.statistic-sampling-enabled=true --J=-Dgemfire.statistic-archive-file=$WORK_DIR/$SERVER_NAME.gfs $HEAP_OPTIONS $JAVA_GC_OPTS $JAVA_GC_PRINT_OPTS

    echo [`date '+%F %T'`]: "Starting server $SERVER_NAME[<%= p('server.port') %>]... Done!"
    ;;

  stop)
    echo [`date '+%F %T'`]: "Stopping server $SERVER_NAME[<%= p('server.port') %>]..."
    gfsh stop server --dir=$WORK_DIR
    echo [`date '+%F %T'`]: "Stopping server $SERVER_NAME[<%= p('server.port') %>]... Done!"
    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;
esac
```

*jobs/server/templates/post-start.erb*

The content shouldn't take much of our attention since it's related mainly to [Apache Geode](http://geode.apache.org/) and not to [BOSH](http://bosh.io/) itself. It basically verifies that the Server is part of the distributed system by using the [GFSH tool](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html) to check the list of members from the existing locators, and that the [Geode REST API](http://geode.apache.org/docs/guide/11/rest_apps/book_intro.html) running embedded into the Server is also correctly deployed and handling requests.

```bash
#!/bin/bash
set -e

export JAVA_HOME=/var/vcap/packages/java
export GEODE_HOME=/var/vcap/packages/geode
export PATH=$PATH:$JAVA_HOME/bin:$GEODE_HOME/bin

SERVER_NAME=server_<%= "#{spec.id}" %>
SERVER_BIND_ADDRESS=<%= "#{spec.address}" %>
REST_API_CONNECTION_STRING=http://$SERVER_BIND_ADDRESS:<%= p('http.port') %>/geode/swagger-ui.html
LOCATORS=<%= link('locator').instances.map { |l| "#{l.address}[#{link('locator').p('locator.port')}]"}.join("\\ ") %>

echo [`date '+%F %T'`]: "Verifying server $SERVER_NAME..."

for locator in $LOCATORS; do
  gfsh -e "connect --locator=$locator" -e "list members" | grep -q $SERVER_NAME
  if [ $? != 0 ]; then
      echo [`date '+%F %T'`]: "Verifying server $SERVER_NAME... Failure!. Locator $locator doesn't see the server as part of the cluster."
      exit 1
  fi
done

REST_API_STATUS=$(curl -s --head -w %{http_code} $REST_API_CONNECTION_STRING -o /dev/null)
if [[ $REST_API_STATUS != 200 ]]; then
  echo [`date '+%F %T'`]: "Verifying locator $LOCATOR_CONNECTION_STRING... Failure! (REST API)."
  exit 1
fi

echo [`date '+%F %T'`]: "Verifying server $SERVER_NAME... Done!"
exit 0
```

*jobs/server/templates/server.properties.erb*

This file is used to further configure the [Apache Geode](http://geode.apache.org/) Server, it's actually empty in our release, but left within the deployment anyway as an easy way to configure the internals of the Server without requiring a modification to the `ctl` script or the `spec` file.

```bash
### Geode Properties using default values ###
```

#### Smoke Tests

This is an special type of job, a [BOSH errand](https://bosh.io/docs/terminology.html#errand), a short-lived job that an operator can run multiple times after the deploy finishes. It should be executed manually, and we'll be using this functionality to make sure our locators and servers, along with the applications embedded within them, are up and running.

*jobs/smoke-tests/spec*

The errand will use the [GFSH Tool](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html), so it has the same dependencies as the server and locator jobs. We'll use [BOSH links](http://bosh.io/docs/links.html) to get some connection information from the servers and locators as well.

```
---
name: smoke-tests

templates:
  errands.erb: bin/run

packages:
- java
- geode

consumes:
- name: locator
  type: locator
- name: server
  type: server

properties: {}
```

*jobs/smoke-tests/monit*

This job and won't be monitored by [Monit](https://mmonit.com/), so we can leave the file empty.

```
```

*jobs/smoke-tests/templates/errands.erb*

This template is used to run the errand on the VM. We use [Geode GFSH](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html) to connect to the locators and verify the list of connected members, along with `curl` tool to check that the embedded Web Applications are running.

```bash
#!/bin/bash
set -e

export JAVA_HOME=/var/vcap/packages/java
export GEODE_HOME=/var/vcap/packages/geode
export PATH=$PATH:$JAVA_HOME/bin:$GEODE_HOME/bin

verify_membership() {
  LOCATORS=<%= link('locator').instances.map { |l| "#{l.address}[#{link('locator').p('locator.port')}]"}.join("\\ ") %>
  SERVERS_AMOUNT="<%= link('server').instances.size %>"
  LOCATORS_AMOUNT="<%= link('locator').instances.size %>"

  echo [`date '+%F %T'`]: "Verifying cluster membership..."

  for locator in $LOCATORS; do
    cluster_servers=$(gfsh -e "connect --locator=$locator" -e "list members" | grep "^server_"  | wc -l)
    cluster_locators=$(gfsh -e "connect --locator=$locator" -e "list members" | grep "^locator_"  | wc -l)

    if [[ "$cluster_servers" -ne "$SERVERS_AMOUNT" ]]; then
        echo [`date '+%F %T'`]: "Verifying cluster membership... Failure!. Locator $locator doesn't report the expected amount of servers."
        exit 1
    fi

    if [[ "$cluster_locators" -ne "$LOCATORS_AMOUNT" ]]; then
        echo [`date '+%F %T'`]: "Verifying cluster membership... Failure!. Locator $locator doesn't report the expected amount of locators."
        exit 1
    fi
  done

  echo [`date '+%F %T'`]: "Verifying cluster membership... Done!."
}

verify_applications() {
  PULSE_ENDPOINTS=<%= link('locator').instances.map { |l| "#{l.address}:#{link('locator').p('http.port')}"}.join("\\ ") %>
  REST_API_ENDPOINTS=<%= link('server').instances.map { |s| "#{s.address}:#{link('server').p('http.port')}"}.join("\\ ") %>

  echo [`date '+%F %T'`]: "Verifying cluster applications..."

  for pulseEndpoint in $PULSE_ENDPOINTS; do
    PULSE_STATUS=$(curl -s --head -w %{http_code} http://$pulseEndpoint/pulse/login.html -o /dev/null)

    if [[ $PULSE_STATUS != 200 ]]; then
      echo [`date '+%F %T'`]: "Verifying cluster applications... Failure!. Pulse can't be located on $pulseEndpoint."
      exit 1
    fi
  done

  for restEndpoint in $REST_API_ENDPOINTS; do
    REST_API_STATUS=$(curl -s --head -w %{http_code} http://$restEndpoint/geode/swagger-ui.html -o /dev/null)

    if [[ $REST_API_STATUS != 200 ]]; then
      echo [`date '+%F %T'`]: "Verifying cluster applications... Failure!. The REST API can't be located on $restEndpoint."
      exit 1
    fi
  done

  echo [`date '+%F %T'`]: "Verifying cluster applications... Done!."
}

verify_configuration() {
  echo "Nothing to do here yet..."
}

verify_membership
verify_applications
verify_configuration
```

---

### <a id="release"></a>Creating & Uploading the Release

At this point all artifacts needed to create the dev release are in place, the folder structure of the release directory should be as follows:

```bash
.
├── blobs
│   ├── geode
│   │   └── apache-geode-src-1.1.1.tar.gz
│   ├── gradle
│   │   └── gradle-3.5-bin.zip
│   └── java
│       └── jdk-8u131-linux-x64.tar.gz
├── config
│   ├── blobs.yml
│   └── final.yml
├── jobs
│   ├── locator
│   │   ├── monit
│   │   ├── spec
│   │   └── templates
│   │       ├── ctl.erb
│   │       ├── locator.properties.erb
│   │       └── post-start.erb
│   ├── server
│   │   ├── monit
│   │   ├── spec
│   │   └── templates
│   │       ├── cache.xml.erb
│   │       ├── ctl.erb
│   │       ├── post-start.erb
│   │       └── server.properties.erb
│   └── smoke-tests
│       ├── monit
│       ├── spec
│       └── templates
│           └── errands.erb
├── packages
│   ├── geode
│   │   ├── packaging
│   │   └── spec
│   ├── gradle
│   │   ├── packaging
│   │   └── spec
│   └── java
│       ├── packaging
│       └── spec
└── src
```

As we would have expected, [BOSH](http://bosh.io/) comes with handy commands to create and upload the release: [create-release](https://bosh.io/docs/cli-v2#create-release) and [upload-release](https://bosh.io/docs/cli-v2#upload-release). There's no need to be connected to the BOSH Director when creating the release, but it's mandatory when uploading it:

```bash
# Make the Credentials available for the BOSH CLI
$ export WORKSPACE_DIRECTORY=/workspace
$ export BOSH_CLIENT=admin
$ export BOSH_CLIENT_SECRET=`bosh int $WORKSPACE_DIRECTORY/config/creds.yml --path /admin_password`

# Create the Release
$ bosh -n create-release --force

# Upload the Release
$ bosh -n -e bosh-lite upload-release
```

That's all, our release is uploaded to the BOSH Director and, from now on, we can reference it from a deployment manifest to start deploying stuff!.

---

### <a id="manifest"></a>Creating the Deployment Manifest

The deployment manifest is a [YAML](http://yaml.org/) file that defines the components and properties of the [BOSH](http://bosh.io/) deployment. When an operator initiates a new deployment using the CLI, the Director receives a manifest and creates or updates a deployment with matching name.

For the sake of simplicity we'll keep this file as small as possible (it's under the "manifests" folder for simplicity, but it **should not even be part of this repository because it actually doesn't belog to the release**). Keep in mind that the deployment manifest is a world on its own and that it has to be aligned with the [cloud-config](https://bosh.io/docs/cloud-config.html) used by the Director, the IaaS being used, the available stemcells, networks and releases, etc.

The sample manifest uploaded to this repository defines a cluster composed by 2 locators and 4 servers, along with the smoke-test job as an errand, but you can change almost anything just by modifying this file (number of instances, IPs, types of VM, size of disks, VM memory, jobs properties, etc.). The official reference can be found at [Manifest v2 Schema](https://bosh.io/docs/manifest-v2.html).

```
---
name: geode-cluster

releases:
- name: geode-bosh
  version: latest

stemcells:
- alias: default
  os: ubuntu-trusty
  version: "latest"

update:
  canaries: 1
  max_in_flight: 2
  canary_watch_time: 60000
  update_watch_time: 60000

instance_groups:
- name: locator
  instances: 2
  azs: [z1, z2, z3]
  vm_type: default
  stemcell: default
  networks:
  - name: default
  jobs:
  - name: locator
    release: geode-bosh
    provides:
      locator: { as: locator }
- name: server
  instances: 4
  azs: [z1, z2, z3]
  vm_type: default
  stemcell: default
  networks:
  - name: default
  jobs:
  - name: server
    release: geode-bosh
    provides:
      server: { as: server }
    consumes:
      locator: { from: locator }
- name: smoke-tests
  instances: 1
  azs: [z1,z2,z3]
  lifecycle: errand
  vm_type: default
  stemcell: default
  networks:
  - name: default
  jobs:
  - name: smoke-tests
    release: geode-bosh
    consumes:
      server: { from: server }
      locator: { from: locator }
```

---

### <a id="deploy"></a>Deploying and Executing the Smoke Tests

We have the deployment manifest finished, the specified jobs have been uploaded as part of the release, and the rest of the referenced components (networks, stemcells, vm types, availability zones, etc.) have been previously uploaded within the cloud-config in the first step, so now we're officialy ready to create the deployment through the BOSH CLI [deploy](https://bosh.io/docs/cli-v2#deploy) command. We also want to run the smoke-tests after the deploy finishes, and that can be achieved by running the BOSH CLI [run-errand](https://bosh.io/docs/cli-v2#run-errand) command.

Putting it all together:

```bash
$ bosh -n -e bosh-lite -d geode-cluster deploy manifests/geode-deployment.yml
$ bosh -e bosh-lite -d geode-cluster run-errand smoke-tests
```

At this point our [Geode](http://geode.apache.org/) cluster, deployed through [BOSH](http://bosh.io/), is finally up and running!!.

We can get details about the instances and VMs running in our deployment (along with their IPs, CPU usage, disk usage, etc.) through the following BOSH CLI commands:

```bash
$ bosh -e bosh-lite -d geode-cluster vms --vitals
$ bosh -e bosh-lite -d geode-cluster instances --details
```

We can use a web browser to access the [Pulse Web Application](http://geode.apache.org/docs/guide/11/tools_modules/pulse/chapter_overview.html) deployed within the locators at http://locatorIp:7070/pulse. We can also use a web browser to access the [Swagger UI](http://swagger.io/) deployed on the servers at http://serverIp:7070/geode/swagger-ui.html, and execute some REST operations throught the [Geode REST API](http://geode.apache.org/docs/guide/11/rest_apps/book_intro.html). We could even use the [Apache Geode GFSH Tool](http://geode.apache.org/docs/guide/11/tools_modules/gfsh/chapter_overview.html) to connect to the cluster from the command line and start managing it. In summary, we can do anything we'd normally do with a regular and manually installed [Geode](http://geode.apache.org/) Cluster, and that's just because this is a *regular [Geode](http://geode.apache.org/) Cluster*, the only difference is that now we have an **easy way to version, package and deploy the cluster in a reproducible manner**!!.

Below is, yet another,  bash script I've used a lot while creating and testing the release:

```bash
#!/bin/bash
WORKSPACE_DIRECTORY=/workspace
export BOSH_CLIENT=admin
export BOSH_CLIENT_SECRET=`bosh int $WORKSPACE_DIRECTORY/config/creds.yml --path /admin_password`

function create_release() {
  set -x
  bosh -n create-release --force
  set +x
}

function upload_release() {
  set -x
  bosh -n -e bosh-lite upload-release
  set +x
}

function create_deployment() {
  set -x
  bosh -n -e bosh-lite -d geode-cluster deploy manifests/geode-deployment.yml
  set +x
}

function run_errands() {
  set -x
  bosh -e bosh-lite -d geode-cluster run-errand smoke-tests
  set +x
}

function delete_deployment() {
  set -x
  rm -Rf .dev_builds
  rm -Rf dev_releases
  bosh -n -e bosh-lite -d geode-cluster delete-deployment --force
  bosh -n -e bosh-lite delete-release geode-bosh
  set +x
}

selection=
until [ "$selection" = "6" ]; do
  echo "##############################################################################################################################################################################"
  echo "Select an option:"
  echo "  1 - Create Release."
  echo "  2 - Upload Release."
  echo "  3 - Create Deployment."
  echo "  4 - Execute Deployment Errands."
  echo "  5 - Clean Deployment."
  echo "  6 - Exit."
  echo "##############################################################################################################################################################################"
  echo -n "Enter a choice: "
  read selection
  echo ""
  case $selection in
    1)
      create_release
      ;;
    2)
      upload_release
      ;;
    3)
      create_deployment
      ;;
    4)
      run_errands
      ;;
    5)
      delete_deployment
      ;;
    6)
      exit
      ;;
        *) echo 'Invalid option, please select an option between 1 and 6';;
    esac
done

```
