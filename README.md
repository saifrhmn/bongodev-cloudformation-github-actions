# bongoDev Homework: Automate AWS Infrastructure as Code with GitHub Actions

> **Purpose of this lab - read this first**
>
> In the previous CloudFormation lab, you uploaded a YAML file manually to create a new **Ubuntu 24.04 LTS EC2 instance running Nginx**. The goal of this homework is to **remove the manual CloudFormation deployment step**.
>
> You will put your infrastructure code in a GitHub repository, configure a **GitHub Actions CI/CD workflow**, and securely connect GitHub to AWS using **OpenID Connect (OIDC)**. From then on, when you push a relevant change to the `main` branch, GitHub Actions will **validate the CloudFormation template and automatically create or update your AWS stack**.
>
> This exercise teaches a real DevOps practice: **Git as the source of truth for infrastructure, automated deployments, repeatable environments, and short-lived cloud credentials**. GitHub Actions runs the pipeline; CloudFormation provisions the resources; Ubuntu installs and runs Nginx.

**Assignment type:** Individual student homework  
**Estimated duration:** 45–75 minutes (first-time IAM/OIDC setup can take longer)  
**Level:** Beginner–intermediate DevOps  
**AWS environment:** Your own *sandbox/learning* account (not an employer production account)

> ⚠️ **COST AND SECURITY WARNING:** This homework creates real AWS billable resources (EC2 instance, EBS storage, public IPv4 address, etc.). Follow **Step 11 - Cleanup** after taking screenshots. The Nginx demo uses **HTTP port 80** so it is **not production-ready**. Do not store AWS access keys in GitHub or commit credentials to the repository.

---

## 1. What you will achieve

By the end of this homework, you will be able to:

1. Explain why **GitHub Actions + CloudFormation** is useful for IaC.
2. Create a GitHub repository and version-control your CloudFormation YAML.
3. Establish secure **GitHub → AWS** access with IAM + OIDC, not permanent access keys.
4. Configure a GitHub Actions workflow that automatically runs on a push to `main`.
5. Provision an Ubuntu EC2 instance, VPC, subnet, routing, security group and Nginx site **without using Launch Instance in the EC2 console**.
6. Make a safe infrastructure change, push it, and observe an **update to the same stack**.
7. Provide proof of your automation and clean up the AWS resources.

### Architecture

```text
Your computer / VS Code
        |
        | git add -> git commit -> git push origin main
        v
GitHub repository (cloudformation.yaml + .github/workflows/deploy.yml)
        |
        | push triggers workflow
        v
GitHub Actions runner (ubuntu-latest)
        |  checkout repository
        |  assume IAM role through OIDC (short-lived credentials)
        |  aws cloudformation validate-template
        |  aws cloudformation deploy
        v
AWS CloudFormation stack: bongodev-github-iac-dev
        |
        +--> VPC + public subnet + IGW + route table
        +--> Security group (HTTP/80 only; no inbound SSH)
        +--> Ubuntu 24.04 EC2 (t3.micro)
                   |
                   | EC2 User Data: apt-get install nginx
                   v
             Nginx web page: Hello from bongoDev!
```

**Important:** Pushing again does **not** necessarily create another instance. CloudFormation will create a stack if it does not exist and update the existing stack if it does. A harmless tag update usually preserves the existing instance. Some other changes can cause replacement.

---

## 2. Files included in this ZIP

```text
task-IAC-with-CloudFormation/
├── README.md                            <-- these step-by-step instructions
├── cloudformation.yaml                  <-- Ubuntu + Nginx + networking IaC
├── .github/
│   └── workflows/
│       └── deploy.yml                   <-- automation workflow
├── iam/
│   ├── deployment-policy-lab.json       <-- classroom deployment permissions
│   ├── trust-policy-new-repos-2026.json <-- use for new GitHub repositories
│   └── trust-policy-legacy-repos.json   <-- fallback for older repositories
├── docs/
│   └── student-submission-checklist.md
└── .gitignore
```

**Do not delete `.github/workflows/deploy.yml`.** The `.github` directory is hidden by some file managers. On macOS Finder, use **Command + Shift + .** to reveal hidden files. In a terminal, run `ls -la`.

### What `cloudformation.yaml` creates

