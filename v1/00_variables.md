<div align="center">

<pre>
=============================================================================
   // DEPLOYMENT PROCEDURE: PAYLOAD (TASK 0) //
=============================================================================
</pre>

<p><em>This document outlines the procedure for deploying the core application payload (Nginx).</em></p>

</div>

---

### 🛰️ Step 1: Prepare Application Payload (cloud-init)

This first step creates the configuration file for our virtual machine.

> **[RATIONALE: Why use cloud-init?]**
> We are using `cloud-init` because it is the modern standard for bootstrapping cloud instances.
> * **It is declarative:** Much like Terraform, we *declare* the state we want (e.g., "nginx installed," "this file exists"), and `cloud-init` figures out *how* to do it.
> * **It is compatible:** It works perfectly on our chosen OS, Ubuntu 24.04 LTS, and is supported by all major cloud providers.
> * **It is clean:** It keeps our VM templates simple, as the VM only needs to fetch this one script on its first boot.
---
**Action:**
The following command creates a `cloud-init.yaml` file in our directory. This file instructs the new VM to update packages, install Nginx, and create a custom webpage.

```bash
##VARIABLES
#Cloud-init config
cat << 'EOF' > "cloud-init.yaml"
#cloud-config
# version: 1.1.6  (2025-11-11)
# owner: Sati
# change: switch permissions, add sed, runcmd in single line
# scope: update package list, install nginx, set as service and configure webpage
package_update: true
packages:
  - nginx
write_files:
  - path: /var/www/html/index.html
    permissions: '0644'
    content: |
      <!DOCTYPE html>
      <html>
        <body style="background-color:rgb(94,107,47); font-family:sans-serif;">
          <h1><b>Czesc Ania</b>, witaj w projekcie pierwszym grupy pierwszej.</h1>
          <h2>Zyczymy owocnego zwiedzania &lt;3 </h2>
          <p><strong>VM Hostname:</strong> ##MY_HOSTNAME##</p>
          <p><strong>VM IP Address:</strong> ##MY_IP_ADDRESS##</p>
          <p><strong>Application Version:</strong> v4 </p>
          <p>Pozdrawiamy: Agnieszka i Adrianna </p>
        </body>
      </html>
runcmd:
  - |
    #!/bin/bash
    HNAME=$(hostname)
    IP_ADDR=$(hostname -I)
    sed -i "s/##MY_HOSTNAME##/$HNAME/g" /var/www/html/index.html
    sed -i "s/##MY_IP_ADDRESS##/$IP_ADDR/g" /var/www/html/index.html
    systemctl enable --now nginx
EOF

```
```Diff
+ ✨🪐🛸CUSTOM WEBPAGE SCREENSHOT🛸🪐✨
```
<div align="center">
  <img src="../assets/stronka.jpg" width="900"/>
</div>

---
### ⚙️ Step 2: Set Mission Parameters (Variables)
Next, we export our environmental variables.

> **[ RATIONALE: Why use variables? ]**
> This separates our configuration (the "what," e.g., `vpc1-custom`) from our logic (the "how," e.g. ,`gcloud compute networks create`).
> * **It's safer:** It prevents typos and mistakes.
> * **It's collaborative:** Everyone on the team can use the same script and just `change their project variable if needed`.
> * **It's reusable:** We can easily reuse these scripts for a different project (V2) just by sourcing a different variable file.
---
```diff
- Note: The project variable must be changed, as each GCP Project ID is globally unique.
```
**Action:** Run these `export commands` in your terminal. They will be available for the duration of your terminal session.

```bash
export project=levelup-sati-t1
export regionus=us-central1 #because it is cheap
export regioneu=europe-west4 #beacuse it is in Europe
export healthchecktag=lb-health-checks-enabled
export healthcheckname=global-http-health-check
export vpc=vpc1-custom
export subnetus=subnet-us-central1
export subneteu=subnet-eu-west4
```
---
### 🔧 Step 3: Project Initialization (Optional)

<table align="centre">
<tr>
<td>
------ * ------ * ---
This step is for a "cold start" mission,
where the project does not exist yet.
------ * ------ * ---
</td>
</tr>
</table>

```diff
- Note: [!WARNING] Do not run this if your project already exists!
```
> **[ RATIONALE: When to use this? ]** 
> * **This is a one-time setup command**: The `gcloud config set project` command is a convenience that tells `gcloud` which project to use for all subsequent commands.
> * That's why: you don't have to add `--project=$project` to every command.
---
**Action:** (Optional) Run the following to create and configure the new project.

```bash
# Create new project
gcloud projects create $project
# now switch to project and define variables
# Set newly created project as default for cli convnience
gcloud config set project $project
```
---
### 🧹 Step 4: Post-Mission Cleanup (Optional)
This final step removes temporary files from your local directory.

> **[ RATIONALE: Why clean up? ]**
> * This step is **optional**. Its purpose is to clean up temporary files created during the deployment process (like `cloud-init.yaml`).

> You should run this when you want to "start from scratch" for a future deployment, ensuring that no old, auto-generated files are left behind. The `cloud-init.yaml` is a *temporary artifact* generated by Step 1; it is not meant to be saved in the repository.

**Action:**
Remove the temporary `cloud-init.yaml` file.

```bash
rm cloud-init.yaml
```
---
<table align="center">
<tr>
<td>

```diff
+ "I'm completely operational, and all my circuits are functioning perfectly (HAL 9000)."
```
<div align="center">
  <img src="../assets/monolith.png" width="300"/>
</div>
