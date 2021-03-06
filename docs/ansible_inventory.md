# Ansible Inventory Files

## Brent Resource Package

The "config" subdirectory in the Brent resource package driver root directory contains Ansible inventory and ancillary configuration files. The Ansible Lifecycle Driver expects an inventory file named "inventory" in this directory (for SSH connections using the [Ansible SSH connection plugin](https://docs.ansible.com/ansible/latest/plugins/connection.html#ssh-plugins)) and/or a file named "inventory.k8s" (for K8s connections using the [Ansible kubectl connection plugin](https://docs.ansible.com/ansible/latest/plugins/connection/kubectl.html)).

The driver determines which inventory file to use based on the deployment location type (i.e. "deploymentLocation.type") in the lifecycle request. If the type is "Kubernetes" the driver will expect an "inventory.k8s" file. If the type is anything other than "Kubernetes", an SSH connection and "inventory" file are expected.

Note: if you wish to split your inventory out into separate host variable files then you may do so. For example:

```
config
  host_vars
    host1.yml
  inventory
  inventory.k8s
```

## Variable Substitution

The Ansible Lifecycle Driver supports the substitution of LM properties in inventory files under the "config" directory, using [Jinja2 template variables](https://jinja.palletsprojects.com/en/2.10.x/templates/#variables). The following properties are available:

* properties: a dictionary of LM request properties.
* system_properties: a dictionary of system properties.
* dl_properties: a dictionary of deployment location properties.

The system properties comprise the following:

| Property Name  | Description |
| ------------------------- | -------------- |
| resourceId                | The LM resource instance id |
| requestId                         | The LM orchestration request ID     |
| metricKey                         | The LM resource metric key (if set)     |
| resourceManagerId                         | The LM resource manager id     |
| deploymentLocation                         | The name of the LM deployment location name in which the resource instance resides    |
| resourceType                         | The resource instance type |

In addition, any LM orchestration request context properties are added in here.

For example, the following inventory file substitutes the LM property "properties.mgmt_ip_address" for "ansible_host":

```
---
ansible_user: ubuntu
ansible_host: {{ properties.mgmt_ip_address }}
ansible_ssh_pass: ubuntu
ansible_connection: ssh
ansible_become_pass: ubuntu
ansible_sudo_pass: ubuntu
```

Note that both square brackets and dot notation are supported during property substitution. For example, "properties['prop1']" and "properties.prop1" are equivalent.

## Support for SSH and K8s Connections

The Ansible Lifecycle Driver supports inventory for SSH and K8s connections, as described in the introduction.

## SSH Connections

An example SSH inventory file (which must be located in the Brent resource package at ansible/config/inventory):

```
---
ansible_user: ubuntu
ansible_host: {{ properties.mgmt_ip_address }}
ansible_ssh_pass: ubuntu
ansible_connection: ssh
ansible_become_pass: ubuntu
ansible_sudo_pass: ubuntu
```

In this example, the ansible_host is set to an LM property value.

## K8s Connections

It is assumed that, for K8s inventory, the infrastructural part (that is, the pods and other K8s infrastructure) have been created using the [K8s VIM Driver](https://github.com/accanto-systems/k8s-vim-driver). In particular, the K8s VIM driver ensures that the following properties have been set which are required by the Ansible Lifecycle Driver K8s connection handling:

* properties.host: the pod name to connect to

An example K8s inventory file (which must be located in the Brent resource package at ansible/config/inventory.k8s):

```
---
ansible_connection: kubectl
ansible_kubectl_pod: {{ properties.host }}
ansible_kubectl_namespace: {{ dl_properties['k8s-namespace'] }}
ansible_kubectl_kubeconfig: {{ dl_properties.kubeconfig_path }}
```

The following table describes the K8s inventory properties:

| Name            | Value | Required                           | Detail                                                                                                                     |
| --------------- | ------- | ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| ansible_connection      | kubectl       | Y                                  | The Ansible connection type, must be "kubectl" for K8s connections |
| ansible_kubectl_pod | properties.host    | Y                                  | The pod to connect to
| ansible_kubectl_namespace     | -       | Y | The K8s namespace in which the pod is running. This could be set either from "properties" or "dl_properties" (see [variable substitution](#variable-substitution))
| ansible_kubectl_kubeconfig    | dl_properties.kubeconfig_path | Y | A kubeconfig file automatically generated by the Ansible Lifecycle Driver

## Using a Jumphost in Inventory

It is often necessary to use a [Jumphost](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-configure-a-jump-host-to-access-servers-that-i-have-no-direct-access-to) to access a VM using SSH. It is possible to do this with the Ansible Lifecycle Driver using an inventory like this:

```
---
ansible_user: ubuntu
ansible_host: {{ properties.mgmt_ip_address }}
ansible_ssh_pass: ubuntu
ansible_connection: ssh
ansible_become_pass: ubuntu
ansible_sudo_pass: ubuntu
ssh_with_jumphost: "-o 'UserKnownHostsFile=/dev/null' -o StrictHostKeyChecking=no -o ProxyCommand='sshpass -p {{ properties.jumphost_password }} ssh -o 'UserKnownHostsFile=/dev/null' -o StrictHostKeyChecking=no -W %h:%p {{ properties.jumphost_username }}@{{ properties.jumphost_ip }}'"
ssh_without_jumphost: "-o 'UserKnownHostsFile=/dev/null' -o StrictHostKeyChecking=no"
ansible_ssh_common_args: "{{ ssh_with_jumphost if properties.jumphost_ip is defined else ssh_without_jumphost }}"
```

If the LM properties have defined a "jumphost_ip" property this configuration will construct "ansible_ssh_common_args" using ProxyCommand to proxy the SSH connection through the jumphost, using jumphost properties from the LM request. If not, a standard direct SSH connection is configured.