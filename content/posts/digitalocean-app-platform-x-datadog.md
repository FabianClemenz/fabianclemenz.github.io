+++
title = "DigitalOcean App Platform X Datadog"
authors = ["Fabian Clemenz"]
description = "How to get Monitoring up and running"
tags = ["digitalocean", "datadog", "monitoring", "server", "apm", "firewall"]
categories = ["infrastructure", "monitoring"]
slug = "digitalocean-app-platform-x-datadog"
date = 2025-11-26T08:55:34+01:00
draft = false
+++
This article was first published on our company website. [https://devsuit.de/ueber-uns/aktuelles/digitalocean-app-platform-x-datadog](https://devsuit.de/ueber-uns/aktuelles/digitalocean-app-platform-x-datadog)

In this article I will show how to connect services inside the DigitalOcean App Platform with Datadog. By the time of writing (**07.11.2025**) there is no direct integration except for logging. But as soon as you need APM or Error Tracking, you have no choice but to build your own solution.

## Prerequisites

The following components are needed:

* DigitalOcean VPC
* DigitalOcean Firewall
* DigitalOcean Droplet

On the droplet, the Datadog Agent is run as a docker container. It receives traces and other application data and sends them to Datadog. Since the Datadog Agent's port on the server has been opened, access must be secured so that only the intended apps can use the agent. This is done by creating a VPC and then allowing access to port **8126** (the Datadog Agent port) through a firewall only from this VPC. All apps use the same VPC and can therefore communicate with the agent.

## Virtual Private Cloud (VPC)

A VPC is a private, isolated network that you create within the DigitalOcean infrastructure. It allows you to run resources (such as Droplets, databases, etc.) in a virtual network segment that is separated from others - similar to how it works in AWS or GCP.

### Create the VPC

Go to **Networking → VPC** and click **Create VPC Network**. In the next window, select the appropriate datacenter region, let DigitalOcean generate the IP range (default setting), and give the VPC a name and description.

The first VPC you create automatically becomes the default VPC. Every newly created component will be assigned to this VPC by default.

## Server Installation

After the basic server configuration (see my article [Basic Server Security](https://fabianclemenz.github.io/posts/basic-server-security)), the Datadog Agent port also needs to be opened. This is done with the command `sudo ufw allow 8126`. Since the Datadog Agent will be run as a Docker container, Docker must be installed on the server. Docker provides a good guide for this: [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/).

After installation, create a folder inside the `/opt` directory so that any user can configure the agent, using the command: `sudo mkdir /opt/datadog-agent`.

I like to use Docker Compose to configure and run containers. The following file can be used for this:

```yaml
services:
  dd-agent:
    image: datadog/agent:latest
    container_name: datadog-agent
    ports:
      - 8126:8126
    restart: always
    environment:
      DD_SITE: "<datadog-site>"
      DD_API_KEY: "<your-api-key>"
      DD_APM_ENABLED: true
      DD_LOGS_ENABLED: true
      DD_CONTAINER_EXCLUDE_LOGS: "name:datadog-agent"
      DD_APM_NON_LOCAL_TRAFFIC: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
```

Additional settings can be configured based on the Datadog documentation. The important line is **DD_APM_NON_LOCAL_TRAFFIC**, which allows traces from external sources outside the server.

The Datadog Agent is started with `sudo docker compose up -d`.

## Firewall

With the firewall in DigitalOcean, incoming and outgoing traffic can be controlled. The firewall can be assigned to multiple components, which then share the same network rules. The firewall operates at the network level within the DigitalOcean infrastructure.

### Create the Firewall

Go to **Networking → Firewall** and click **Create Firewall**. In the next window, give the firewall a descriptive name. By default, the inbound rules only allow the SSH port from all IPv4 and IPv6 addresses. This setting must be changed so the server remains accessible via SSH. To do this, set the type to **Custom** and enter the desired SSH port (see [server installation](#server-installation)) as the port.

Next, create a new rule. Set the type to Custom, set the port to **8126** (the Datadog Agent port), and then enter the VPC's IP range (e.g., `10.114.0.0/20`) as the source.

After that, you can select the previously created Droplet under **Apply to Droplets**.

## Configure the Application

For the application to send traces to the newly created Datadog Agent, the private IP address of the server (e.g., `10.114.0.2`) must be set as the **DD_AGENT_HOST**. This ensures that the library (e.g., **ddtrace**) sends the data not to localhost but to the Datadog Agent running on the server.

In addition, the app must be added to the created VPC. This can be done within the app under **Settings → Region**.

## Conclusion

Due to the limitations of the DigitalOcean App Platform, which prevents the native use of the Datadog Agent, and the lack of direct Datadog integration in DigitalOcean, the approach shown here is the only way to monitor applications within the DigitalOcean App Platform using Datadog.
