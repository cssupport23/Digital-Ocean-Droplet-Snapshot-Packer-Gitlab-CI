# DigitalOcean Snapshot with Packer and Ansible

This project is designed to build a DigitalOcean snapshot using [Packer](https://www.packer.io/) and provision it with [Ansible](https://www.ansible.com/). The snapshot contains a Next.js application setup with PM2, ensuring the latest version of the app repository is installed and configured.

## Prerequisites

- DigitalOcean account with an API token stored in AWS SSM.
- AWS CLI installed to retrieve parameters from SSM.
- GitLab CI/CD pipeline setup.
- [Packer](https://www.packer.io/) installed.
- [Ansible](https://www.ansible.com/) installed.
- SSH access to the DigitalOcean droplet.

## Steps

### 1. GitLab CI Configuration

In your GitLab repository, define the `.gitlab-ci.yml` file:

```yaml
image: 
  name: hashicorp/packer:full
  entrypoint: [""]

variables:
  DIGITAL_O: '/token/gitlab/digitalocean'
  CICD_TOK_SSM: 'cicdtoken'
  VERSION_SSM: '/nextjs-app/version'
  CISD_USER_SSM: '/cicd/user'
  GIT_REPO: /aws/nextjsapp.git
  GITL_DOMAIN: /gitlab/domain

stages:
  - build

before_script:
  - |
    cat /etc/os-release
    apk update
    apk upgrade 
    apk add openssh-client ansible aws-cli git
    export CICD_TOKEN=$(aws ssm get-parameter --name "${CICD_TOK_SSM}"  --query 'Parameter.Value' --output text)
    export VERSION=$(aws ssm get-parameter --name "${VERSION_SSM}"  --query 'Parameter.Value' --output text)
    export CICD_USER=$(aws ssm get-parameter --name "${CISD_USER_SSM}"  --query 'Parameter.Value' --output text)
    export DIGITALOCEAN_TOKEN=$(aws ssm get-parameter --name "${DIGITAL_O}"  --query 'Parameter.Value' --output text)
    export GITLAB_DOMAIN=$(aws ssm get-parameter --name "${GITL_DOMAIN}"  --query 'Parameter.Value' --output text)
    export REPO_URL="http://$CICD_USER:$CICD_TOKEN@$GITLAB_DOMAIN/$GIT_REPO"
    mkdir nj_repo
    git clone --branch $VERSION $REPO_URL nj_repo
    cd nj_repo
    export COMMIT_HASH=$(git rev-parse HEAD)
    cd ..
    echo "Updating group_vars/all.yml with the fetched data..."
    sed -i "s|personal_access_token:.*|personal_access_token: $CICD_TOKEN|" ./ansible/group_vars/all.yml
    sed -i "s|version:.*|version: $COMMIT_HASH|" ./ansible/group_vars/all.yml  
    sed -i "s|username:.*|username: $CICD_USER|" ./ansible/group_vars/all.yml
    sed -i "s|app_rep:.*|app_rep: $REPO_URL|" ./ansible/group_vars/all.yml
    cat ./ansible/group_vars/all.yml
    packer --version
    packer plugins install github.com/digitalocean/digitalocean

build:
  stage: build
  script:
    - |
      echo "Running Packer"
      packer validate nextjs_snaphot.json
      packer build ./nextjs_snaphot.json | tee packer_output.log
  artifacts:
    paths:
      - packer_output.log
  tags:
    - digitalocean
```

### 2. Packer Configuration

The Packer template (`nextjs_snaphot.json`) is set up to create a snapshot on DigitalOcean:

```json
{
    "variables": {
      "do_token": "{{env `DIGITALOCEAN_TOKEN`}}"
    },
    "builders": [
      {
        "type": "digitalocean",
        "api_token": "{{user `do_token`}}",
        "region": "nyc3",
        "size": "s-1vcpu-1gb",
        "image": "ubuntu-22-04-x64",
        "ssh_username": "root",
        "snapshot_name": "digitalocean-ami-{{timestamp}}"
      }
    ],
    "provisioners": [
      {
        "type": "ansible",
        "playbook_file": "ansible/playbook.yml"
      }
    ]
}
```

### 3. Ansible Role

The Ansible playbook provisions the droplet by setting up Node.js, PM2, and cloning the latest version of the application.

```yaml
---
# ansible/playbook.yml
- name: Prepare host
  hosts: all
  become: true  
  gather_facts: false  
  roles:
    - pm2_setup

# ansible/roles/pm2_setup/tasks/main.yml
---
- name: Update APT package index
  apt:
    update_cache: yes
  become: yes

- name: Install curl (required for Node.js installation)
  apt:
    name: curl
    state: present
  become: yes

- name: Add NodeSource repository for Node.js 20.x
  shell: "curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -"
  become: yes

- name: Install Node.js and npm
  apt:
    name: nodejs
    state: present
  become: yes

- name: Install PM2 globally using npm
  npm:
    name: pm2
    global: yes
  become: yes

- name: Clone the App repository using HTTP with credentials
  git:
    repo: "{{ app_rep }}"
    dest: /var/www/html
    version: "{{ version }}"
    clone: yes
    update: yes
    force: yes

- name: Verify directory and user
  shell: |
    pwd
    whoami
    ls -alh /var/www/html
  register: command_output

- name: Print the output of the shell command
  debug:
    var: command_output.stdout

- name: Run npm install in the application directory
  command: npm install
  args:
    chdir: /var/www/html
```

### 4. Running the Pipeline

1. Push the `.gitlab-ci.yml` file and related configuration files to your GitLab repository.
2. When the pipeline runs:
    - It will retrieve the necessary parameters (DigitalOcean API token, GitLab credentials, and repository information) from AWS SSM.
    - Packer will build a snapshot on DigitalOcean.
    - Ansible will provision the droplet by setting up Node.js, PM2, and pulling the latest version of the application repository.
3. After the build completes, a snapshot of the configured DigitalOcean droplet will be available for launching future droplets.

---