| AWS resource | Purpose |
|---|---|
| `DemoVPC` | VPC `10.42.0.0/16` |
| `PublicSubnet` | Public subnet `10.42.1.0/24` |
| Internet gateway + route table | Internet access for installing and serving Nginx |
| `WebSecurityGroup` | Public inbound TCP **80** for this training site |
| `UbuntuNginxServer` | Ubuntu 24.04 LTS EC2 of type `t3.micro` |
| EC2 `UserData` | Automatically runs Ubuntu `apt-get` and starts Nginx |
| Stack outputs | `WebsiteURL`, `InstanceId`, `PublicIPAddress`, `VpcId` |

The template uses Canonical's AWS public SSM parameter for a current Ubuntu 24.04 AMI (no region-specific AMI ID to copy). Versions can differ across deployments at different times; for a strictly reproducible production build you would pin the AMI and package versions.

---

## 3. Prerequisites and preparation

You need:

- A **GitHub account** (https://github.com/).
- An **AWS account** with permission to create IAM identity providers, IAM roles and policies, CloudFormation stacks, EC2 and VPC resources. Ask your teacher/administrator if your account is restricted.
- **Git** installed (`git --version`) and an editor such as VS Code.
- A terminal (macOS Terminal, Linux shell or Windows Git Bash).
- A GitHub authentication method for `git push` (e.g. GitHub CLI login or Git Credential Manager). **GitHub account passwords are not accepted for HTTPS git pushes.**
- A learning/sandbox AWS account with a suitable EC2 service quota. Confirm you understand costs.

For this assignment use:

| Setting | Value |
|---|---|
| GitHub repository name | `bongodev-cloudformation-github-actions` |
| Git branch | `main` |
| AWS region | `us-west-2` (N. Virginia; choose another supported region if needed) |
| IAM role name | `bongodev-github-iac-deploy-role` |
| IAM policy name | `bongodev-github-iac-lab-policy` |
| AWS CloudFormation stack | `bongodev-github-iac-dev` |
| CloudFormation environment parameter | `dev` |

You will replace placeholders such as `<AWS_ACCOUNT_ID>`, `<GITHUB_OWNER>`, `<OWNER_ID>` and `<REPO_ID>` with *your own* values. **Do not include literal `<` or `>` in the final AWS trust policy.**

**Why set up IAM before the first push?** Your first push containing `deploy.yml` can automatically start a deployment. Setting up AWS access and GitHub variables first prevents avoidable failures.

---

## 4. Step 1 - Create an EMPTY GitHub repository

1. Sign in at https://github.com/new.
2. Set **Repository name** to `bongodev-cloudformation-github-actions`.
3. Choose **Public** for the easiest homework setup (the project contains no keys or secrets). You can use a private repository if you know how to authenticate GitHub CLI/API queries.
4. **Do not** select *Add a README file*, *Add .gitignore* or *Choose a license*. Create an **empty** repository.
5. Click **Create repository**.
6. Record your repository's **owner name**, **repository name**, and full URL:

   `https://github.com/<GITHUB_OWNER>/bongodev-cloudformation-github-actions`

> A repository created today uses GitHub's newer **immutable OIDC subject** format. Do not blindly copy older tutorials using only `repo:OWNER/REPO:ref:refs/heads/main`.

---

## 5. Step 2 - Find your GitHub owner ID and repository ID (important in 2026)

**Why?** GitHub repositories created on or after **July 15, 2026** normally put the *numeric owner and repository IDs* into the OIDC subject claim. AWS IAM must trust the **exact** subject your workflow will present.

For a **public repository**, open this URL in your browser (replace the names):

```text
https://api.github.com/repos/<GITHUB_OWNER>/bongodev-cloudformation-github-actions
```

You'll see JSON. Find:

- Top-level `"id"` → this is **`REPO_ID`**.
- Inside the `"owner"` object, find its `"id"` → this is **`OWNER_ID`**.
- `"owner"."login"` → GitHub account or organization name (`GITHUB_OWNER`).
- Top-level `"name"` → repository name (`REPO_NAME`).

If you already use GitHub CLI, you can get the exact identity prefix for any accessible repository (including private repos):

```bash
gh api repos/<GITHUB_OWNER>/bongodev-cloudformation-github-actions \
  --jq '"\(.owner.login)@\(.owner.id)/\(.name)@\(.id)"'
```

For example, if your GitHub owner is `student123`, owner ID `112233` and repository ID `778899`, the complete subject is:

```text
repo:student123@112233/bongodev-cloudformation-github-actions@778899:ref:refs/heads/main
```

**Your actual numeric values will be different.** Make sure this matches `main` and not a different branch. The example IDs are fictitious.

**Older GitHub repositories:** A repository created before July 15, 2026 *might* use the previous subject format without IDs. For those only, use `iam/trust-policy-legacy-repos.json`. New repositories should use `iam/trust-policy-new-repos-2026.json`.

---

## 6. Step 3 - Add the GitHub OIDC identity provider in AWS IAM

1. Sign in to https://console.aws.amazon.com/iam/.
2. In the left navigation, choose **Identity providers**.
3. Select **Add provider**.
4. For **Provider type**, select **OpenID Connect**.
5. For **Provider URL**, enter exactly:

   `https://token.actions.githubusercontent.com`

6. For **Audience**, enter exactly:

   `sts.amazonaws.com`

7. Add/create the provider. If an identical GitHub provider **already exists** in your account, use it; do not create a duplicate.

**Explain it:** AWS will accept a GitHub-issued OIDC token that meets the IAM role's trust policy and issue **temporary AWS credentials**. We do **not** store AWS access keys in GitHub Secrets or files.

---

## 7. Step 4 - Create permissions policy and IAM deployment role

### 4A. Create a customer-managed IAM policy

1. In **IAM → Policies**, select **Create policy**.
2. Choose the **JSON** editor.
3. Open `iam/deployment-policy-lab.json` in VS Code and copy **the entire JSON** into the IAM policy editor.
4. Review the JSON, continue, and name the policy `bongodev-github-iac-lab-policy`.
5. Select **Create policy**.

**Security explanation:** The permissions cover the CloudFormation operations, public Ubuntu AMI parameter lookup, and EC2/VPC resources needed in this lab. The EC2 create/delete actions use `Resource: "*"` for classroom simplicity, so this is **not a tightly scoped production IAM policy**. Use it only in an isolated sandbox AWS account, and delete it after the homework. For production, restrict actions to allowed resources/tags, use CloudFormation service roles, approvals, protected branches, and organization security policies.

### 4B. Create a role GitHub Actions can assume

1. In **IAM → Roles**, choose **Create role**.
2. Select **Web identity** (the wizard may label it **Federated identity** depending on the console layout).
3. Select the existing GitHub OIDC identity provider: `token.actions.githubusercontent.com`.
4. Select audience `sts.amazonaws.com`.
5. If AWS asks for GitHub organization/repository/branch fields, enter your GitHub owner, repository, and `main`. **We will explicitly review and replace the trust policy afterward.**
6. Continue to the permissions screen, attach the policy `bongodev-github-iac-lab-policy`.
7. Set role name `bongodev-github-iac-deploy-role`; create it.
8. Open the new role, select **Trust relationships**, and choose **Edit trust policy**.
9. Open `iam/trust-policy-new-repos-2026.json`. Replace these exact placeholders:

   - `<AWS_ACCOUNT_ID>` → your **12-digit AWS account ID** (visible in the AWS console account menu).
   - `<GITHUB_OWNER>` → your GitHub username/organization name.
   - `<OWNER_ID>` → numeric ID from Step 2.
   - `<REPO_NAME>` → `bongodev-cloudformation-github-actions`.
   - `<REPO_ID>` → numeric ID from Step 2.

10. Paste the completed JSON into **Edit trust policy**, verify the values, and **Save changes**.

The critical trust condition will look **similar** to the following (fake example IDs):

```json
"Condition": {
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:student123@112233/bongodev-cloudformation-github-actions@778899:ref:refs/heads/main"
  }
}
```

The policy must also contain a `Federated` principal pointing to **your** AWS account's GitHub OIDC provider. Do not paste only the snippet above; use the complete JSON file.

**If your repo uses the older non-ID subject:** Use `iam/trust-policy-legacy-repos.json` instead, replacing account, owner and repo values. **Do not open trust to all branches or all GitHub repositories with a wildcard.**

### 4C. Copy the role ARN

From the IAM role's **Summary**, copy the **Role ARN**, for example:

```text
arn:aws:iam::123456789012:role/bongodev-github-iac-deploy-role
```

This is a role identifier, **not an AWS access key**. Save it for the next step. Your AWS account number and role ARN should still be handled carefully when sharing publicly.

---

## 8. Step 5 - Configure repository variables in GitHub

1. Open your GitHub repository.
2. Select **Settings → Secrets and variables → Actions**.
3. Choose the **Variables** tab (not Secrets).
4. Create these **repository variables**:

| Name | Value |
|---|---|
| `AWS_ROLE_ARN` | Your IAM role ARN from Step 4C |
| `AWS_REGION` | `us-west-2` (or the region you chose) |

5. Double-check the spelling. `AWS_ROLE_ARN` and `AWS_REGION` must match the workflow's `${{ vars.NAME }}` references.

**Why variables and not secrets?** These are configuration identifiers, not permanent credentials. OIDC provides short-lived credentials automatically at execution time. **Do not create `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` in this lab.**

---

## 9. Step 6 - Push the downloaded project to GitHub

### 6A. Extract and open the project

Extract `task-IAC-with-CloudFormation.zip`. Open the **inner folder containing `README.md` and `cloudformation.yaml`** in VS Code and open a terminal in that folder. Make sure `.github/workflows/deploy.yml` is present.

### 6B. Initialize Git and commit the files

Run these commands from the project root:

```bash
git init
git branch -M main
git add .
git commit -m "Add IaC and GitHub Actions automation"
```

If Git asks for a name and email, set them using `git config --global user.name "Your Name"` and `git config --global user.email "your-github-email"`, then retry the commit.

### 6C. Connect Git to your GitHub repository

Change `<GITHUB_OWNER>` to your username or organization:

```bash
git remote add origin https://github.com/<GITHUB_OWNER>/bongodev-cloudformation-github-actions.git
git push -u origin main
```

Your git client might open a browser to authenticate. Sign in to GitHub using the supported authentication prompt/Git Credential Manager or `gh auth login` first. Do **not** put your GitHub password in the terminal.

**Checkpoint:** In GitHub's **Code** tab, you should see `README.md`, `cloudformation.yaml`, `iam/`, `docs/`, and `.github/workflows/deploy.yml` (open the `.github` directory if necessary).

**Why does this trigger a deploy?** The workflow contains:

```yaml
on:
  push:
    branches: [main]
    paths:
      - cloudformation.yaml
      - .github/workflows/deploy.yml
  workflow_dispatch:
```

A push to `main` that adds/changes either of these files launches the pipeline. `workflow_dispatch` also permits a manual run. Changes only to the README usually will not trigger an automated deploy because of the `paths` filter.

---

## 10. Step 7 - Watch GitHub Actions automatically deploy Ubuntu + Nginx

1. In the GitHub repository, open **Actions**.
2. Find the workflow **Deploy Ubuntu Nginx with CloudFormation**.
3. Select the latest workflow run (triggered by your initial push).
4. Open the job **Validate and deploy infrastructure**.
5. Watch these steps:

   - **Checkout repository:** downloads your YAML into the runner.
   - **Configure temporary AWS credentials (OIDC):** assumes the trusted AWS role.
   - **Confirm AWS identity:** displays the temporary caller identity.
   - **Validate CloudFormation template:** checks template validity using AWS.
   - **Create or update CloudFormation stack:** provisions the infrastructure.
   - **Print website and infrastructure outputs:** lists stack outputs, including the website URL.

6. Wait for the workflow to complete with a green **success** status. Actual duration varies with AWS provisioning.

**Very important:** You should **not** manually click *Create stack* in AWS. This is the core assignment: **GitHub Actions does the deployment for you.**

### What the workflow actually runs

Open `.github/workflows/deploy.yml` and locate this command:

```bash
aws cloudformation deploy \
  --template-file cloudformation.yaml \
  --stack-name "$STACK_NAME" \
  --parameter-overrides Environment=dev \
  --region "$AWS_REGION" \
  --no-fail-on-empty-changeset
```

`deploy` creates a new stack on the first deployment. On a later push, it updates that stack when template changes exist. `--no-fail-on-empty-changeset` treats a no-op deployment as success.

### Step 8 - Verify resources in AWS

1. Go to https://console.aws.amazon.com/cloudformation/.
2. Select the **same AWS region** that you configured in GitHub (`AWS_REGION`).
3. Open **Stacks** and find `bongodev-github-iac-dev`.
4. Confirm the stack shows **`CREATE_COMPLETE`** after the first successful deployment.
5. Open **Resources** and locate the VPC, subnet, security group and EC2 instance.
6. Open the **Outputs** tab and copy **`WebsiteURL`**.
7. Paste the site URL in a browser, for example `http://YOUR-PUBLIC-IP`.
8. Confirm the following content appears:

```text
Hello from bongoDev!
Infrastructure as Code is working!
Ubuntu 24.04 LTS + Nginx + AWS CloudFormation
Environment: dev
```

9. In **EC2 → Instances**, confirm a running Ubuntu `t3.micro` and record its instance ID.

**No SSH or EC2 key pair is needed.** EC2 **User Data** installs Nginx via `apt-get` on the new Ubuntu instance. CloudFormation `CREATE_COMPLETE` can happen before User Data finishes; if the site is not immediately available, investigate and retry.

### Step 9 - Demonstrate that a SECOND push triggers an UPDATE

Your task is to prove that **a Git change updates the same infrastructure**.

1. In `cloudformation.yaml`, find the EC2 instance's tag:

   ```yaml
   - Key: Course
     Value: bongoDev-DevOps
   ```

2. Change **only the tag value** to:

   ```yaml
   - Key: Course
     Value: bongoDev-DevOps-Homework-V2
   ```

3. Save the file, then run:

   ```bash
   git add cloudformation.yaml
   git commit -m "Update EC2 Course tag through IaC"
   git push origin main
   ```

4. Open **GitHub → Actions** and watch the *new* workflow run.
5. In AWS CloudFormation, observe the stack updating, then reaching **`UPDATE_COMPLETE`**.
6. In **EC2 → Instances → your instance → Tags**, confirm:

   `Course = bongoDev-DevOps-Homework-V2`

7. Compare the before/after **Instance ID**. A tag-only update normally keeps the same EC2 instance.

**Critical distinction:** Do not use editing the HTML inside EC2 User Data as the main update test. By default, the Ubuntu bootstrap script executes on the first launch only. Changing a User Data script in a template does **not** automatically rerun the website installation on an existing Ubuntu instance. For application changes in real projects, use a deployment mechanism such as an application pipeline, SSM automation, or instance replacement.

### Optional extra experiment: Manually rerun the pipeline

In **GitHub → Actions → Deploy Ubuntu Nginx with CloudFormation**, select **Run workflow** (on the `main` branch), then observe that the deployment can finish successfully even with **no template changes**.

---

## 11. Step 10 - Submit your evidence BEFORE deleting resources

Complete the [student submission checklist](docs/student-submission-checklist.md). Submit:

1. Your GitHub repository URL and the commit URL for the **second push**.
2. Screenshot of **successful GitHub Actions** runs (initial creation and second update).
3. Screenshot of the **CloudFormation stack** (`CREATE_COMPLETE` / `UPDATE_COMPLETE`).
4. Screenshot of the **Ubuntu EC2 instance and updated `Course` tag**.
5. Screenshot of the live **Nginx website** showing `Hello from bongoDev!`.
6. A short paragraph explaining **GitHub Actions vs CloudFormation** and why **OIDC** is preferred over long-lived AWS access keys.
7. Confirmation that the stack was **deleted** after the evidence was captured.

Do not submit access keys, credentials, private key files, or unnecessary personal/account details.

---

## 12. Step 11 - MANDATORY cleanup (avoid AWS charges)

**Do not leave the instance running after completing the lab.**

1. Open **AWS CloudFormation → Stacks** in the same region used by the workflow.
2. Select `bongodev-github-iac-dev`.
3. Choose **Delete**, then confirm the deletion.
4. Refresh **Events** and wait until the stack is removed or shows `DELETE_COMPLETE` in deleted stack history.
5. Open **EC2 → Instances** and verify the lab EC2 instance is terminated. Confirm that the VPC and network resources created by the stack are no longer present. Investigate any `DELETE_FAILED` errors.
6. In GitHub, **do not push the deployment YAML again** after cleanup, or the workflow may recreate the stack. If you still need to edit the repository, **disable the workflow** in the GitHub **Actions** tab first.
7. After the course, delete the dedicated IAM role and classroom permissions policy when no longer needed. Do **not** delete a GitHub OIDC provider that another project in the account uses.

You can also delete the stack with AWS CLI using *your own authorized local AWS credentials*:

```bash
aws cloudformation delete-stack \
  --stack-name bongodev-github-iac-dev \
  --region us-west-2

aws cloudformation wait stack-delete-complete \
  --stack-name bongodev-github-iac-dev \
  --region us-west-2
```

**Note:** The `GitHub Actions` role in this lab is intended for deployment; it does **not** need stack deletion permission. Use your console user's permissions for cleanup.

---

## 13. Troubleshooting (read the relevant symptom)

### Error: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

- First check **IAM → Role → Trust relationships**.
- Verify `aud` is `sts.amazonaws.com`.
- Verify AWS account ID, GitHub username, repo name, **owner ID**, **repo ID**, and `main` branch in `sub`.
- **GitHub repositories created after July 15, 2026 use an immutable OIDC subject** with both numeric IDs (unless configured otherwise). Using the old name-only subject causes a mismatch.
- Verify the IAM OIDC provider exists and `AWS_ROLE_ARN` points to the correct IAM role.
- If you changed your repository name after setting up the trust policy, update the policy accordingly.

### Error: `AccessDenied` for `ec2:*`, `cloudformation:*`, or `ssm:*`

- Confirm `bongodev-github-iac-lab-policy` is attached to the **role** used by GitHub Actions.
- Review the exact denied action and resource in the Actions logs / CloudFormation **Events** tab.
- Check for organization Service Control Policies (SCPs), permission boundaries, region restrictions, or quotas: attaching a role policy cannot override them.
- Do not solve this by granting `AdministratorAccess` to GitHub Actions.

### Error: `AWS_ROLE_ARN` or AWS region missing / invalid

- In GitHub **Settings → Secrets and variables → Actions → Variables**, make sure repository variables exist with *exact* names: `AWS_ROLE_ARN` and `AWS_REGION`.
- Use a valid IAM role ARN, e.g. `arn:aws:iam::123456789012:role/bongodev-github-iac-deploy-role`.
- Make sure the region supports the chosen EC2 instance type (`t3.micro`).

### Workflow never starts

- Verify file path is exactly **`.github/workflows/deploy.yml`**, inside the repo's `main` branch.
- Check the repository **Actions** setting allows workflows and the workflow is enabled.
- The default workflow responds only to `main` pushes changing the template/workflow. A README-only change **does not** trigger it.
- Use **Actions → workflow → Run workflow** to perform a manual run after the file is on `main`.

### Error: CloudFormation stack reaches `ROLLBACK_COMPLETE`

- In AWS CloudFormation, inspect **Events** for the *first failing resource and error* (e.g. instance capacity, blocked AMI, insufficient EC2 permissions, restricted region, instance limits).
- Fix the underlying issue, then delete the failed `ROLLBACK_COMPLETE` stack using your authorized console identity, and rerun the workflow.

### Website is blank / `ERR_CONNECTION_TIMED_OUT` / site not yet available

- Open the **HTTP** URL from stack Outputs, not HTTPS.
- Ensure the stack's EC2 instance is `Running`, passed EC2 status checks, and has a public IPv4 address.
- Confirm security group inbound **TCP port 80** from `0.0.0.0/0` for this lab.
- Confirm public subnet routing to the Internet Gateway.
- EC2 User Data needs time to run `apt-get` and install Nginx. `CREATE_COMPLETE` is not a guarantee of Nginx readiness.
- If permitted to use AWS Systems Manager or another authorized diagnostic channel, check `/var/log/cloud-init-output.log` or `/var/log/bongodev-bootstrap.log`. No SSH ingress is configured in this lab.

### Error: `No updates are to be performed`

- The provided workflow uses `--no-fail-on-empty-changeset` and normally treats no changes as success.
- To demonstrate an actual update, modify the **EC2 Course tag value** and push the YAML again.

---

## 14. Questions to answer in your homework

1. What is the difference between **GitHub Actions** and **AWS CloudFormation**?
2. Why can a `git push` deploy infrastructure without anyone logging into the AWS console?
3. What does **OIDC** do, and why is it safer than committing AWS access keys?
4. On a second push, why is the existing stack **updated** instead of creating a new one?
5. Is **CloudFormation stack creation complete** always the same as **Nginx ready**? Why not?
6. Why does this homework update the **EC2 tag** instead of changing the webpage HTML?
7. What should happen to AWS resources when you have finished the assignment?

**Definition to remember:** *IaC = describe the desired infrastructure as code; GitHub Actions = run the deployment automation; CloudFormation = reconcile AWS resources to that template.*

---

## 15. Official references (recommended reading)

- GitHub Docs - [Configuring OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- GitHub Docs - [Immutable subject claims for new repositories](https://docs.github.com/en/actions/reference/security/oidc#immutable-subject-claims)
- AWS-maintained action - [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
- AWS - [CloudFormation deploy CLI reference](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy/index.html)
- Ubuntu / Canonical - [CloudFormation templates with the latest Ubuntu AMI](https://ubuntu.com/aws/docs/aws-how-to/instances/build-cloudformation-templates/)
- AWS - [EC2 instance user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)

---

**bongoDev - Learn. Practice. Prove you're ready.**
