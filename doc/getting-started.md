---
layout: documentation
---

# Getting started

## Introduction
Welcome to the OpenDevStack. The OpenDevStack is a framework to help in setting up a project infrastructure and continuous delivery processes on OpenShift and Atlassian toolstack with one click. This guide shall help you to setup the OpenDevStack, so you can work with it and test it in a local environment setup. The steps for the setup can also be adapted for running the OpenDevstack with an existing OpenShift installation or to connect it with your Atlassian tools, if you use [Atlassian Crowd](https://www.atlassian.com/software/crowd "Atlassian Crowd") as SSO provider.

**Important: The credentials provided in the guide are only meant to be used within the local test installation. For use in production you will have to customize paths, URLs and credentials!**

## Requirements
The following requirements have to be met to setup a local environment
### Git
We use Git as code repository, so you have to be familiar to work with [Git](https://git-scm.com/ "Git").
### Vagrant
The OpenDevStack uses Vagrant to provide a sample infrastructure for the Atlassian tools, Rundeck and a so called Ansible controller, a VM, where you can run Ansible scripts against the Atlassian VM. It is recommended to use the latest Vagrant version available from the [HashiCorp Vagrant page](https://www.vagrantup.com "Vagrant").

### Virtualbox
Vagrant uses Virtualbox for running the provisioned VMs. Therefore you must have [Virtualbox](https://www.virtualbox.org/ "Virtualbox") installed.
### Atlassian tools licenses
To use the Atlassian tools you need a license to run them. For testing and evaluation Atlassian provides evalutation licenses, which you can get on the [My Atlassian license page](https://my.atlassian.com/products/index "My Atlassian").
Here you have to keep in mind, that you have to register, if you don't have an Atlassian account. The registration is for free.
You need licenses for the following products:

- Bitbucket
- Jira
- Confluence
- Crowd

The evaluation licenses are valid for 30 days from the date of purchase. If you need a server id, this will be provided by the tools within the installation, so you are able to inlcude the licenses within the
installation wizards of the tools, after the base setup provided by the Ansible scripts.

### Minishift

The provided provision application and Rundeck jobs work with links, which are designed to connect to a installed and configured [Minishift](https://docs.openshift.org/latest/minishift/index.html "Minishift") instance. Minishift is a tool provided by Redhat to run OpenShift locally by providing a single-node OpenShift cluster inside a VM.
Information, how to setup Minishift can be found at the [Minishift Getting Started guide](https://docs.openshift.org/latest/minishift/getting-started/index.html "Getting Started with Minishift").
Currently the OpenDevStack works with Openshift 3.9.0.

### Cygwin / Linux

You must have the possibility to run bash scripts to import the provided OpenShift templates. On Linux systems you can use these scripts out-of-the box, on Windows systems you will have to install either a bash port for Windows like [Cygwin](https://www.cygwin.com/ "Cygwin").
For Windows, our recommendation is to use Cygwin for starting a minishift cluster and further configuration. Make sure to select the curl package under the "net" category when installing cygwin.

`minishift` will use the [.kube/config](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#explore-the-home-kube-directory) mechanism to configure access to the kubernetes cluster. Minishift will place the config in the Windows home directory. To make this work under `cygwin`, we will point the cygwin home directory to the Windows Home directory.
This can easily be achieved by changing the `db_home` entry in `/etc/nsswitch.conf` to

    dbhome: windows

or

    dbhome: /%H

This is described in the [cygwin user guide](https://cygwin.com/cygwin-ug-net/ntsec.html#ntsec-mapping-nsswitch-syntax).

### Ansible

The OpenDevStack uses [Ansible](https://www.ansible.com/ "Ansible") to install and configure the necessary software for the enabling stack, so it's recommended to get familiar with its core concepts and usage. Also credentials are stored within an Ansible vault, so even if you commit them in a public repository they are not available unless you know the vault password.

## Setup your local environment

From now on we assume, you work from a Bash (Cygwin / Linux).

### Tailor

We use [tailor](https://github.com/opendevstack/tailor) to handle our versioned OpenShift templates and keep our cluster in sync. Please see its [installation instructions](https://github.com/opendevstack/tailor#installation) for your platform. The following lists the version requirements:

| OpenDevStack version | Tailor version |
|---|---|
| 0.1.x | = 0.8 |
| 1.0.x | >= 0.9.1 |

### Prepare infrastructure

First you have to clone the [ods-core](http://www.github.com/opendevstack/ods-core) repository.

Navigate to the folder **ods-core > infrastructure-setup**.
There you will find a Vagrant file. You can start the infrastructure provisioning and setup by using
```shell
vagrant up
```
After Vagrant has provisioned the VMs you are able to connect to them. There are two VMs, `atlcon` and `atlassian1`.
First connect to the Ansible controller `atlcon` from the directory you ran the Vagrantfile from via
```shell
vagrant ssh atlcon
```
After the connect change the directory to `/vagrant/ansible`.
Here you have to execute the following command:
```shell
ansible-playbook -i inventories/dev dev.yml --ask-vault-pass
```
This playbook prepares the ansible controller and basic installations on the `atlassian1` VM like a local database and the necessary schemas with their respective user.
The password for the vault located under `ansible/inventories/dev/group_vars/all/vault.yml` is `opendevstack`.
Depending on your network or proxy configuration it might happen that some online resources are not reachable. Please try to execute the playbook again in such a case.

**All ansible playbook commands in this guide have to be executed from the Ansible controller like described before!**

### Install Atlassian Tools and Rundeck

The following steps explain the Atlassian tools and the Rundeck installation.

#### Crowd Setup

##### Setup Application
Downloading and Configuring as service
```shell
ansible-playbook -v -i inventories/dev playbooks/crowd.yml --ask-vault
```

<!-- TODO
This can be fixed in the vagrant file. But should we really notice this here, because the recommendation is to use cygwin?

If you are using windows you might get a warning that the directory containing the ansible files is world writable.
Therefore the ansible.cfg config-file will not be used, which will cause ansible to use the wrong directory to search for roles.
To fix this you can modify the ansible.cfg file in /etc/ansible and change the role_path to /vagrant/ansible/roles.
-->

##### Run Configuration Wizard

Access http://192.168.56.31:8095/crowd/console

Be patient. First time accessing this page will take some time.

###### Step 1: License key
Here you can see the server id you need for the license you can get from the [My Atlassian page](https://my.atlassian.com/products/index "My Atlassian"). Use the link to get an evaluation license (Crowd Server) or enter a valid license key into the textbox.
###### Step 2: Crowd installation
Here choose the **New installation** option.
###### Step 3: Database Configuration
The next step is the database configuration.
Choose the **JDBC Connection** option and configure the database with the following settings

{: .table-bordered }
{: .table-sm }
| Option            | Value                                                                                                                |
| ----------------- | -------------------------------------------------------------------------------------------------------------------- |
| Database          | PostgreSQL                                                                                                           |
| Driver class name | org.postgresql.Driver                                                                                                |
| JDBC URL          | jdbc:postgresql://localhost:5432/atlassian?currentSchema=crowd&amp;reWriteBatchedInserts=true&amp;prepareThreshold=0 |
| Username          | crowd                                                                                                                |
| Password          | crowd                                                                                                                |
| Hibernate dialect | org.hibernate.dialect.PostgreSQLDialect                                                                              |



###### Step 4: Options
Choose a deployment title, e.g. *OpenDevStack* and set the **Base URL** to `http://192.168.56.31:8095/crowd`

###### Step 5: Mail configuration
For the local test environment a mail server is not necessary, so you can skip this step by choosing **Later**

###### Step 6: Internal directory
Enter the name for the internal crowd directory, e.g. *OpenDevStack*

###### Step 7: Default administrator
Enter the data for the default administrator, so you are able to login to crowd.

###### Step 8: Integrated applications
Enable both integrated applications.

###### Step 9: Log in to Crowd console
Now you can verify the installation and log in with the credentials defined in the previous step.

##### Configure Crowd
You will have to configure crowd to enable the Atlassian tools and Rundeck to login with crowd credentials.

###### Add OpenDevStack groups
You will have to add the following groups to crowd's internal directory

{: .table-bordered }
{: .table-sm }
| Group                       | Description                                         |
| --------------------------- | --------------------------------------------------- |
| opendevstack-users          | Group for normal users without adminstration rights |
| opendevstack-administrators | Group for administration users                      |

###### Add Atlassian groups
You also have to add the groups from the atlassian tools, even if you don't use them.

{: .table-bordered }
{: .table-sm }
| Group                     | Description                    |
| ------------------------- | ------------------------------ |
| bitbucket-administrators  | Bitbucket administrator group  |
| bitbucket-users           | Bitbucket user group           |
| jira-administrators       | Jira administrator group       |
| jira-developers           | Jira developers group          |
| jira-users                | Jira user group                |
| confluence-administrators | Confluence administrator group |
| confluence-users          | Confluence user group          |

To do so, access the crowd console at http://192.168.56.31:8095/crowd/console/
Choose the **Groups** menu point and click **Add group**
Enter the group name like shown above and link it to the created internal directory.

###### Add groups to user
Now you have to add all groups to the administrator.
Go to the **Users** section in Crowd, choose your administration user and open the **Groups** tab.
Click **Add groups**, search for all by leaving the Search fields empty and add all groups.

###### Add applications to crowd
You will have to add the applications you want to access with your Crowd credentials in the Crowd console.
Access the Crowd console at http://192.168.56.31:8095/crowd/console/
Choose the **Applications** menu point and click **Add application**
In the following wizard enter the data for the application you want to add. See the data for the applications in the test environment in the table below.

{: .table-bordered }
{: .table-sm }
| Application type    | Name       | Password   | URL                               | IP address    | Directories                                 | Authorisation | Additional Remote Adresses |
| ------------------- | ---------- | ---------- | --------------------------------- | ------------- | ------------------------------------------- | ------------- | -------------------------- |
| Jira                | jira       | jira       | http://192.168.56.31:8080         | 192.168.56.31 | Internal directory with OpenDevStack groups | all users     | 0.0.0.0/0 |
| Confluence          | confluence | confluence | http://192.168.56.31:8090         | 192.168.56.31 | Internal directory with OpenDevStack groups | all users     | 0.0.0.0/0 |
| Bitbucket Server    | bitbucket  | bitbucket  | http://192.168.56.31:7990         | 192.168.56.31 | Internal directory with OpenDevStack groups | all users     | 0.0.0.0/0 |
| Generic application | rundeck    | secret     | http://192.168.56.31:4440/rundeck | 192.168.56.31 | Internal directory with OpenDevStack groups | all users     | 0.0.0.0/0 |
| Generic application | provision  | provision  | http://192.168.56.1:8088             | 192.168.56.1  | Internal directory with OpenDevStack groups | all users     | 0.0.0.0/0 |

#### Bitbucket Setup

##### Setup Application

```shell
ansible-playbook -v -i inventories/dev playbooks/bitbucket.yml --ask-vault
```

##### Run Configuration Wizard

Access http://192.168.56.31:7990

Be patient. First time accessing this page takes some time.

On the configuration page you have the possibility to define the application name, the base URL and to get an evaluation license or enter a valid license.
If you choose to get an evaluation license you can retrieve it from the my atlassian page. You will be redirected automatically.
After adding the license you have to create a local Bitbucket administrator account.
Don't integrate Bitbucket with Jira at this point, but proceed with going to Bitbucket.

##### Configure Crowd access
Go to the Bitbucket start page at http://192.168.56.31:7990/
Open the administration settings and navigate to the **User directories** menu.
Here you have to add a directory of type *Atlassian Crowd*.
Here you have to add the Crowd server URL `http://192.168.56.31:8095/crowd`
You also have to add the application name and the password you have defined for Bitbucket in crowd.
For the local test environment this is `bitbucket` `bitbucket`
Now activate **nested groups** and deactivate the **incremental synchronization**
The group membership should be proofed every time a user logs in.
Test the settings and save them.
Now change the order of the user directories. The Crowd directory has to be on first position.

##### Configure user groups
###### Add groups
After configuring the crowd directory change to **Groups**
Here you have to add the groups defined in crowd in the previous steps, if
they are not available yet.

{: .table-bordered }
{: .table-sm }
| Group                    | Description                   |
| ------------------------ | ----------------------------- |
| bitbucket-administrators | Bitbucket administrator group |
| bitbucket-users          | Bitbucket user group          |



###### Add permissions
The last step is to configure the permissions for the created groups.
Go to the **Global permissions** menu.
In the groups section add the `bitbucket-administrators` group with *System Admin* rights.
Add the `bitbucket-users` group with *Project Creator* rights.

#### Jira Setup
##### Setup Application
```shell
ansible-playbook -v -i inventories/dev playbooks/jira.yml --ask-vault
```

##### Run Configuration Wizard
Access http://192.168.56.31:8080

Be patient. First time accessing this page takes time.

###### Step 1: Setup application properties
Here you have to choose the application title and the base URL.
You can leave the data as is for the test environment.

###### Step 2: Specify your license key
Here you have to enter the license key for the Jira instance (Jira Software (Server)). With the provided link in the dialogue you are able to generate an evaluation license at Atlassian.

###### Step 3: Set up administrator account
Now you have to set up a Jira administrator account.

###### Step 4: Set up email notifications
Unless you have configured a mail server, leave this for later.

###### Step 5: Basic configuration
To finish this part of the Jira installation, you will have to provide some informations to your prefered language, your avatar and you will have to create an empty or a sample project.
After these basic configurations, you have access to the Jira board.

##### Configure Crowd access

###### Configure user directory
Open the **User management** in the Jira administration.
To enter the administration, you have to verify you have admin rights with the password for your admin user.
Click the **User Directories** entry at the left..
Now choose **Add Directory**.
Here you have to add a directory of type *Atlassian Crowd*.
Here you have to add the Crowd server URL `http://192.168.56.31:8095/crowd`
You also have to add the application name and the password you have defined for Jira in crowd.
For the local test environment this is `jira` `jira`
Now activate **nested groups** and deactivate the **incremental synchronization**
The group membership should be proofed every time a user logs in.
Test the settings and save them.
Now change the order of the user directories. The Crowd directory has to be on first position.
###### Configure SSO with crowd
To finish the SSO configuration, you will have to run the following playbook command:
```shell
ansible-playbook -v -i inventories/dev playbooks/jira_enable_sso.yml --ask-vault
```
This will configure the authenticator.

**After Jira has been restarted, you are not able to login with the local administrator anymore, but with your crowd credentials.**

#### Confluence Setup

##### Setup Application
```shell
ansible-playbook -v -i inventories/dev playbooks/confluence.yml --ask-vault
```
##### Run Configuration Wizard
Access http://192.168.56.31:8090

###### Step 1: Set up Confluence
Here you have to choose **Production Installation**, because we want to configure an external database.

###### Step 2: Get add-ons
Ensure the add-ons are unchecked and proceed.

###### Step 3: License key
Here you are able to get an evaluation license from atlassian or to enter a valid license key.

###### Step 4: Choose a Database Configuration
Here you have to choose **External Database** with the option *PostgrSQL*

###### Step 5: Configure Database
Click the **Direct JDBC** button and configure the database with the following values:

{: .table-bordered }
{: .table-sm }
| Option            | Value                                       |
| ----------------- | ------------------------------------------- |
| Driver Class Name | org.postgresql.Driver                       |
| Database URL      | jdbc:postgresql://localhost:5432/confluence |
| User Name         | confluence                                  |
| Password          | confluence                                  |

Be patient. This step takes some time until next page appears.

###### Step 6: Load Content
Here you have to choose **Empty Site** or **Example Site**

###### Step 7: Configure User Management
Choose **Manage users and groups within Confluence**. Crowd will be configured later.

###### Step 8: Configure System Administrator account
Here you have to configure a local administrator account. After this step, you are able to work with Confluence. Just press Start and create a space.

##### Configure Crowd access
###### Configure user directory
Open the **User management** in the Confluence administration.
To enter the administration, you have to verify you have admin rights with the password for your admin user.
Click the **User Directories** entry at the left in the **USERS & SECURITY** section.
Now choose **Add Directory**.
Here you have to add a directory of type *Atlassian Crowd*.
Here you have to add the Crowd server URL `http://192.168.56.31:8095/crowd`
You also have to add the application name and the password you have defined for Confluence in crowd.
For the local test environment this is `confluence` `confluence`
Now activate **nested groups** and deactivate the **incremental synchronization**
The group membership should be proofed every time a user logs in.
Test the settings and save them.
Now change the order of the user directories. The Crowd directory has to be on first position.

###### Configure SSO with crowd
To finish the SSO configuration, you will have to run the following playbook command:

```shell
ansible-playbook -v -i inventories/dev playbooks/confluence_enable_sso.yml --ask-vault
```
This will configure the authenticator.
**After Confluence has been restarted, you are not able to login with the local administrator anymore, but with your crowd credentials.**

#### Create opendevstack project in Bitbucket
We will mirror the opendevstack project into this Bitbucket instance.
Therefore, we need to create a new _project_.

* Go to the Projects page in Bitbucket
* Hit "Create" button
* enter Project Name: OpenDevStack and key: OPENDEVSTACK
* Hit `Create Project`
* In the settings section, allow the `bitbucket-users` group write access.

You will be directed to the projects dashboard.
Using the '+' sign  you need to create a couple of repositories:

* ods-core
* ods-configuration
* ods-configuration-sample
* ods-jenkins-shared-library
* ods-project-quickstarters
* ods-provisioning-app

On the Project Dashboard Navigate to the "Settings" menu and grant the group "opendevstack-users" admin access.

Navigate to the **ods-core/infrastructure-setup/scripts** directory and execute
`mirror-repos.sh`

Use your crowd login when asked for credentials.
Verify that you have mirrored the github repos and that they have been populated in your Bitbucket instance. The ods-configuration repositpory will remain empty.

Setup project branch permissions - `production` should be guarded against direct merges except through admins

#### Rundeck Setup
##### Setup Application
Rundeck needs an account to access Bitbucket later. We will create an ssh keypair for this and add this later to the Bitbucket `cd_user` account.

Open the shell and generate a ssh key. On cygwin enter the following command:
```shell
ssh-keygen -f /home/vagrant/cd_user -t rsa -C "CD User"
```
This saves the public and private key in a file `cd_user.pub` and `cd_user`.


Create a file called `/home/vagrant/rundeck_vars.yml` that customizes some of the rundeck configuration, e.g. the ssh key.

This is a yaml file, looking structurally like this Example

```yaml
rundeck_bitbucket_host_external: 192.168.56.31
rundeck_bitbucket_host_internal: localhost
rundeck_bitbucket_port: 7999
rundeck_cduser_name: cd_user
rundeck_cduser_private_key: |
  -----BEGIN RSA PRIVATE KEY-----
  MIIJKgIBAAKCAgEA9byVUZKe0dB0gkFL5g4Zcxb3AUNPvtD2tpkejyaLoF/XnQj+
  qn+UX9WZSn0YyTQH+cmNF1SFuMmq/eSZpdAL7JSRY2bAw9RLo3dPpabO2N3Teib1
  HSvCnPncNQZa/tPUaWSddX0BTWEpS1fAl4NFfUmN02k+cEHIErv2OcbhMnq675aO
  p4rU3NHN01kymhUCLz5cUCAj4CyEhxv3Fe7zSeKGuSceaD2Yq1vEnp8WmYnqdiFf
  ....
  0rMrGoSgTuttxQ+oU2a+2pRQD+vFXg6BpXMJNXeXyPuSIVfqfSFTqUdshZC8d76Q
  8IwfUR/GtEjTO4l9nDr0eqb4LixvpREVVvMOH+Ea/a8yATejH9xR7xNHAA0AQqZ+
  t1pNCqijBNTk2oUYNu9t9m16zF3Ly+ZIikBm0D67ke5yC5ziSPa1Xs6E70ens04H
  RwP9We5Y453L2st43FlQXVAyXd4OacJcUqvYqQpd7c7u1syhpRzG5ALYcfoNJA==
  -----END RSA PRIVATE KEY-----
```

You have to replace the private key with the key you created earlier and change
other variables according to your environment. Be careful about the 2 spaces at the beginning of every line of the private key.

Now execute the playbook:

```shell
ansible-playbook -v -i inventories/dev playbooks/rundeck.yml -e "@/home/vagrant/rundeck_vars.yml" --ask-vault
```

You can change `host` and `cduser` according to your environment.
<!-- TODO
This is superfluous if we mirror the repos first to our vagrant / local bitbucket server.
-->
After the playbook has been finished Rundeck is accessible via http://192.168.56.31:4440/rundeck


### Configure Minishift
#### Minishift startup
First you have to install Minishift. You have to use a minishift version >= 1.14.0, so openshift v3.9.0 (see below) is supported.

To do so, follow the installation instructions of the [Minishift Getting Started guide](https://docs.openshift.org/latest/minishift/getting-started/index.html "Getting Started with Minishift").

Before you start up Minishift with the `minishift start` command you will have to create or modify a `config.json` file.
This file is located in the `.minishift/config` folder in the user home directory.
On a Windows system, you will find this file under `C:\Users\<username>\.minishift\config\config.json` or under cygwin `~/.minishift/config/config.json`.
If the file doesn't exist, you will have to create it.
The file has to have the following content:

```javascript
{
    "cpus": 2,
    "memory": "8192",
    "openshift-version": "v3.9.0",
    "disk-size": "40GB",
    "vm-driver": "virtualbox"
}
```

It is important to use *v3.9.0* as minimum version to ensure, that the templates provided by the OpenDevStack work properly. If you are on windows you have to run the "minishift start" command as administrator.

After the start up you are able to open the webconsole with the `minishift console` command. This will open the webconsole in your standard browser.
Please access the webconsole with the credentials `developer` `developer`.
It is *important* not to use the `system` user, because Jenkins does not allow a user named `system`.
<!-- TODO we are not doing anything on the console with jenkins, why is this hint important -->

### Configure the path for the OC CLI
The OC CLI is automatically downloaded after "minishift start".
To add it to the path you can run
```shell
minishift oc-env
```
and execute the displayed command.

#### Login with the CLI
You have to login via the CLI with
```shell
oc login -u system:admin
```

#### Setup the base template project
After you have logged in, you are able to create a project, that will contain the base templates and the Nexus Repository Manager. Please enter the following command to add the base project:
```shell
oc new-project cd --description="Base project holding the templates and the Repositoy Manager" --display-name="OpenDevStack Templates"
```
This command will create the base project.

#### Adjust user rights for the developer user
To be able to see all created projects, you will have to adjust the user rights for the developer use. Do so by using the provided command
```shell
oc adm policy --as system:admin add-cluster-role-to-user cluster-admin developer
```

#### Create service account for deployment
Rundeck needs a technical account in Minishift to be able to create projects and provision resources. Therefore, we create a service account, which credentials are provided to Rundeck in a later step.
```shell
oc create sa deployment -n cd
oc adm policy --as system:admin add-cluster-role-to-user cluster-admin system:serviceaccount:cd:deployment
```
After you have created the service account we need the token for this account.
```shell
oc sa get-token deployment -n cd
```
Save the token text. It will be used in the Rundeck configuration later.

#### Install Minishift certificate on Atlassian server
You have to add the Minishift certificate to the `atlassian1` JVM, so Bitbucket is able to execute REST Calls against Minishift, triggered by Webhooks.
Go to the directory, where you have started Vagrant.
Here open a SSH connection to the `atlassian1` server
```shell
vagrant ssh atlassian1
```
On the server change to the root account
```shell
sudo -i
```
Here execute the following command to get the certificate from the Minishift server:
```shell
 openssl s_client -connect 192.168.99.100:8443 -showcerts < /dev/null 2>/dev/null| sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /tmp/minishift.crt
```
You should now have two PEM encoded certificate in /tmp/minishift.crt.
Remove the first one (this is the server certificate) and keep the CA Cert.

Check that you got the CA certificate:
```shell
openssl x509 -in /tmp/minishift.crt -text
```

You should see a section:
```
    X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
```

Now import the certificate in the default JVM keystore.
```shell
sudo /usr/java/latest/jre/bin/keytool -import -alias minishift -keystore /usr/java/latest/jre/lib/security/cacerts -file /tmp/minishift.crt
```

The default password is `changeit`.
Confirm with yes when ask to trust the certificates.
Restart the bitbucket service
```shell
sudo service atlbitbucket restart
```

We need this certificate for the Rundeck part later as well.
On the atlassian1 server clone the `ods-project-quickstarters` from your Bitbucket server.
```shell
sudo su - rundeck
git clone http://192.168.56.31:7990/scm/opendevstack/ods-project-quickstarters.git
git config --global user.email "cd_user@opendevstack.local"
git config --global user.name "cd_user"
cat /tmp/minishift.crt >> ods-project-quickstarters/ocp-templates/root.ca/ca-bundle.crt
cd ods-project-quickstarters
git commit -am "added local root ca"
git push origin master
```

Use your crowd login when asked for credentials.
we do this as the rundeck user, so we can accept the ssh host key.

### Prepare environment settings
Switch to your local machine and clone the repositories: `ods-configuration-sample` and `ods-configuration` from your bitbucket server.
Copy the entire directory structure from `ods-configuration-sample` into `ods-configuration`and remove the .sample postfixes.

```shell
git clone http://192.168.56.31:7990/scm/opendevstack/ods-configuration-sample.git
git clone http://192.168.56.31:7990/scm/opendevstack/ods-configuration.git
cp -r ./ods-configuration-sample/. ./ods-configuration
find ods-configuration -name '*.sample' -type f | while read NAME ; do mv "${NAME}" "${NAME%.sample}" ; done
```
(Assuming your host/ip for bitbucket is: 192.168.56.31:7990)

Now you will have to check the `.env` configuration files in `ods-configuration`. Change all values with the suffix `_base64` to a Base64 encoded value.  

### Setup and Configure Nexus3
Amend `ods-configuration/ods-core/nexus/ocp-config/route.env` and change the domain to match your openshift/minishift domain (for example `nexus-cd.192.168.99.100.nip.io`)

Go to `ods-core/nexus/ocp-config` - and type
```shell
tailor update
```
You should see a proposed list of new objects that are created - and confirm with `y`

```shell
Comparing templates in C:\code_bix\opendevstack_at_BIX\ods-core\nexus\ocp-config with OCP namespace cd.
Limiting resources to dc,is,pvc,route,svc with selector app=nexus3.
Found 0 resources in OCP cluster (current state) and 5 resources in processed templates (desired state).

[32m+ dc/nexus3 to be created
[0m--- Current State (OpenShift cluster)
+++ Desired State (Processed template)
@@ -1 +1,63 @@
+apiVersion: v1
+kind: DeploymentConfig
+metadata:
+  annotations:
+    original-values.tailor.io/spec.template.spec.containers.0.image: sonatype/nexus3:latest
+  creationTimestamp: null
+  labels:
+    app: nexus3
+  name: nexus3
+spec:
+  replicas: 1
+  selector:
+    app: nexus3
+    deploymentconfig: nexus3
+  strategy:
+    activeDeadlineSeconds: 21600
+    recreateParams:

.......

```

#### Configure Repository Manager
Access Nexus3 http://nexus-cd.192.168.99.100.nip.io/
Login with the default credentials for Nexus3 `admin` `admin123`

##### Configure repositories
Open the **Server administration and configuration** menu
by clicking the gear icon in the top bar.
Now create three Blob Stores.

| Type | Name             | Path                               |
| ---- | ---------------- | ---------------------------------- |
| File | candidates       | /nexus-data/blobs/candidates       |
| File | releases         | /nexus-data/blobs/releases         |
| File | atlassian_public | /nexus-data/blobs/atlassian_public |

After this step you will have to create the following repositories in the **Repositories** Subsection.

| Name             | Format | Type   | Online  | Version policy | Layout policy | Storage    | Strict Content Type Validation | Deployment policy | Remote Storage | belongs to group                                                    |
| ---------------- | ------ | ------ | ------- | -------------- | ------------- | ---------- | ------------------------------ | ----------------- | ------------------------------------------------------------------ | ------------ |
| candidates       | maven2 | hosted | checked | Release        | Strict        | candidates | checked                        | Disable-redeploy  | | none                                                                   |
| releases         | maven2 | hosted | checked | Release        | Strict        | releases   | checked                        | Disable-redeploy  | | none                                                                   |
| npmjs           | npm     | proxy  | checked |                |               | default    | checked                        |   |  https://registry.npmjs.org  | |
| atlassian_public | maven2 | proxy  | checked | Release        | Strict        | atlassian_public  | checked                 | Disable-redeploy  | https://maven.atlassian.com/content/repositories/atlassian-public/ |
| jcenter | maven2 | proxy  | checked | Release        | Strict        | default  | checked                 | Disable-redeploy  | https://jcenter.bintray.com | maven-public
| sbt-plugins | maven2 | proxy  | checked | Release        | permissive | default  | unchecked                 | Disable-redeploy  | http://dl.bintray.com/sbt/sbt-plugin-releases/ | ivy-releases
| sbt-releases | maven2 | proxy  | checked | Release        | permissive | default  | unchecked                 | Disable-redeploy  | https://repo.scala-sbt.org/scalasbt/sbt-plugin-releases | ivy-releases
| typesafe-ivy-releases | maven2 | proxy  | checked | Release        | permissive | default  | unchecked                 | Disable-redeploy  | https://dl.bintray.com/typesafe/ivy-releases | ivy-releases
| ivy-releases | maven2 | group  | checked | Release        | permissive | default  | unchecked                 | Disable-redeploy  | |


##### Configure user and roles
First disable the anonymous access in the **Security > Anonymous** section.
Under **Security > Roles** create a nexus-role *OpenDevStack-Developer*.

| Role ID                | Role name              | Role description                  |
| ---------------------- | ---------------------- | --------------------------------- |
| opendevstack-developer | OpenDevStack-Developer | Role for access from OpenDevStack |

This role has to have the following privileges:

| Privilege                                    |
| -------------------------------------------- |
| nx-repository-admin-maven2-candidates-browse |
| nx-repository-admin-maven2-candidates-edit   |
| nx-repository-admin-maven2-candidates-read   |
| nx-repository-view-maven2-\*-\*              |
| nx-repository-view-maven2-candidates-\*      |
| nx-repository-view-npm-\*-\* |

Now create a user under **Security > Users**.

| Name      | Password  |
| --------- | --------- |
| developer | developer |

You can choose any First name, Last name and Email.
Make this account active and assign role `OpenDevStack-Developer` to this account.

This account is later used for authentication against nexus to pull artifacts during build phase

### Import base templates
After you have configured Nexus3, import the base templates for OpenShift.
Clone the [ods-project-quickstarters](https://www.github.com/opendevstack/ods-project-quickstarters).
Navigate to the folder, where the cloned repository is located and navigate to the `ocp-templates/scripts` subfolder.
From with this folder, check if you are still logged in to the OpenShift CLI and login, if necessary.

Amend `ods-configuration/ods-project-quickstarters/ocp-templates/templates/templates.env` and run

```shell
./upload-templates.sh
```

This script creates the basic templates used by the OpenDevStack quickstarters in the `cd` project.
If you have to modify templates, there are also scripts to replace existing templates in OpenShift.

### Prepare CD project for Jenkins

Now create secrets inside the CD project.

```shell
oc process -n cd templates/secrets -p PROJECT=cd | oc create -n cd -f-
```

We will now build base images for jenkins and jenkins slave:

* Customize the configuration in the `ods-configuration` project at **ods-core > jenkins > ocp-config > bc.env**
* Execute `tailor update` inside ods-core/jenkins/ocp-config:

* Start jenkins slave base build: `oc start-build -n cd jenkins-slave-base`
* check that builds for `jenkins-master` and `jenkins-slave-base` are running and successful.
* You can optionally start the `jenkins-master` build using `oc start-build -n cd jenkins-master`

#### Prepare Jenkins slave docker images
To support different kinds of projects, we need different kinds of Jenkins slave images.
These slave images are located in the project [jenkins-slave-dockerimages](https://github.com/opendevstack/jenkins-slaves-dockerimages).

So as a first step clone this repository.
Make the required customizations in the `ods-configuration` under **jenkins-slaves-dockerimages > maven > ocp-config > bc.env**

and run `tailor update` inside `ods-project-quickstarters\jenkins-slaves\maven\ocp-config`:

and start the build: `oc start-build -n cd jenkins-slave-maven`.

Repeat for every project type you require.

### Configure CD user
The continuous delivery process requires a dedicated system user in crowd for accessing bitbucket.
Access the [crowd console](http://192.168.56.31:8095/crowd/console/) and choose **Add user** in the **Users** menu.
Enter valid credentials. The only restriction here is, that the user has the username `cd_user` and that the user belongs to the internal crowd directory.
After creating the user you have to add the following groups:

| Group              |
| ------------------ |
| opendevstack-users |
| bitbucket-users    |

After you have created the user in crowd, you must add the public cd_user SSH key to the Bitbucket account.

Open [Bitbucket](http://192.168.56.31:7990/), login with your crowd administration user and go to the administration.
Here open the User section. If you can't see the CD user, you have to synchronize the Crowd directory in the **User directories** section.
Click on the CD user. In the user details you have the possiblity to add a SSH key. Click on the tab and enter the _public key_ from the generated key pair.

### Setup and configure SonarQube

Amend `ods-configuration/ods-core/sonarqube/ocp-config/sonarqube.env`
and type

```shell
tailor update

```
confirm with `y` and installation should start.

After the installation has taken place, you will have to build SonarQube: `oc start-build -n cd sonarqube`

Go to http://sonarqube-cd.192.168.99.100.nip.io/ and log in with your crowd user. Click on your profile on the top right, my account / security - and create a new token (and save it in your notes). This token will be used throughout the codebase to trigger the code quality scan.

TODO: Explain all variables
END_TODO

Check out the cd project

### Prepare Docker Registry
<!-- TODO
This is required for later for the quickstarters, see, e.g. be_spring_boot.yaml
-->
 The Docker registry preparation is needed for several quickstarters, e.g. be_spring_boot. To do so, make sure you have the Docker client binary installed on your machine.

* `minishift addons apply registry-route`
* Run `minishift docker-env` to display the commend you need to execute in order to configure your Docker client.
* Execute the displayed command, e.g. on Windows CMD `@FOR /f "tokens=*" %i IN ('minishift docker-env') DO @call %i`
* `oc login -u developer -n default`
* `oc whoami -t` should show the token for you user
* `docker login -u developer -p `<Token from oc whoami -t>` docker-registry-default.192.168.99.100.nip.io:443`
* `docker pull busybox`
* `docker tag busybox docker-registry-default.192.168.99.100.nip.io:443/openshift/busybox`
* `docker push docker-registry-default.192.168.99.100.nip.io:443/openshift/busybox`

<!-- END TODO -->

### Prepare Rundeck and required Dockerfiles

After configuring the Atlassian tools and Minishift, Rundeck has to be configured as well.
Access [Rundeck](http://192.168.56.31:4440/rundeck), login and open the configuration.

#### Create Quickstarters project
Create a project named `Quickstarters`. The project doesn't need any additional information, so leave all other options blank.

#### Openshift API token
You have to store the API token for the service account in Rundeck, so Rundeck is able to communicate with Openshift.

* In the **Key Storage** section click on **Add or Upload a Key**, choose the Key Type *Password*.
* Copy the token text you saved earlier to the textfield.
* Leave Storage path blank.
* The key has to have the name `openshift-api-token`
* Save the key.

#### CD user private key
For initial code commit the CD user's private key has to be stored in Rundeck, to enable an SSH communication between Rundeck and Bitbucket.

* In the **Key Storage** section click on **Add or Upload a Key**, choose the Key Type *Private key*.
* Enter / Upload the private key generated for the CD user.
* Leave Storage path blank.
* The key has to have the name `id_rsa_bitbucket`
* Save the key.

#### Configure SCM plugins

Within the ods-project-quickstarters create a new branch called `rundeck-changes` - and let it inherit from production
<!--
TODO: verify the branch source is correct!
END_TODO
-->

Open the configuration and go to the **SCM** section. This section is available as soon as you are in the project configuration for the `Quickstarters` project.

##### Setup Import plugin

* Change the **File Path Template** to `${job.group}${job.name}.${config.format}`
* Change the format for the **Job Source Files** to `yaml`
* Enter the SSH Git URL for the `ods-project-quickstarters` repository.
You have to enter valid authorization credentials, stored in Rundeck's key storage. This will be the ` id_rsa_bitbucket` key specified in the previous step.
* Branch: Choose "rundeck-changes"
* In the next step ensure that the regular expression points to yaml files. Change the regexp to `rundeck-jobs/.*\.yaml`
* Change the file path template to `rundeck-jobs${job.group}${job.name}-${job.id}.${config.format}`
* Import the job definitions under job actions.


##### Setup Export plugin
If you use the Github repository, and use as is this step isn't necessary!
If you use your own repository, configure the export plugin in same way as the import plugin, except the file path template - set to `rundeck-jobs/${job.group}${job.name}.${config.format}`

##### Update the job properties

Go to the project page and then configure. Edit the configuration file (using the button) and add the following lines - based on your environment

```INI
# bitbucket https host including url schema
project.globals.bitbucket_host=https\://192.168.56.31
# bitbucket ssh host including url schema
project.globals.bitbucket_sshhost=ssh://git@192.168.56.31:7999
# openshift host including url scheme
project.globals.openshift_apihost=https://192.168.99.100:8443
# openshift host without url scheme - used to grab CA etc
project.globals.openshift_apihost_lookup=192.168.99.100:8443
# openshift nexus host including url scheme
project.globals.nexus_host=http://nexus-cd.192.168.99.100.nip.io/
# public route of docker registry including url scheme
project.globals.openshift_dockerregistry=https://docker-registry-default.192.168.99.100.nip.io:443
# os user and group rundeck is running with
project.globals.rundeck_os_user=root:root
```
### Add shared images
OpenDevStack provides shared images used accross the stack - like the authproxy based on NGINX and lua for crowd

In order to install, create a new project called `shared-services`

Make the required customizations in the `ods-configuration` under **ods-core > shared-images > nginx-authproxy-crowd >  ocp-config > bc.env and secret.env**

and run `tailor update` inside `ods-core\shared-images\nginx-authproxy-crowd`:

and start the build: `oc start-build -n shared-services nginx-authproxy`.

### Configure provisioning application
Clone the provisioning application repository.

Because we disabled anonymous access for nexus, we need to provide some data.

What you need to provide are gradle guild variables. You do this by creating a `gradle.properties` file in the ods-provisioning-app project:

```INI
nexus_url=http://nexus-cd.192.168.99.100.nip.io
nexus_user=developer
nexus_pw=developer
```

If you run the application from your IDE, there is no further configuration needed.

After startup via the IDE the application is available at http://localhost:8088/

You can login in with the Crowd admin user you set up earlier.

### Setup within Openshift

Create 3 openshift projects projects
- `prov-cd` (for the jenkins builder)
- `prov-test` (*production* branch will be built and deployed here)
- `prov-dev` (other branches will be built and deployed here)

Start with `prov-cd` and issue:
```
cd ocp-config/prov-cd
tailor update
```

Add `prov-cd/jenkins` and `prov-cd/default` service accounts with edit rights into -dev & -test projects, so jenkins can update the build config and trigger the corresponding `oc start build / oc update bc` from within the jenkins build.

For the runtime projects (`prov-test` and `prov-dev`) run:
```shell
cd ocp-config/prov-app
tailor update -f Tailorfile.dev
tailor update -f Tailorfile.test
```

Once Jenkins id deployed, you can trigger the build in the `prov-cd/ods-provisioning-app-production` pipeline. Deployment should happen automatically and you can start using the provision app.


## Try out the OpenDevStack
After you have set up your local environment it's time to test the OpenDevStack and see it working.
Open the Provisioning application in your web browser and login with your crowd credentials.

Provision your first project and have a look at OpenShift.
