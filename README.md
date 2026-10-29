PHP CI/CD Deployment — GitHub Actions & AWS EC2
A practical, end-to-end Continuous Integration / Continuous Deployment (CI/CD) project that automatically updates a Core PHP website running on Amazon Linux EC2 whenever a new commit is pushed to the main branch on GitHub.

The workflow uses GitHub Actions to validate PHP syntax, connects to EC2 over SSH, and runs a secure server-side deploy.sh script. The server fetches the latest source code from GitHub, while Apache serves the application.
Project status: Repository source and CI/CD configuration are provided. A successful live AWS deployment has not yet been verified. Configure the EC2 instance, SSH trust, GitHub secrets, and runner network access before expecting the deployment job to pass.

Coursework note: This is a GitHub Actions adaptation of a lab that originally specifies GitLab CI/CD. If your assessment requires GitLab, this GitHub implementation alone does not fulfill that requirement.

Overview
Item	Details
Application	Core PHP demo website
Repository	
Source branch	main
CI/CD platform	GitHub Actions
Hosting	AWS EC2 (Amazon Linux 2023)
Web server	Apache (httpd) + PHP
Deployment	GitHub Actions → SSH → server-side deploy.sh → git fetch / git reset
Application directory	/var/www/myapp
Deployment user	deploy


Features
- Automated deployment on pushes to main and optional manual runs (workflow_dispatch).
- PHP syntax validation before the remote deployment step.
- Separate SSH key pairs for GitHub Actions → EC2 and EC2 → GitHub.
- Read-only repository access from the EC2 instance using a GitHub deploy key.
- Server-side deployment script with Git repository URL checks and a lock to prevent overlapping deployments.
- Strict SSH host-key checking; no need to disable verification.
- Deployment traceability through Git commit hashes and GitHub Actions logs.
- Rollback guidance and troubleshooting documentation.
Architecture
Developer
    |
    | git push origin main
    v
GitHub Repository
    |
    v
GitHub Actions (ubuntu-24.04)
    |-- Checkout source
    |-- Install PHP CLI + SSH client
    |-- php -l index.php
    |
    | SSH: deployment key (Actions -> EC2)
    v
AWS EC2: deploy user
    |
    | /usr/local/bin/deploy.sh
    | git fetch origin main
    | git reset --hard origin/main
    | SSH: read-only deploy key (EC2 -> GitHub)
    v
/var/www/myapp
    |
    v
Apache + PHP -> Website visitors
Important: The two SSH connections use different keys. GitHub Actions authenticates to EC2 to run the script; EC2 authenticates to GitHub to fetch the repository.
Project Structure
php-aws-cicd-demo/
├── .github/
│   └── workflows/
│       └── deploy.yml           # GitHub Actions CI/CD workflow
├── docs/
│   ├── SETUP_GUIDE.md           # Detailed EC2/GitHub setup instructions
│   ├── ROLLBACK_PLAN.md         # Recovery steps and rollback limitations
│   └── EVIDENCE_CHECKLIST.md    # Suggested lab demonstration evidence
├── .gitignore                  # Files that must not be committed
├── deploy.sh                   # Server-side deployment script
├── index.php                   # Example PHP application
└── README.md                   # Project documentation
Upload correctly: All of these files must be at the repository root. Do not keep them inside a Github files/ directory. In particular, .github/workflows/deploy.yml must be at that exact path for GitHub Actions to discover the workflow.

Prerequisites
- A GitHub account and access to this repository.
- An AWS account with permission to manage an EC2 instance and its security group.
- An Amazon Linux 2023 EC2 instance, an administrative .pem key, and an accessible public IP or DNS name.
- Git and an SSH client on your computer; a terminal or VS Code is sufficient.
- Knowledge of basic Linux commands. AWS usage may incur charges.
Setup Guide
1. Clone the repository locally
git clone https://github.com/dinushamalshan37-oss/php-aws-cicd-demo.git
cd php-aws-cicd-demo
If you already created the GitHub repository, copy the extracted project contents into this local repository root, then commit and push the files:
git add .
git commit -m "Add PHP CI/CD application and workflow"
git push origin main
Avoid committing any .pem, SSH private key, .env, or AWS credential file.
2. Launch and secure AWS EC2
In the AWS EC2 console, launch an Amazon Linux 2023 instance and download its administrator key pair securely.
- Permit HTTP (TCP 80) for the demo website.
- Permit SSH (TCP 22) from your own IP for administration.
- Do not permanently open port 22 to 0.0.0.0/0 just to make CI/CD work.
Connect from your computer (replace the placeholders):
ssh -i /path/to/aws-admin-key.pem ec2-user@YOUR_EC2_PUBLIC_IP
Install the required packages on EC2:
sudo dnf install -y git openssh-clients httpd php php-fpm
sudo systemctl enable --now httpd
sudo systemctl enable --now php-fpm
Create a limited deployment account and prepare the application directory:
sudo useradd -m -s /bin/bash deploy
sudo mkdir -p /var/www/myapp
sudo chown -R deploy:apache /var/www/myapp
sudo chmod 750 /var/www/myapp
Do not rerun useradd if the user already exists.
3. Configure Apache
Create /etc/httpd/conf.d/myapp.conf on the instance:
<VirtualHost *:80>
    DocumentRoot /var/www/myapp
    <Directory /var/www/myapp>
        Require all granted
        AllowOverride None
    </Directory>
