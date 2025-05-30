+++
title = "how to install docker on linux mint"
date = 2025-05-30
+++

This guide explains how to install Docker on Linux Mint/Ubuntu systems.

## Install Required Packages

First, update your package index and install the necessary packages:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```

## Add Docker's Official GPG Key

Create the keyrings directory and add Docker's GPG key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

## Add the Docker Repository

Add Docker's repository with the correct Ubuntu code name:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  jammy stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

## Update Package Index

Update the package index to include Docker packages:

```bash
sudo apt-get update
```

## Install Docker

Finally, install Docker and its components:

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Verify Installation

To verify that Docker is installed correctly, run:

```bash
sudo docker run hello-world
```

## Optional: Add User to Docker Group

To run Docker commands without sudo, add your user to the docker group:

```bash
sudo usermod -aG docker $USER
```