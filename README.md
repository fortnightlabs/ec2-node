Setup Node on EC2
-----------------

These are the steps needed to manually setup node.js on an EC2 instance.

**After you do the one-time setup** on your computer (information below),
you can use the provided shell scripts to setup and bundle a new
instance.

* `ec2-ubuntu` - creates an ubuntu server on ec2

    > ec2-ubuntu my-project

* `ubuntu-node` - sets up the server

    > ssh my-project ec2-node/ubuntu-node my-project

* `ec2-snapshot` - snapshot the newly created server

    > scp -r ~/.ec2 my-project:
    > ssh my-project "bash ec2-node/ec2-snapshot my-project"

* `ec2-node` - for the truly lazy. Does the create, clone, setup and the
  bundle in one glorious command.

    > ec2-node my-project

Both `ubuntu-node` and `ec2-snapshot` scripts expect you to already have
cloned the ec2-node github repository to the server.

    > ssh my-project "git clone git@github.com/fortnightlabs/ec2-node"

set up your computer
====================

**on your local machine**

go to the [amazon web
interface](https://aws-portal.amazon.com/gp/aws/developer/account/index.html)

note your `AWS_USER_ID` in the upper right hand corner. create a new
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

edit ~/.ec2/aws-keys

    > vi ~/.ec2/aws-keys

and save the information

    export AWS_USER_ID=XXXX-XXXX-XXXX
    export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
    export AWS_SECRET_ACCESS_KEY=xXxXXXxxXXXxXxxxXxxXXXXxXxXXxXxXxxXXxXxX

create a new X.509 certificate, download the private key and certificate
files, and then save the files in `$HOME/.ec2/`

    > mv ~/Downloads/*.pem ~/.ec2/

setup permissions on `~/.ec2`

    chmod 0600 ~/.ec2/*
    chmod 0700 ~/.ec2

if needed, install the [tools for working with
ec2](http://developer.amazonwebservices.com/connect/entry.jspa?externalID=351)

    > brew install ec2-api-tools
    > brew install ec2-ami-tools

add the following lines to your `.bash_profile`

    export JAVA_HOME="/System/Library/Frameworks/JavaVM.framework/Home"
    export EC2_PRIVATE_KEY="$(/bin/ls $HOME/.ec2/pk-*.pem)"
    export EC2_CERT="$(/bin/ls $HOME/.ec2/cert-*.pem)"
    export EC2_AMITOOL_HOME="/usr/local/Cellar/ec2-ami-tools/1.3-45758/jars"
    export EC2_HOME="/usr/local/Cellar/ec2-api-tools/1.3-53907/jars"
    source $HOME/.ec2/aws-keys

open a new shell, or

    > source ~/.bash_profile

if needed, create a new keypair for sever auth

    > ec2-add-keypair ec2-node > ec2-node.pem
    > chmod 0600 ec2-node.pem

copy that key to `~/.ec2/` as well

    > cp ec2-node.pem ~/.ec2/

create the server
=================

start-up a canonical provided ubuntu image (list of latest images
available at: <http://uec-images.ubuntu.com/releases/lucid/release/>),
here's the choice I made

* **us-east-1** - cheaper
* **32-bit** - more compatible, small and micro require it
* **ebs** - simpler storage, micro requires it

**note:** we're setting up the instance on a small (rather than micro)
platform because we need `/mnt` (not available on micro) to do the
bundling.  Once we have our new instance handy, we will move it to micro.

    > ec2-ubuntu am1.small mi-1234de7b us-east-1

the script will create a new instance and wait until it's running by
watching `ec2-describe-instances` for the string `running`

then the script will edit your `~/.ssh/config` file to make it easy to ssh
into the new instance

**note:** the `HostName` and `HostKeyAlias` will change based on the
output provided by `ec2-describe-instances`.

    Host my-project
      HostName ec2-67-202-82-82.compute-1.amazonaws.com
      HostKeyAlias ec2-67-202-82-82.compute-1.amazonaws.com
      User ubuntu
      ForwardAgent yes
      StrictHostKeyChecking no
      IdentityFile ~/.ec2/my-project.pem

now you can ssh into your instance

    > ssh my-project

set up the server
=================

**on your local machine**

copy the ubuntu-node script to the server

    > scp ubuntu-node my-project:

and then run it

    > ssh my-project "./ubuntu-node my-project"

the script will install useful utilities, then create an `apps` user for
the code to be deployed under (the scripts assume a capistrano
deployment strategy)

next, the script will set `upstart` to manage the node server process
and setup the sudoer priviledges correctly so the app user can restart
it.

finally, the script will install `node.js` and `npm` in the the
`/u/apps/.nodelocal` directory

now, we use need to use capistrano to finish deploying the code

    > export NODE_PROCESS="my-project"
    > cap deploy:setup
    > cap deploy:update
    > cap deploy:start

you can now visit <http://ec2-67-202-82-82.compute-1.amazonaws.com> (the
domain will be different) to make sure it works

deploying the instance
======================

going forward, you can simply deploy the instance with capistrano

    > NODE_PROCESS="my-project" cap deploy

**note** you really should copy the provided Capfile into your project
directory and change `#{ENV['NODE_PROCESS']}` to your application name
("my-project" in these examples)

bundle the instance
===================

**on your local machine**

copy your keys to the newly created server

    > scp -r ~/.ec2 my-project:

then, copy the ec2-snapshot script to the server

    > scp ec2-snapshot my-project:

finally run the snapshot script

    > ssh my-project "bash ec2-node/ec2-snapshot my-project"

the snapshot script will run `ec2-describe-instances` to find out
information about the instance

then it will run `ec2-describe-volumes` to find the volume that the
instance is using

next it will create a snapshot of the volume with `ec2-create-snapshot`

it will wait until the snapshot is completely by periodically checking
with `ec2-describe-snapshots`

when it finishes, the script will register the snapshot as an AMI with
amazon

    > ec2-register                      \
      --name my-project                 \
      --snapshot snap-1234abcde         \
      --kernel aki-1234abc              \  # from ec2-describe-instances
      --architecture i386               \
      --root-device-name /dev/sda1

after the image is registered, you will see output like `ami-1234de7b`

create a new instance from the AMI
==================================

Just run the ami identified in the output from the above step with
`ec2-run-instances`.  It should be all set-up and ready to go.

    > ec2-run-instances ami-98c035f1 --instance-type t1.micro --key ec2-node

A list of `--instance-type` options is available at
<http://docs.amazonwebservices.com/AWSEC2/latest/DeveloperGuide/>
