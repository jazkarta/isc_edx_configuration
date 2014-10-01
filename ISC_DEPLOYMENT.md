# Provisioning and Deploying the ISC edx stack.

This page documents the procedure for provisioning, deploying and maintaining a
single-instance edx sandbox to AWS with all ISC customizations.

## Preparations

### Install Ansible and dependencies

This project uses
[ansible](http://docs.ansible.com/intro_installation.html#getting-ansible) and
the AWS management modules defined in that package.  Those modules depend on
[boto](http://boto.readthedocs.org/en/latest/) to connect to and manage EC2
resources. You will need to install both packages and their dependencies:

```
$ sudo pip install paramiko PyYAML jinja2 httplib2 ansible boto
```

Although it may be possible to work with both packages installed in a
virtualenv, this is **not reliable** and is discouraged.  It is best to install
into your global python (as unpleasant as that may be).

### Set Up boto access

In order for boto to work with AWS resources, it must have an access key id and
a secret access key.  AWS strongly urges that you set up an
[IAM user](http://docs.aws.amazon.com/IAM/latest/UserGuide/Using_SettingUpUser.html)
with all required privileges to do this.  Once the user is created, you'll need
to [create access credentials](http://docs.aws.amazon.com/IAM/latest/UserGuide/ManagingCredentials.html).
Download the credentials and then put them where `boto` will be able to find them.

Create a file in your home directory called `.boto` and enter the following
information, inserting the values for your IAM user:

```ini
[Credentials]
aws_access_key_id = <access_key_id>
aws_secret_access_key = <secret_access_key>
```

If you have multiple AWS accounts, you can create
[profiles](http://boto.readthedocs.org/en/latest/boto_config_tut.html#credentials)
to manage credentials for each separately.  For example, in `~/.boto`:

```ini
[profile isc]
aws_access_key_id = <access_key_for_isc_iam_user>
aws_secret_access_key = <secret_key_for_isc_iam_user>

[profile other_acct]
...

[Credentials]
aws_access_key_id = <default_access_key_id>
aws_secret_access_key = <default_secret_key>
```

If you do this, you may pass the `profile` extra variable (`-e`) when calling
`ansible-playbook` as described below.  If the profile you name is not present,
boto will fall back to the default credentials supplied in the `[Credentials]`
section.

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

Change directories into the configuration repository, check out the isc_release
branch and then move into the playbooks directory:

```
$ cd isc_edx_configuration
$ git fetch
$ git checkout isc_release
$ cd /playbooks
```

## Provisioning

Provisioning a new server need only happen once. Re-running the play will
destroy the existing instance and replace it:

```
$ ansible-playbook -i local isc/aws_monolith.yml
```

This will use the `local` inventory file, which ensures that the only host you
adress to begin with is localhost. The inventory will also ensure that you
connect to it directly, rather than via ssh.

If you wish to set up a second instance without destroying the current
instance, you may do so by providing an alternative value for
`instance_name_tag`:

```
$ ansible-playbook -i local isc/aws_monolith.yml -e "instance_name_tag=some_other_name"
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
region | **yes** | us-west-2 | determines which aws region should be used
zone | **yes** | us-west-2a | determines the availability zone used within the designated region
instance_type | **yes** | m3.medium | a minimum of an m3.medium is recommended by EdX
security_group | **yes** | default | must be a security group available in the designated region
keypair | **yes** | pk-isclc | must be a keypair available in the designated region
ami | **yes** | ami-cfa1e6ff | must be an AMI for an Ubuntu precise 64-bit OS available in the designated region
vpc_subnet_id | **yes** | subnet-ed03fa9a | must be a VPC subnet associated with the designated availability zone
root_ebs_size | **yes** | 50 | A minimum of 20GB is recommended by EdX, 50GB is recommended
terminate_instance | no | true | if false, then no instances will be terminated
assign_public_ip | no | true | if false, no public IP address will be assigned
elb | no | | designate the ELB in which instances to be terminated are located. The instances will be removed from the ELB
instance_profile_name | no | | the IAM user profile to use.  This should be automatically detected by the security credentials provided, but is available in case
aws_access_key | no | | a manual override for the access key id credential to use with `boto`
aws_secret_key | no | | a manual override for the secret key credential to use with `boto`

### A few words on configuration options

Extra options may be passed to ansible during the run of a playbook using the
`-e` flag as shown above. The format of the value passed to this flag is always
a single string containing one or more space-separated `key=value` pairs:

```
$ ansible-playbook playbook.yml -e "key1=val1 key2=val2 ... keyN=valN"
```

The `security_group` selected must *at least* allow access to the server on
ports 22 (ssh), 80 (http), and 443 (https). This will allow access to the EdX
LMS and to managing the server via ssh login. To access the EdX studio
functions, ports 18010 (http) and 48010 (https) need to be opened.

The `keypair` selected must be available to your local ssh agent in order for
tasks on the provisioned server to be completed. This can be accomplished in
one of two ways.  You can pass the path to the private key on your local
machine using the `--private-key` option to ansible-playbook. Alternatively,
you can add the key to your ssh agent as follows:

```
$ ssh-add /path/to/keypair.pem
```

This will make the key available to your ssh agent so that when it is
evaluating possible keys for connection, it can try this one as well. Remember
that for ssh private keys to be read, they must be readable only by the user
running ssh (you):

```
$ ls -l /path/to/keypair.pem
-r--------@ 1 user  group    1692 Jan 12  2014 keypair.pem
```
If this is not the case, you must fix it with the `chmod` command:

```
$ chmod 400 /path/to/keypair.pem
```

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
5. `NGINX_ENABLE_SSL=True`: enable ssl for the nginx http server
6. `COMMON_ENABLE_BASIC_AUTH=True`: enable basic auth to protect all sites.
7. `COMMON_HTPASSWD_USER=name`: provide the username for basic auth (this defaults to `isc`)
8. `COMMON_HTPASSWD_PASS=pwd`: provide the password for basic auth (this defaults to `edx`)


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

*documentation to come*
