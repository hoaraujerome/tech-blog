+++
date = '2022-02-07T15:14:42-04:00'
title = 'Jenkins As Code With Packer, Ansible, Terraform, and AWS'
+++

## What Will We Cover

1.  Build an OS image for AWS with a Jenkins ready to use
    
2.  Provision an EC2 instance to host the Jenkins server
    

See all the code for this article here: [https://github.com/hoaraujerome/devops\_cicd](GitHub)

## Using Packer with Ansible to build an AMI image

Packer tool is responsible for creating the OS image, while Ansible is responsible for installing everything we need on this image.

![buildAMI](https://github.com/hoaraujerome/devops_cicd/raw/main/misc/devops_cicd-Packer.jpg)

The final version of the image has Jenkins (with "GIT", "Pipeline", and "Pipeline: AWS Steps" plugins), Docker, AWS CLI, Terraform, and Java installed for running Jenkins pipelines hosted on GitHub.

### Packer Configuration

The configuration used is very standard. For simplicity, I decided to use Amazon Linux 2 AMI (HVM) as the source for the AMI and the Ansible Local Provisioner. The latter requires installing Ansible through the shell provisioner on the guest/remote machine before running the playbook.

### Ansible Playbook

I’m listing here the roles and their responsibilities:

*   AWS CLI: download and install
    
*   Docker: install Docker via YUM and add jenkins user to docker group
    
*   Terraform: download and install
    
*   Jenkins: install Jenkins and its dependencies via YUM, apply custom configuration including basic security, install plugins, create AWS credentials, and create a Jenkins job.
    

## Using Terraform to provision a Jenkins server

Now that we have an image (AMI) with a Jenkins ready to use, we just need to spin up an EC2 instance with this image. This instance must of course be deployed in a public subnet. Terraform is used to provision this infrastructure with code.

![provisionJenkins.jpg](https://github.com/hoaraujerome/devops_cicd/raw/main/misc/devops_cicd-Terraform.jpg)