</VirtualHost>
Validate and restart Apache:
sudo apachectl configtest
sudo systemctl restart httpd
sudo systemctl status httpd
4. Key A — EC2 to GitHub (read-only)
In the EC2 terminal, create a dedicated SSH identity as the deploy user:
sudo -iu deploy
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -C "ec2-github-readonly" -f ~/.ssh/github_repo
For this unattended lab setup, a passwordless repository key may be used; keep its private file restricted to the deploy user.

Configure ~/.ssh/config:
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_repo
    IdentitiesOnly yes
Then:
chmod 600 ~/.ssh/config
cat ~/.ssh/github_repo.pub
Copy only the public key shown by the final command into GitHub → Repository Settings → Deploy keys → Add deploy key. Leave Allow write access unchecked.

Verify the SSH host fingerprint against GitHub's official SSH fingerprints before adding github.com to ~/.ssh/known_hosts. Do not blindly trust unverified host keys.

Test:
ssh -T git@github.com
git ls-remote git@github.com:dinushamalshan37-oss/php-aws-cicd-demo.git
exit
A GitHub greeting on ssh -T can be a successful authentication even if the command returns exit code 1.
5. Clone the application on EC2
Back in your EC2 administrator account:
sudo -u deploy git clone --branch main \
  git@github.com:dinushamalshan37-oss/php-aws-cicd-demo.git \
  /var/www/myapp
Check the repository:
sudo -u deploy git -C /var/www/myapp remote -v
sudo -u deploy git -C /var/www/myapp log -1 --oneline
The destination directory must be empty before cloning.
6. Install the deployment script on EC2
Open deploy.sh locally and set its REPO_URL variable to the exact SSH remote:
REPO_URL="git@github.com:dinushamalshan37-oss/php-aws-cicd-demo.git"
Copy the edited file from your local computer to EC2:
scp -i /path/to/aws-admin-key.pem deploy.sh \
  ec2-user@YOUR_EC2_PUBLIC_IP:/tmp/myapp-deploy.sh
In the EC2 terminal, securely install and manually test it:
sudo install -o root -g deploy -m 750 \
  /tmp/myapp-deploy.sh /usr/local/bin/deploy.sh
sudo -u deploy /usr/local/bin/deploy.sh
Expected logs include a deployed commit SHA and Deployment completed successfully. Check the demo site at http://YOUR_EC2_PUBLIC_IP.
The script uses git reset --hard origin/main: uncommitted changes to tracked files on the server will be lost. Keep secrets and uploads outside tracked application files.

7. Key B — GitHub Actions to EC2
On your own computer, create a second and different SSH key pair:
ssh-keygen -t ed25519 -C "github-actions-to-ec2" -f github_actions_deploy
Append the contents of github_actions_deploy.pub to /home/deploy/.ssh/authorized_keys on EC2. On EC2, first secure that file:
sudo install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
sudo touch /home/deploy/.ssh/authorized_keys
sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
sudo nano /home/deploy/.ssh/authorized_keys
Paste the public key as a new line and save. Test from your own computer:
ssh -i github_actions_deploy deploy@YOUR_EC2_PUBLIC_IP
Never commit or share the private key.
8. Set GitHub Actions secrets and variables
Open this repository's Settings → Secrets and variables → Actions and configure:
Name	Type	Value
DEPLOY_SSH_PRIVATE_KEY	Repository secret	Full contents of the private github_actions_deploy key
EC2_SSH_KNOWN_HOSTS	Repository secret	Verified EC2 SSH host-key entry for the same hostname/IP as DEPLOY_HOST
DEPLOY_HOST	Repository variable	EC2 public IP or DNS name
DEPLOY_USER	Repository variable	deploy


