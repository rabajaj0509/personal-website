---
author: "Rahul Bajaj"
title: "Managing Containers with Systemd"
description: "To reduce the need for manual intervention, use systemd to manage container-based applications on a health check."
date: 2023-07-18T15:02:42-04:00
categories: ["containers"]
tags: ["containers", "systemd"]
cover:
  image: "/images/systemd-service.jpg"
  alt: "Systemd"
---

Containers are inherently ephemeral, making them more difficult to manage than traditional programmes operating on virtual or bare metal servers. Container monitoring, on the other hand, is a critical capability for applications based on current microservices architectures in order to achieve maximum performance.

Containerized applications frequently necessitate monitoring. Performing health-checks is one technique to keep these containers up and running at all times. The usage of the `curl` command to verify if the application within the container is still up and running is one of the approaches I've come across for doing health-checks. The `cronjob` utility is used to perform curl commands on the container application at a periodic basis. If the container is not in a running state, a configuration management tool like Ansible can be used to start/restart it. Although this method does not need direct human intervention, it does not seem to have a mechanism for `logging` the event i.e. reason for downtime of the application/container. Therefore, it is not the best option for enhancing the container's self-healing capabilities.

Instead, we make use of the `systemd` utility of Linux which is ubiquitous across most of the Linux distributions. The systemd utility makes it easy to dynamically start/restart/stop configuration files for each deamon.

In this blog, we will:
1. Create an Ansible role.
2. Define the required variables.
3. Create the service file usign a Jinja2 template
4. Manage unit files within the containers using a playbook

With the aid of systemd, we'll start an apache container and control its daemon. The `Jinja2 templates` are used to construct the unit file. We save the template files in the `/usr/lib/systemd/system/` directory after they've been created. We can then use the`systemd` ansible module to administer the deamon once the files have been stored in the correct location with the required permissions.

# Create an Ansible role

```
cd roles
mkdir -p apache/{defaults,tasks,templates,handlers}
```


# Define the required variables

```
---
apache_image: "httpd"
apache_image_tag: "latest"
```

# Create the service file usign a Jinja2 template

```
[Unit]
Description=run apache container
After=network.target docker.service
Requires=docker.service

[Service]
ExecStartPre=-/usr/bin/docker stop apache
ExecStartPre=-/usr/bin/docker rm apache
ExecStartPre=/usr/bin/docker pull {{ apache_image }}:{{ apache_image_tag }}
ExecStart=/usr/bin/docker run \
  --port='8080:8080' \
  --name=apache \
  --volume={{ apache_data_dir }}:{{ apache_data_container_dir }}:ro \
  --restart=always \
  --log-driver=journald \
  {{ apache_image }}:{{ apache_image_tag }}
Restart=always

[Install]
WantedBy=multi-user.target
```

Let's break down the important parts:

1. Unit Section:
  * **Description**: Describes the purpose of the service.
  * **After**: Defines the dependencies that must be started before this service.
  * **Requires**: Specifies the units that this unit depends on.

2. Service Section:
  * **ExecStartPre**: Commands to be executed before starting the service. Here, it stops and removes any existing Docker container named 'apache'. Then it pulls the specified Docker image (using variables `apache_image` and `apache_image_tag` provided during template rendering).

  * **ExecStart**: Command to start the Docker container with various options:

    *--port='8080:8080'*: Maps port 8080 from the container to the host.  
    *--name=apache*: Assigns the name 'apache' to the container.  
    *--volume={{ apache_data_dir }}:{{ apache_data_container_dir }}*:ro: Mounts a directory from the host to a directory in the container in read-only mode.  
    *--restart=always*: Specifies that the container should always restart if it stops.  
    *--log-driver=journald*: Defines the logging driver for the container.  
    *{{ apache_image }}:{{ apache_image_tag }}*: Specifies the Docker image and its tag to be used.  

  * **Restart**: Defines the restart policy for the service, set to 'always'.

3. Install Section:
  * **WantedBy**: Specifies the target that this service should be enabled for.

# Manage unit files within the containers using a playbook


```
/roles/apache/handlers/main.yml

---
- name: restart container-apache
  systemd:
    name: container-apache.service
    daemon_reload: yes
    state: restarted
```

```
/roles/apache/tasks/main.yml

---
- name: set selinux boolean value
  seboolean:
    name: container_manage_cgroup
    state: yes
    persistent: yes

- name: create the apache container unit file
  template:
    src: container-apache.service.j2
    dest: /usr/lib/systemd/system/container-apache.service
    owner: root
    mode: 0660
  notify: restart container-apache

- name: link and enable container-apacahe
  systemd:
    name: container-apache
    enabled: yes

- name: run apache container
  systemd:
    name: container-apache
    state: started
```