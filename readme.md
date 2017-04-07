# Overview

This repository contains instructions to deploy RedHat Openshift on nodes running in AWS.

These instructions relies mostly on scripts that automates much of the host preparation steps :
* Register the RHEL hosts with a subscription
* Activate the proper yum repositories for Openshift
* Install all the pre-requisites to prepare the hosts for the Openshift Advanced installation procedure which relies on Ansible

# Prerequisites

All VMs created must be using the latest RHEL AMI.

* RHEL subscription for Openshift 
* At least one master VM.  The masters shall be `t2.xlarge` VMs.
* At least one node VM.  The nodes shall be `t2.large` VMs.
* All VMs must have at least 30 gigs of root storage.
* All VMs must have one EBS attached on sdb.  That EBS must have at least 30 gig of space.
* All VMs must have one Elastic IP attached.

DNS entries must also be configured because Openshift routes rely on DNS hostnames to route HTTP/HTTPS requests.
The DNS entries must be in the same subdomain.  For example, <subdomain> in below's table could be substituted with 
`openshift.example.com`.

| DNS Entry                                                   | Description                                                       |
| ----------------------------------------------------------- | ----------------------------------------------------------------- |
| <subdomain> IN A <master node elastic IP>                   | This hostname will be used to access openshift's web console      |
| *.<subdomain> IN A <master node elastic IP>                 | Routes matching this wildcard will be resolved to the router's IP |
| master.<subdomain> IN A <master node elastic IP>            | Master node public hostname                                       |
| node1.<subdomain> IN A <Node 1 elastic IP>                  | Node1 public hostname                                             |
| node2.<subdomain> IN A <Node 2 elastic IP>                  | Node2 public hostname                                             |
| master.internal.<subdomain> IN A <master node private IP>   | Master node private hostname                                      |
| node1.internal.<subdomain> IN A <master node private IP>    | Node1 private hostname                                            |
| node2.internal.<subdomain> IN A <master node private IP>    | Node2 private hostname                                            |

# SSL Certificates

The ansible inventory file will require an SSL Server Certificate in the working directory :
* server-cert.pem - The Server certificate chain
* server-cert.key - The unencrypted private key
* ca.crt - The CA that signed the certificate

You can use these utils to create a CA, and then have your own certificate signed :
* [Scripts that creates a CA and sign certificate requests](https://github.com/dickeyf/cert-ca)
* [Scripts that creates server and client certificate requests](https://github.com/dickeyf/cert-requests)

You will have to start with cert-ca, and follow the instructions to create the CA.
Then, you will need to create a server certificate request using cert-request.  You will need to specify SANs for :
* Each hostnames which will be publicly accessible
* For all DNS wildcards which will be publicly accessible HTTP routes.

Once the request is created, follow the instructions to sign it.

# Instructions

## Checkout this repository

```
git clone https://github.com/dickeyf/openshift-aws-install.git
cd openshift-aws-install
```

## Preparing the hosts for Openshift Installation

The RHEL hosts will have to be registered with a Redhat subscription that have access to Openshift Enterprise and have the
proper packages installed.  The setupHosts.sh script will prompt you for your subscriptions credentials, and then automate
these steps :
* Register the hosts with your RHEL subscriptions
* Only activate the repositories for Openshift Enterprise
* Install the required RPMs
* Setup Docker storage backend to use the EBS block device
* Docker-Machine installed on your developer workstation (If you need to push docker images)

Before proceeding, you will have to copy the AWS host's ssh private to this directory in a filename named `id_rsa`.
This will allow the scripts to login on the hosts.

This command will invoke the script which automates the above steps.  Note that there is no limit on how many hosts you
can prepare.  The scripts is host role agnostic, so hosts can be in the list in any order whether there are master or not :
```
./prepareHosts.sh <AWS Host 1 Elastic IP> <AWS Host 2 Elastic IP> <AWS Host 3 Elastic IP> <...>
```

Input your account username and password for your RHEL subscription.  Then, you should see the script making progress.

## Create the Ansible inventory hosts file

The inventory file is created with the help of a script which will prompt you to collect information about the nodes.

```
./createInventory.sh
```

You will need to provide the public and private hostname for all nodes.  You will also need to provide the
default subdomain for routes.

## Deploy Openshift

Once the inventory file is created it is now possible to deploy Openshift.  The deploy.sh script will ssh to the
master node and execute the Ansible playbook to install Openshift on all nodes.

```
./deploy.sh <masterNode IP or hostname>
```

You should see Ansible making progress installing Openshift.

Once this is complete you should be able to connect to the web console at : `https://<master node hostname>:8443`
however you won't be able to login until users are created.

## Creating users and the admin user

To create the users you will need to ssh into the master node, and add the users in htpasswd format.

Start by creating the admin user and giving cluster admin privileges:
```
ssh -i id_rsa ec2-user@<masternode>
sudo -i
cd /etc/origin/master
htpasswd htpasswd admin
oadm policy add-cluster-role-to-user admin admin
```

Then, you can add each users with additional htpasswd invocations :

```
htpasswd htpasswd <new-user>
```

## Using Docker-machine to push docker images to your Openshift environment

If you would like to use Docker images which are not hosted on a Docker repository, then you will need Docker-Machine
to create a Docker host which can load docker image and push them to your Openshift project.

```
docker-machine create --driver virtualbox --engine-insecure-registry docker-registry-default.<subdomain> default
```

This will create a Virtual Box VM running docker on a Linux host.  This Docker engine will have been configured to not
validate SSL certificate for the docker-registry-default.<subdomain> host.  Unless you setup SSL termination behind the
Openshift router, not validating that SSL Certificate will be necessary.

Once the Docker host is created and ready, you can now load the environment variables necessary for the Docker Client :
```
eval $(docker-machine env default)
```

You can now load an image, and push it to an Openshift project like so :

```
# First login to Openshift if you aren't already.
oc login <subdomain>:8443 --username=<user> --password=<password>
# Login Docker to Openshift's repository, using your username, and token (returned by "oc whoami -t")
docker login docker-registry-default.<subdomain> --username=<user> --password=`oc whoami -t`
docker load -i <docker-image>.tgz
# Assuming the docker image is named <image> and have the tag <tag>, the following command will create a tag
# with a repository reference:
docker tag <image>:<tag> docker-registry-default.<subdomain>/<project-name>/<image>:<tag>
# This tagged image can now be pushed
docker push docker-registry-default.<subdomain>/<project-name>/<image>:<tag>
```

## Running a demonstration application on your new environment

At this point, your environment is ready to be used.

This project can be used as an example : [Quickstart guide to use Solace's VMR in Openshift](https://github.com/SolaceLabs/solace-openshift-quickstart)