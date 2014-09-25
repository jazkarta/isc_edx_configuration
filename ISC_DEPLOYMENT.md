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

```
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
$ ansible-playbook isc/aws_monolith.yml
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

```

## Deployment

The deployment phase must be exe

## Maintenance


