# Section 4: Inventory Management in Gitlab CI/CD Pipelines

In modern DevOps practices, integrating Ansible with Continuous Integration and Continuous Deployment (CI/CD) pipelines is essential for achieving automated, reliable, and consistent deployments. This section will focus specifically on integrating Ansible inventory management with GitLab CI, a popular CI/CD tool used by many organizations.

GitLab CI/CD pipelines allow you to automate the entire deployment process, from code integration to deployment across multiple environments. When using Ansible within GitLab CI, managing your inventory effectively is crucial to ensure that deployments are correctly targeted and variables are properly managed.

## Understanding GitLab CI/CD Pipeline Execution and SSH Configuration

In GitLab CI/CD, each stage of your pipeline is made up of one or more jobs. These jobs are executed by GitLab runners, which are isolated environments where your commands are run. The recommended setup for these runners is to use Docker containers, providing a clean and consistent environment for each job.

>Key Points about GitLab CI Job Execution:

- **Isolated Environment**: Each job within a stage runs inside a fresh environment provided by a GitLab runner. This environment is essentially a new "machine" that only includes the predefined variables provided by GitLab, such as the branch name, commit hash, and pipeline ID.
- **No Persistent Configuration**: Because the environment is isolated and temporary, any configuration needed for your job must be set up at the beginning of the job. For example, if your job needs to connect to a remote server via SSH, you will need to configure the SSH client within this environment.
- **SSH Configuration for Ansible**: To enable SSH access for Ansible within a GitLab CI job, you must prepare the SSH client by adding the necessary SSH keys and configuration. This setup ensures that Ansible can connect to your target machines during the deployment process.

## 0. Requirements:

### Private Key Authorization
- Store a private key that is authorized on the destination machines as a GitLab CI/CD variable. This key will be used for SSH connections during the Ansible tasks.
![alt text](<img/Screenshot from 2024-08-29 15-31-01.png>)

### Script to Handle Private Key Passphrase
- If your private key is protected by a passphrase, you will need a script to handle this passphrase. The passphrase should be stored as a GitLab CI/CD variable.

Example script to add the SSH key with a passphrase:

```bash
#!/bin/bash -e
# Path: ./cd/add_ssh_key_with_passphrase.sh
if [ -z "$SSH_PASSPHRASE" ]
then
  echo "SSH passphrase is empty"
  exit 1
fi

/usr/bin/expect << EOF
spawn ssh-add /root/.ssh/id_rsa
expect "Enter passphrase for /root/.ssh/id_rsa:"
send "${SSH_PASSPHRASE}
";
expect "Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)"
EOF
```

- This script uses the `expect` command to automate the process of entering the passphrase for the private key.

### Ansible Configuration File
- Place an Ansible configuration file (`ansible.cfg`) in the root directory of your repository. This file configures various Ansible settings to ensure smooth operation within the GitLab CI environment.

Example `ansible.cfg`:

```ini
[defaults]
interpreter_python = auto_silent
retry_files_enabled = False
forks = 8
host_key_checking = False
timeout = 60
force_valid_group_names = ignore

[ssh_connection]
pipelining = true
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o PreferredAuthentications=publickey
```

- **Key settings**:
  - `interpreter_python = auto_silent`: Ensures Python is automatically selected.
  - `host_key_checking = False`: Disables SSH host key checking to avoid prompts.
  - `pipelining = true`: Enables SSH pipelining for faster execution.
  - `ssh_args`: Configures SSH to use public key authentication with session persistence.

By following these steps, you can ensure that your Ansible tasks run smoothly within a GitLab CI pipeline, with secure and efficient SSH connections to your target machines.

## 1. Example of Configuring SSH within a GitLab CI Job

To begin, you need to define your CI/CD pipeline in a `.gitlab-ci.yml` file. This file will specify the stages, jobs, and scripts to be executed during the pipeline. Here’s a basic example that includes Ansible for deploying an application:

```yaml
deploy_production:
  stage: deploy
  variables: 
    ANSIBLE_HOST_KEY_CHECKING: "False"
    ANSIBLE_CONFIG: "./ansible.cfg"
  before_script:
    # SSH Key Setup
    - mkdir -p /root/.ssh && chmod -R 700 /root/.ssh
    - cp ${SSH_PRIVATE_KEY} /root/.ssh/id_rsa && chmod 600 /root/.ssh/id_rsa
    - eval "$(ssh-agent -s)"
    # Handle SSH private key with passphrase
    - ./cd/add_ssh_key_with_passphrase.sh
    - ssh-add -l
  script:
    - ansible-playbook -i inventory/production/hosts.yml deploy.yml
  only:
    - master
```

In this example:
- **${SSH_PRIVATE_KEY}** : This variable should be already declared in CICD variable (type file), in Gitlab project setting.
- **SSH Key Setup**: The job starts by creating the `.ssh` directory, adding the SSH private key (stored in a GitLab CI variable), and setting appropriate permissions.
- **Host Verification**: The `ssh-keyscan` command adds the target server to the known hosts, allowing SSH connections to be made without manual host verification prompts.
- **Running Ansible**: Finally, Ansible is run with the prepared SSH configuration, allowing it to connect to the target server and perform the deployment.
- The pipeline consists of a single `deploy` stage.
- The `deploy_production` job runs the Ansible playbook using the production inventory file.
- The job is triggered only when changes are pushed to the `master` branch.

## 2. Storing and Managing Sensitive Data

Sensitive data such as passwords, API keys, and other secrets should never be hardcoded in your inventory files or playbooks. GitLab CI provides several ways to securely manage this data:

### a. GitLab CI/CD Secrets
You can store sensitive information as CI/CD variables in GitLab. These variables can be masked and are not exposed in the pipeline logs. To use these variables in your Ansible playbooks:

```yaml
deploy_production:
  stage: deploy
  script:
    - ansible-playbook -i inventory/production/hosts.yml deploy.yml --extra-vars "db_password=$DB_PASSWORD"
  only:
    - master
  variables:
    DB_PASSWORD: ${DB_PASSWORD}
```

### b. Ansible Vault
For additional security, you can use Ansible Vault to encrypt sensitive variables and files. Ansible Vault allows you to encrypt entire files or specific variables within your inventory. During the GitLab CI pipeline execution, you can decrypt the vault using a password stored as a GitLab CI/CD variable.

Example:
```yaml
deploy_production:
  stage: deploy
  script:
    - ansible-playbook -i inventory/production/hosts.yml --vault-password-file <(echo $VAULT_PASSWORD) deploy.yml
  only:
    - master
  variables:
    VAULT_PASSWORD: ${VAULT_PASSWORD}
```

## 3. Best Practices for GitLab CI Integration

To ensure smooth and reliable deployments with GitLab CI and Ansible, consider the following best practices:

- **Use Branch-Based Environments**: Leverage GitLab CI’s ability to create branch-based environments, which allows for parallel development and testing across different branches without affecting the main production environment.
  
- **Automate Inventory Updates**: If your infrastructure is dynamic, automate inventory updates using GitLab CI pipelines. This ensures that your inventory files are always in sync with the current state of your infrastructure.

- **Version Control Your Inventory**: Keep your Ansible inventory files under version control within the same Git repository as your playbooks. This ensures that changes to your inventory are tracked and can be rolled back if necessary.

- **Test Locally Before Committing**: Always test your Ansible playbooks and inventory changes locally before pushing them to the GitLab CI pipeline. This reduces the risk of errors during deployment.

- **Monitor and Rollback**: Use GitLab’s monitoring and rollback features to keep track of deployments and quickly revert changes if something goes wrong.

By following these guidelines, you can create a robust and flexible CI/CD pipeline that integrates seamlessly with Ansible, ensuring reliable and efficient deployments across your environments.

---

This concludes Section 4, focusing on integrating Ansible inventory management with GitLab CI. Let me know if you need further details or any adjustments!