Get the EC2 SSH host public key from an already trusted administrative session and verify its fingerprint before constructing the known_hosts entry. Do not disable StrictHostKeyChecking.

The workflow references a GitHub Actions environment named production. Configure environment protection rules as appropriate.

Runner networking: Standard GitHub-hosted runners use changing public IP addresses. A secure EC2 SSH setup therefore needs a network path that does not require unrestricted internet SSH access, such as a carefully managed self-hosted runner or a suitable runner with controlled egress and narrowly scoped security-group rules. Without a valid path from the runner to EC2, the remote deploy step can time out.
Running the Pipeline
The workflow lives in [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) and runs when code is pushed to main. A manual trigger is also available in Actions.

To test the automatic trigger, edit a visible heading in index.php, then run:
git add index.php
git commit -m "Test automatic deployment"
git push origin main
Open GitHub Actions, choose Deploy PHP to EC2, and inspect the latest run.

Expected sequence: Checkout PHP source → Install PHP and SSH client → Validate PHP syntax → Run server-side deploy.sh over SSH.

Verify the deployed commit and Apache service from EC2:
sudo -u deploy git -C /var/www/myapp log -1 --oneline
sudo systemctl status httpd
The commit SHA on EC2 should match the commit deployed by GitHub Actions. Refresh http://YOUR_EC2_PUBLIC_IP to check the updated page.
Deployment and Security Notes
1. Use two separate SSH identities with least-privilege access.
2. Keep .env, AWS keys, private SSH keys, logs, and user uploads out of Git.
3. Verify GitHub and EC2 SSH host keys; keep strict host-key checking enabled.
4. Scope SSH access to trusted networks; avoid open port 22 to the internet.
5. Treat git reset --hard as destructive to uncommitted tracked server changes.
6. Back up application data and any database before significant releases.
7. Add automated tests, health checks, and a reviewed rollback procedure before real production use.
8. Shut down or delete unused AWS resources when the lab is finished to avoid unexpected charges.
Troubleshooting
Error / symptom	What to check
Workflow not appearing	Confirm .github/workflows/deploy.yml is at the repository root, committed on main
Missing repository secrets	Ensure the exact secret/variable names above are configured
Permission denied (publickey) (Actions → EC2)	Check DEPLOY_SSH_PRIVATE_KEY, EC2 authorized_keys, deploy username, SSH permissions
Permission denied (publickey) (git fetch from EC2)	Check the separate GitHub repository deploy key and read-only access
Host key verification failed	Verify and correct the corresponding known_hosts entry
SSH connection timeout	Check runner-to-EC2 routing, security-group rules, IP restrictions, firewall
Repository URL does not match	Set REPO_URL in /usr/local/bin/deploy.sh to the exact value of git remote get-url origin
not a git repository	Ensure the project was cloned into /var/www/myapp
Apache 403 / blank page	Check DocumentRoot, Apache/PHP status, directory permissions, SELinux where applicable
Script permission denied	Check /usr/local/bin/deploy.sh ownership root:deploy and mode 750


Rollback Strategy
Before deployment, record the current working commit and maintain backups. To inspect history on EC2:
sudo -u deploy git -C /var/www/myapp log -5 --oneline
For a temporary source-code-only rollback, after selecting a verified older commit:
sudo -u deploy git -C /var/www/myapp reset --hard PREVIOUS_COMMIT_SHA
A later successful deployment of main can overwrite this temporary rollback. For a durable rollback, revert the problem commit in GitHub and redeploy. Neither method automatically reverses database migrations nor restores uploaded user files. See [`docs/ROLLBACK_PLAN.md`](docs/ROLLBACK_PLAN.md).
Documentation
- [Detailed setup guide](docs/SETUP_GUIDE.md)
- [Rollback plan](docs/ROLLBACK_PLAN.md)
- [Lab evidence checklist](docs/EVIDENCE_CHECKLIST.md)
- GitHub Actions documentation
- GitHub deploy keys documentation
- AWS EC2 SSH connection guide
Author
Dinusha Malshan
Software Engineering Student — NIBM, Sri Lanka
GitHub: @dinushamalshan37-oss
Built for learning CI/CD, Linux administration, cloud deployment, SSH authentication, and DevOps fundamentals.
