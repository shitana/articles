# Section 3: Advanced Inventory Techniques

In this section, we’ll explore advanced techniques for managing Ansible inventory. These techniques are essential when dealing with complex environments, dynamic infrastructures, and custom requirements that go beyond the capabilities of standard static and dynamic inventories.

## Dynamic Inventories: Integrating with Cloud Providers and CMDBs

Dynamic inventories allow Ansible to adapt to changes in your infrastructure automatically. This is especially useful in cloud environments, where instances and resources are often ephemeral. Here’s how you can leverage dynamic inventories effectively:

### 1. Cloud Provider Integrations
Ansible provides built-in plugins for popular cloud providers like AWS, Azure, and Google Cloud. These plugins automatically generate inventories based on the current state of your cloud resources.

- **AWS Example**:
  - Ansible’s EC2 plugin can dynamically list instances in your AWS environment. Here’s a simple example:
  ```yaml
  plugin: aws_ec2
  regions:
    - us-east-1
  filters:
    tag:Environment: production
  ```
  - This configuration pulls all EC2 instances tagged with “production” in the `us-east-1` region.

### 2. CMDB Integrations
For organizations with a CMDB (Configuration Management Database), integrating it with Ansible ensures that your inventory is always in sync with your infrastructure’s official records.

- **ServiceNow Example**:
  - Using a ServiceNow plugin, you can query CMDB records to create an inventory dynamically:
  ```yaml
  plugin: servicenow
  instance: your_instance
  username: your_username
  password: your_password
  filters:
    operational_status: 1
  ```

## Creating Custom Dynamic Inventory Scripts

While built-in plugins cover many scenarios, there may be cases where you need a custom dynamic inventory script. These scripts can be written in Python or any language that outputs JSON.

### Example: Custom Script for Docker Containers
Here’s a basic example of a custom dynamic inventory script that lists Docker containers as hosts:

```python
#!/usr/bin/env python

import subprocess
import json

def list_containers():
    result = subprocess.run(['docker', 'ps', '--format', '{{.Names}}'], stdout=subprocess.PIPE)
    containers = result.stdout.decode('utf-8').strip().split('
')
    return containers

def generate_inventory():
    containers = list_containers()
    inventory = {
        "all": {
            "hosts": containers,
            "vars": {
                "ansible_connection": "docker"
            }
        }
    }
    return inventory

if __name__ == "__main__":
    print(json.dumps(generate_inventory(), indent=2))
```

- This script lists all running Docker containers and sets them as hosts in the inventory with Docker as the connection method.

## Grouping Strategies: Nested Groups and Host Patterns

Grouping hosts effectively is crucial for applying roles, playbooks, and variables across different sections of your infrastructure.

### 1. Nested Groups
Nested groups allow you to create hierarchical structures within your inventory. This is useful for applying different configurations at various levels.

- **Example**:
  ```yaml
  all:
    children:
      webservers:
        children:
          nginx_servers:
            hosts:
              web01:
              web02
          apache_servers:
            hosts:
              web03
      dbservers:
        hosts:
          db01:
          db02
  ```

### 2. Host Patterns
Host patterns are a powerful way to target specific hosts within your inventory. Ansible supports various pattern types, such as unions, intersections, and exclusions.

- **Examples**:
  - Target all web servers:
    ```yaml
    - hosts: webservers
    ```
  - Target all but Apache servers:
    ```yaml
    - hosts: webservers:!apache_servers
    ```
  - Target specific hosts by name:
    ```yaml
    - hosts: web01:web02
    ```

## Leveraging Inventory Plugins for Complex Environments

Ansible inventory plugins extend the capabilities of inventories beyond the basic static and dynamic types. These plugins are particularly useful in complex environments where custom data sources or specific organizational requirements are involved.

### 1. External Inventory Plugins
External inventory plugins can be used to pull inventory data from various sources, such as REST APIs, databases, or external files.

- **Example: JSON File Plugin**:
  - The `constructed` plugin allows you to construct inventory based on an external JSON file:
  ```yaml
  plugin: constructed
  hosts:
    plugin: jsonfile
    file: /path/to/your/inventory.json
  groups:
    webservers: "'web' in inventory_hostname"
  ```

### 2. Combining Multiple Plugins
You can combine multiple plugins to create a hybrid inventory that draws from different data sources.

- **Example: Combining AWS EC2 and Static Inventory**:
  ```yaml
  plugin: aws_ec2
  regions:
    - us-east-1
  compose:
    ansible_host: public_ip_address
  groups:
    aws_instances: "'tag:Role' in tags"
  ```
  - You can also include a static inventory file in the same directory, and Ansible will merge them during execution.

By utilizing these advanced inventory techniques, you can ensure that your Ansible playbooks and roles are applied accurately across diverse and dynamic infrastructures. This level of flexibility and control is essential for managing complex environments efficiently.

---

This completes Section 3 on advanced inventory techniques. Let me know if you'd like to proceed with another section or need any adjustments!