+++
date = '2023-12-31T15:11:05-04:00'
title = 'Dockerizing CDKTF with Python'
+++

Cloud Development Kit for Terraform ([CDKTF](https://developer.hashicorp.com/terraform/cdktf)) is a framework that allows you to use familiar programming languages to define and provision infrastructure using Terraform. CDKTF supports multiple languages, including Python, which is a popular choice for DevOps engineers.

Unfortunately, there is currently no official Docker image for it. Using a Dockerfile, you can ensure that your CDKTF application has all the dependencies and configurations needed to run smoothly and consistently. In this blog post, I will show you the Dockerfile I built for my project that uses CDKTF with Python to create a Kubernetes cluster.

Let’s get started!

### File Structure

To speed up the development process and docker build time, I have found the following file structure the most appropriate for a CDKTF project:

```yaml
Git Repo
|- app
| |- <Stack#1>.py
| |- <Stack#1>_tests.py
| |- ...
| |- <Stack#N>.py
| |- <Stack#N>_tests.py
|- cdktf.out
| |- stacks
| |- |- <Stack#1>
| |- |- |- .terraform.lock.hcl
| |- |- ...
| |- |- <Stack#N>
| |- |- |- .terraform.lock.hcl
| Dockerfile
| Pipfile
| Pipfile.lock
| cdktf.json
```

The stacks include the parent directory to the Python system path, so the imports to the CDK and TF providers modules work nicely.

### Docker file

My project contains other docker images that also need Python, and I want to reuse the same base image for all of them. This way, I can avoid duplicating the installation steps and reduce the size and complexity of my docker images.

**Base Image (**[**source**](https://github.com/hoaraujerome/kubernetes-the-hard-way-on-aws/blob/main/Dockerfile.base)**)**

```dockerfile
# ***
# Ubuntu is used for consistency with other use cases of the project.
# Otherwise a minimal and more secured OS base image like Alpine is
# a better choice.
# ***
FROM ubuntu:22.04

# ***
# Python Installation
# ***
ENV PYTHON_VERSION=3.12
# non interactive needed for software-properties-common
ENV DEBIAN_FRONTEND=noninteractive

# software-properties-common required for add-apt-repository
# ppa required for python 3.12 https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa/
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    apt-get install -y software-properties-common && \
    add-apt-repository ppa:deadsnakes/ppa && \
    apt-get update && \
    apt-get install -y python$PYTHON_VERSION && \
    apt-get clean all

RUN python3 --version | awk '{if ($2 < 3.10) exit 1;}'
```

**CDKTF Image (**[**source**](https://github.com/hoaraujerome/kubernetes-the-hard-way-on-aws/blob/main/provisioning/Dockerfile)**)**

```dockerfile
FROM base:local

# ***
# cdktf Installation (requires Node.js)
# ***
ENV TERRAFORM_VERSION=1.5.5-1
ENV NODE_MAJOR=20
ENV NODE_VERSION=20.5.1-1nodesource1
ENV CDKTF_CLI_VERSION=0.18.0
ENV PIP_PIPENV_VERSION=2023.11.15

# make & g++ required for cdk-cli
RUN apt-get update && \
    apt-get install -y make g++ && \
    curl -fsSL https://apt.releases.hashicorp.com/gpg | \
      gpg --dearmor | \
      tee /usr/share/keyrings/hashicorp-archive-keyring.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
      https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
      tee /etc/apt/sources.list.d/hashicorp.list && \
    apt-get update && \
    apt-get install -y terraform=$TERRAFORM_VERSION && \
    curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | \
      gpg --dearmor -o /usr/share/keyrings/nodesource.gpg && \
    echo "deb [signed-by=/usr/share/keyrings/nodesource.gpg] \
      https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | \
      tee /etc/apt/sources.list.d/nodesource.list && \
    apt-get update && \
    apt-get install -y nodejs=$NODE_VERSION && \
    apt-get clean all

ENV CDKTF_HOME_DIR=/home/cdktf

RUN useradd -m cdktf -d $CDKTF_HOME_DIR

ENV PATH=$CDKTF_HOME_DIR/.local/bin:$PATH

# Best practice: do not use root
USER cdktf

WORKDIR $CDKTF_HOME_DIR

RUN terraform --version  && \
    node --version || \
    exit 1

RUN npm install cdktf-cli@$CDKTF_CLI_VERSION

RUN npx cdktf --version || \
    exit 1

# TODO pin version instead
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && \
    python3 get-pip.py --user && \
    rm get-pip.py

RUN python3 -m pip -V || exit 1

RUN python3 -m pip install --user pipenv==$PIP_PIPENV_VERSION

# init CDKTF application to optimize build process time
COPY --chown=cdktf:cdktk Pipfile Pipfile.lock cdktf.json ./
RUN pipenv install

# generate CDK Constructs for Terraform providers
# specified in cdktf.son. Can take several minutes.
# alternative: commit the imports directory generated
RUN npx cdktf get

# cdktf.out directory contains terraform dependency lock files
COPY --chown=cdktf:cdktk cdktf.out/ cdktf.out

# copy custom files that performs provisioning tasks 
COPY --chown=cdktf:cdktk Makefile provisioning.sh ./

ENTRYPOINT [ "make" ]
```

Finally, execute the following command to mount the stack source code as a volume and run CDKTF against them:

```bash
docker run \
    --rm \
    -v ./provisioning/app:/home/cdktf/app \
    cdktf:local \
        <args>
```

In upcoming posts, I will share how I use CDKTF to provision a Kubernetes cluster on AWS. You can find more details about this project [here](https://github.com/hoaraujerome/kubernetes-the-hard-way-on-aws/tree/main).

As I delve deeper into CDKTF, I am open to learning more. Your feedback is valuable, so feel free to comment below or connect with me on [GitHub](https://github.com/hoaraujerome).

Thank you for reading, and happy coding!
