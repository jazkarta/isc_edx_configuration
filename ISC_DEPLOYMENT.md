# Provisioning and Deploying the ISC edx stack.

This page documents the procedure for provisioning, deploying and maintaining a
single-instance edx sandbox to AWS with all ISC customizations.

## Preparations

### Install Ansible and dependencies

You will need to install [ansible](http://docs.ansible.com/intro_installation.html#getting-ansible)
and [boto](http://boto.readthedocs.org/en/latest/) globally.  Ansible does not
work well with virtualenv.

```
$ sudo pip install paramiko PyYAML jinja2 httplib2 ansible boto
```

### Set Up boto access

In order for boto to work with AWS resources, it must have an access key id and
a secret access key.  AWS strongly urges that you set up an [IAM user]() with
all required privileges to do this.  Once the user is created, you'll need to
put the two values into a place where boto can read them.

Create a file in your home directory called `.boto` and enter the following
information, inserting the values for your IAM user:

```ini
[Credentials]
aws_access_key_id = <access_key_id>
aws_secret_access_key = <secret_access_key>
```
**WARNING!!**

On occasion, ansible will fail to find this file.  If you suddenly find
yourself unable to complete the provisioning play below, you may need to set
the equivalent environment variables in your shell: `AWS_ACCESS_KEY_ID` and
`AWS_SECRET_ACCESS_KEY`.  Ansible documentation insists that these variable
names are flexible and offers several alternatives, however I have not found
other names to work correctly.

### Set Up working environment

You will also need to create a location in which to do your work.  You'll need
to have both the configuration repository and the repository that contains
customized resources used by the ISC edx stack:

```
$ mkdir isc
$ git clone git@github.com:jazkarta/isc_edx_configuration.git
$ git clone git@github.com:jazkarta/ISC.git
```

In the end, you should have a directory that looks like this:

```
isc
├── ISC
└── isc_edx_configuration
```

Finally, change directories into the configuration repository playbooks
directory:

```
$ cd isc_edx_configuration/playbooks
```

## Provisioning

Provisioning a new server need only happen once. Re-running the play will
destroy the existing instance and replace it:

```
$ ansible-playbook -i "localhost," isc/aws_monolith.yml
```

If you wish to set up a second instance without destroying the current
instance, you may do so by providing an alternative value for
`instance_name_tag`:

```
$ ansible-playbook isc/aws_monolith.yml -e "instance_name_tag: some_other_name"
```

The default value for `instance_name_tag` is "isc_staging"

The ansible playbook will execute, printing information on its progress to the
terminal. When it finishes you should see somthing a bit like this:

```
PLAY RECAP ********************************************************************
54.69.123.14               : ok=10   changed=6    unreachable=0    failed=0
localhost                  : ok=4    changed=1    unreachable=0    failed=0
```

The IP address above will represent the new AWS EC2 instance you just
provisioned.

### Other available configuration

In addition to the `instance_name_tag` variable, there are several other
configuration settings which may be of use in provisioning.  These may be added
to the `-e` argument when calling the `aws_monolith.yml` playbook:

Variable | Required | Default | Purpose
:--- | :--- | :--- | :---
region | yes | us-west-2 | determines which aws region should be used
zone | no | NONE | determines which aws availability zone should be used
elb | no | NONE | provide an elb id if this instance needs to be removed from an ELB
keypair | yes | pk-cpe | provide the name of a keypair tied to the region above
security_group | yes | vpc-ssh-access | provide the name if a security group tied to the region above
instance_type | yes | m3.medium | a minimum of an m3.medium is recommended by EdX
ami | yes | ami-cfa1e6ff | pick an ami available in the region above.  It must be Ubuntu precise 64-bit
vpc_subnet_id | yes | subnet-29b9a46f | must be the id of a VPC subnet within the network of the AWS account used
root_ebs_size | yes | 50 | A minimum of 20GB is recommended by Edx, 50GB is recommended
instance_profile_name | no | NONE | Name of the IAM instance profile to use
terminate_instance | no | true | if false, then no instances will be terminated


## Deployment

The deployment phase must be executed on the newly provisioned server. It is a
very long-running process and so I strongly recommend running it in a logged
screen session.

First, connect to the new server via ssh:

```
$ ssh -i </path/to/keypair.pem_file> ubuntu@<ip.addr.of.server>
```

You should see something like this:

```
*******************************************************************
*         _ __  __                                                *
*   _   _| |\ \/ /  This system is for the use of authorized      *
* / -_) _` | >  <   users only.  Usage of this system may be      *
* \___\__,_|/_/\_\  monitored and recorded by system personnel.   *
*                                                                 *
* Anyone using this system expressly consents to such monitoring  *
* and is advised that if such monitoring reveals possible         *
* evidence of criminal activity, system personnel may provide the *
* evidence from such monitoring to law enforcement officials.     *
*                                                                 *
*******************************************************************
```

Change directories to the directory where the configuration repository has been
set up:

```
$ cd /var/tmp/configuration/playbooks
```

### The initial build

From here you will want to execute the edx system sandbox installation:

```shell
$ sudo screen -Ldm ansible-playbook -c local ./isc_edx_sandbox.yml -i "localhost," -e "edx_ansible_source_repo=https://github.com/jazkarta/isc_edx_configuration.git configuration_version=isc_release edx_platform_repo=https://github.com/jazkarta/edx-platform.git edx_platform_version=isc NGINX_ENABLE_SSL=True COMMON_ENABLE_BASIC_AUTH=True COMMON_HTPASSWD_USER=name COMMON_HTPASSWD_PASS=pwd"
```

That's a lot to digest.  Let's take it apart:

1. `sudo`: The playbook must be run as sudo
2. `screen -Ldm` This launches ansible in a logged screen session and then
   immediately detatches. Output will be logged in ./screenlog.0
3. `ansible-playbook -c local ./isc_sandbox.yml`: run the isc_sandbox playbook
   using a local connection, not ssh.
4. `-i "localhost,"`: set the host list to localhost
5. `-e`: use the following extra variables. Vars passed with `-e` override any
   settings in the roles and tasks used by the playbook.

Let's take a quick look at the vars and values used in the above command:

1. `edx_ansible_source_repo=https://github.com/jazkarta/isc_edx_configuration.git`:
   sets the url for the repository from which the configuration will be checked
   out when building the edx distribution.
2. `configuration_version=isc_release`: determines the tag or branch of the above repository to check out
3. `edx_platform_repo=https://github.com/jazkarta/edx-platform.git`:
   sets the url for the repository for the edx source code.
4. `edx_platform_version=isc`: determines the tag or branch of the source repository to use
5. `COMMON_ENABLE_BASIC_AUTH=True`: enable basic auth to protect all sites.
6. `COMMON_HTPASSWD_USER=name`: provide the username for basic auth (this defaults to `isc`)
7. `COMMON_HTPASSWD_PASS=pwd`: provide the password for basic auth (this defaults to `edx`)


Once the above command is executed, you should have a log file in your current
working directory, `screenlog.0`.  You can tail this log to keep track of the
progress of the build.  Expect it to take a couple of hours.

```shell
$ tail -f screenlog.0
...
TASK: [edxapp | code sandbox | put sandbox apparmor profile in complain mode] ***
changed: [localhost]

TASK: [edxapp | code sandbox | Install base sandbox requirements and create sandbox virtualenv] ***
changed: [localhost]
...
```

At any time, you can quit following along by typing `^c` (control-c) in the
window where the log is displaying.

Because you are running the playbook in a screen, it is safe to log out and go
get dinner while the build is happening.

### Applying customizations

Once the build process is complete, you may be tempted to go get a beer and
celebrate, but there is still a bit of work left to do.  The build process does
not apply any of the customizations needed for ISC branding and microsites. In
order to apply these customizations there is one last step required.  You must
run the `/edx/bin/update` command for the edx-platform:

```
$ sudo /edx/bin/update edx-platform isc
```

This will run the `deploy` steps of the above process once again, and will
incorporate changes made via the `server-vars.yml` file in the ISC repository.
This command should not take more than 10-20 minutes to complete.


## Maintenance


