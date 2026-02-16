Step 1: Prepare Jenkins Server and Configure it
        * Install Java and Jenkins
          https://www.jenkins.io/doc/book/installing/linux/#debianubuntu
Step 2: Prepare Ansible Server and Configure it
        * Install Ansible  https://docs.ansible.com/projects/ansible/latest/installation_guide/installation_distros.html#installing-ansible-on-ubuntu 
        * Generate the SSH key as Ubuntu user as ansible_user=ubuntu.         
        * Create SSH Public key and Copy that to Deployment Server's Authorized Keys
        * Check if remote SSH succesful by: ssh ubuntu@<deploymentServer>
Step 2: Prepare Deployment Server and Configure it
Step 3: Create Git Repository and Push the Codebase to Remote Repository
Step 4: Write hosts, ansible.cfg and ansible playbook and Push to remote repository
Step 5: Write Jenkinsfile
Step 6: Go to Jenkins WebUI and install necessary Plug-Ins such as Ansible and SSH Agent, add credentials for   the Jenkins server to access to Ansible server: user ubuntu and private-key and then Create a Pipeline
Step 7: Create a GitHub Webhook to complete the CICD Pipeline. (https://<jenkinsServer>:8080/github-webhook/)

************************************** In Detail **********************************************

Here’s an updated `INSTRUCTION.md` that matches your latest Jenkinsfile, where:

- Jenkins only orchestrates.
- Ansible runs on a separate control node.
- The Ansible server pulls from GitHub and deploys to separate web servers.

***

# CI/CD Pipeline with Jenkins, Ansible Control Node, and Separate Web Servers

## Step 1: Prepare Jenkins Server

- Install Java and Jenkins on the Jenkins server (Ubuntu/Debian).
- Ensure the Jenkins server can reach the **Ansible control node** over SSH (port 22).
- In Jenkins, create an SSH credential for the Ansible control node:
    - Go to **Manage Jenkins → Credentials → System → Global credentials (unrestricted) → Add Credentials**.
    - Kind: **SSH Username with private key**.
    - Username: the user you will use on the Ansible server (for example `ansible` or `ubuntu`).
    - Paste the corresponding private key.
    - Set **ID** to `ansible-server-ssh` (this must match the Jenkinsfile).
- You do **not** need Ansible installed on the Jenkins server in this setup.
- Jenkins will not clone the repo locally; it will trigger Git and Ansible commands on the Ansible control node.

***

## Step 2: Prepare the Ansible Control Node

This is the host where:

- The Git repository is cloned.
- `ansible-playbook` is executed.
- Ansible connects to one or more separate web servers.

On the Ansible control node:

- Provision a Debian/Ubuntu server (for example, an EC2 instance).
- Install Ansible.
- Install Git:

```bash
sudo apt update
sudo apt install -y git
```

- Configure SSH access:
    - Add the **public key** corresponding to the Jenkins credential `ansible-server-ssh` to
`~<ANSIBLE_USER>/.ssh/authorized_keys` (e.g. `~ansible/.ssh/authorized_keys`).
    - Test SSH from Jenkins (or a similar environment) using that key:

```bash
ssh <ANSIBLE_USER>@<ANSIBLE_SERVER_IP>
```

- The Ansible control node must be able to SSH into all web servers it will manage (using the keys specified in your Ansible inventory).

***

## Step 3: Prepare Web Servers (Deployment Targets)

Each web server:

- Runs a Debian/Ubuntu-based OS with `apt`.
- Is reachable from the Ansible control node via SSH (port 22).
- Will have NGINX installed and configured by Ansible.

On each web server:

- Ensure the Ansible control node’s SSH key is added to `~ubuntu/.ssh/authorized_keys` (or the appropriate user).
- Confirm network connectivity from the Ansible control node:

```bash
ssh ubuntu@<web_server_ip>
```


***

## Step 4: Repository Structure on GitHub

Your GitHub repository should look like:

```text
.
├── Jenkinsfile
├── hosts
├── web-playbook.yml
└── web/
    ├── index.html
    ├── styles.css
    └── ...
```

- `web-playbook.yml`: Ansible playbook to install NGINX and deploy the site.
- `hosts`: Ansible inventory defining your web servers.
- `web/`: Static website files to be copied to `/var/www/html/`.

You can fork or create your own repository; just keep these files in place and adjust URLs/branches as needed.

***

## Step 5: Ansible Inventory (`hosts`)

On the Ansible control node, inside the cloned repository, the `hosts` file should define a group that matches your playbook, for example:

```ini
[web-server]
<web_server_1_ip> ansible_user=ubuntu
<web_server_2_ip> ansible_user=ubuntu
```

- Adjust `ansible_user` to the SSH user you use on each web server.
- You can have one or many web servers in the group.

***

## Step 6: Ansible Playbook (`web-playbook.yml`)

This playbook, stored at the repository root, should:

1. Update apt cache.
2. Install NGINX.
3. Ensure NGINX is running and enabled.
4. Replace the default web root with your static content.

Example:

```yaml
- hosts: web-server
  become: true

  tasks:
    - name: update apt
      apt:
        update_cache: yes

    - name: install nginx with apt
      apt:
        name: nginx
        state: present

    - name: ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deleting default landing page
      file:
        path: /var/www/html
        state: absent

    - name: Re-creating the entire directory
      file:
        path: /var/www/html
        state: directory

    - name: Copying web content from the host machine
      copy:
        src: "{{ item }}"
        dest: /var/www/html/
      with_fileglob:
        - "web/*"
```

> Note: This assumes the `web/` directory is present in the same directory where `ansible-playbook` is run (the repo root on the Ansible control node).

***

## Step 7: Jenkins Pipeline (`Jenkinsfile`)

This Jenkinsfile:

- Does **not** clone the repository on the Jenkins server.
- SSHes to the Ansible control node using `ansible-server-ssh`.
- On the Ansible control node:
    - Clones the repo the first time.
    - Runs `git pull` on subsequent runs.
    - Executes the Ansible playbook with the local `hosts` file.

```groovy
pipeline {
    agent any

    environment {
        ANSIBLE_SERVER_IP = 'ANSIBLE_CONTROL_NODE_IP' // e.g. 10.0.1.10
        ANSIBLE_USER      = 'ansible'                 // or ubuntu, etc.
    }

    stages {
        stage('Deploy via Ansible Server') {
            steps {
                sshagent(credentials: ['ansible-server-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no "$ANSIBLE_USER@$ANSIBLE_SERVER_IP" '
                            # Clone or update repo on Ansible server
                            if [ ! -d ~/nginx-website-ec2-ansible-jenkins-CICD ]; then
                                git clone https://github.com/farhaz1449/nginx-website-ec2-ansible-jenkins-CICD.git
                            else
                                cd ~/nginx-website-ec2-ansible-jenkins-CICD && git pull
                            fi

                            # Run Ansible from that repo against remote web servers
                            cd ~/nginx-website-ec2-ansible-jenkins-CICD
                            ansible-playbook -i hosts web-playbook.yml
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline execution successful.'
        }
        failure {
            echo 'Pipeline execution failed.'
        }
    }
}
```

**Replace:**

- `ANSIBLE_CONTROL_NODE_IP` with your actual Ansible control node IP.
- `ansible` (user) with the actual SSH username on that node if different.
- The repository URL if you use your own repo instead of the example.

***

## Step 8: Jenkins Job Configuration

1. In Jenkins, create a new **Pipeline** job:
    - Click **New Item** → choose a name → select **Pipeline**.
2. In the job configuration:
    - Under **Pipeline**, either:
        - Use **Pipeline script from SCM** pointing to your GitHub repo and `Jenkinsfile`, **or**
        - Paste the Jenkinsfile directly under **Pipeline script**.
3. Make sure:
    - The SSH credential ID (`ansible-server-ssh`) exists and is valid.
    - The environment variables in the Jenkinsfile match your setup.

Run **Build Now** to test:

- Jenkins connects to the Ansible control node.
- The Ansible control node clones or updates the repo.
- The playbook runs and configures NGINX on your web servers.

***

## Step 9: GitHub Webhook (Optional, Recommended)

To trigger deployments on every push:

1. In the Jenkins job configuration:
    - Under **Build Triggers**, enable:
        - **GitHub hook trigger for GITScm polling** (or the equivalent for your SCM setup).
2. In your GitHub repository:
    - Go to **Settings → Webhooks → Add webhook**.
    - Payload URL:

```text
http://<jenkins-server-domain-or-ip>:8080/github-webhook/
```

(Use `https` if TLS is enabled.)
    - Content type: `application/json`.
    - Choose **Just the push event**.
    - Click **Add webhook**.

Now each push to the configured branch will trigger Jenkins, which in turn updates the repo on the Ansible control node and runs the playbook.

***

## Step 10: Testing the Deployment

1. After a successful pipeline run, open a browser and visit:

```text
http://<your_web_server_ip>/
```

2. You should see the content from the `web/` directory (for example, `index.html`).
3. To test CI/CD:
    - Modify a file in `web/`.
    - Commit and push to the tracked branch.
    - Watch Jenkins trigger the pipeline.
    - After success, refresh the site and confirm the changes.

***

## Troubleshooting

- **SSH failures Jenkins → Ansible control node:**
    - Verify the `ansible-server-ssh` credential and that its public key is in `authorized_keys` for `ANSIBLE_USER` on the Ansible control node.
    - Confirm you can do:

```bash
ssh <ANSIBLE_USER>@<ANSIBLE_SERVER_IP>
```

- **SSH failures Ansible control node → web servers:**
    - Ensure the Ansible control node’s SSH key is added to each web server’s `authorized_keys`.
    - Confirm hostnames/IPs and `ansible_user` in `hosts`.
- **Git or Ansible not found:**
    - Ensure `git` and `ansible` are installed on the Ansible control node and available in the user’s PATH.
- **NGINX not serving content:**
    - On each web server:
        - Check NGINX status:

```bash
sudo systemctl status nginx
```

        - Check the content:

```bash
ls -R /var/www/html
```


This updated `INSTRUCTION.md` now fully reflects the design where Jenkins orchestrates deployments by SSHing into an Ansible control node, which then pulls from GitHub and runs the playbook against separate web servers.

