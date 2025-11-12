<div align="center">

<pre>
=============================================================================
  // DEPLOYMENT PROCEDURE: COMPUTE (TASKS B & half C) //
=============================================================================
</pre>

<p><em>This document outlines the procedure for provisioning the multi-region compute layer.</em></p>

</div>

---
> **💫[!NOTE]💫👩‍🚀👩‍🚀👩‍🚀👩‍🚀👩‍🚀**

```Diff
+ Operator's Log:
```
> * Initial tests with custom Ania scripts (`wgrywka.sh`) failed.
> * `cloud-init` was selected as the stable, declarative alternative. 
> * The use of Ubuntu Minimal 24.04 presented its own challenges; a V2 iteration should evaluate a different base image (e.g., Debian or Google's Container-Optimized OS).
---
### 🔭🌑 Introduction: Compute Strategy
> **[ RATIONALE: Compute Strategy ]**
> * This procedure provisions the application compute layer. We are creating two identical, multi-zone **Managed Instance Groups (MIGs)**, one in the **US** and one in the **EU**, for high availability and regional redundancy.
> * The configuration for these MIGs is defined by **Instance Templates**.

   =============================================================================

### 🌎🇺🇸 Step 1: Provision US-Central1 Compute Resources

#### 1a. Create Instance Template (US)

> **[ RATIONALE: Instance Template Configuration ]**
> The Instance Template is the "blueprint" for our VMs.
> * **`user-data`**: We pass the `cloud-init.yaml` file (created in the previous module). This is the most reliable method for bootstrapping the instance (installing Nginx, etc.).
> * **Shielded VM**: We enable `--shielded-secure-boot` and `--shielded-vtpm` to ensure the integrity of the boot process and protect against kernel-level malware or rootkits.
> * **Tags**: The `$healthchecktag` is applied. This is critical, as it allows our Firewall Rule (from the network module) to permit traffic from the GCP health checkers.

**Action:**
Create the US Instance Template.

```bash
## US side
#Create Instance Template
gcloud compute instance-templates create "templatka-${regionus}-v1" \
  --network="${vpc}" \
  --region="${regionus}" \
  --subnet="${vpc}-backend-${subnetus}" \
  --machine-type="e2-micro" \
  --tags="${healthchecktag}" \
  --metadata-from-file="user-data=cloud-init.yaml" \
  --create-disk="auto-delete=yes,boot=yes,device-name=persistent-disk-0,image=projects/ubuntu-os-cloud/global/images/ubuntu-minimal-2404-noble-amd64-v20251020,mode=rw,size=10,type=pd-balanced" \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring

```
---
#### 1b. Create Managed Instance Group (US)

> **[ RATIONALE: MIG Configuration ]**
> The MIG manages the lifecycle of our VM instances.
> * **`Multi-Zone`**: We deploy across zones `b` and `c` for high availability within the region.
> * **`Health Check`**: We link the `$healthcheckname` (created in the network step). This enables **auto-healing**; if an instance fails this check, the MIG will automatically recreate it. 
> * **`Initial Delay`**: We set a `300s` (5 min) delay. This is critical, as it gives `cloud-init` ample time to install Nginx and start the webserver before the health checker begins probing the instance. Without this, the MIG might enter a reboot-loop.

**Action:**
Create the US MIG✈️.

```bash
#Create MIG
gcloud compute instance-groups managed create mig-$regionus \
    --template=templatka-$regionus-v1 \
    --size=2 \
    --zones=$regionus-b,$regionus-c \
    --health-check=$healthcheckname \
    --default-action-on-vm-failure=repair \
    --initial-delay=300

```
---
#### 1c. Configure Autoscaling (US)

> **[ RATIONALE: Autoscaling Policy ]**
> We are applying a simple, CPU-based autoscaling policy.
> * It will maintain a minimum of 2 instances and scale up to 4 if the average CPU utilization exceeds 60%.
> * The predictive method is disabled for this (V1) simple configuration.

**Action:**
Set the autoscaling policy for the US MIG✈️.

```bash
#Set autoscaling
gcloud compute instance-groups managed set-autoscaling mig-$regionus \
    --region=$regionus \
    --mode=on \
    --min-num-replicas=2 \
    --max-num-replicas=4 \
    --target-cpu-utilization=0.6 \
    --cpu-utilization-predictive-method=none

```
---
#### 1d. Set Named Ports (US)

> **[ RATIONALE: Service Port Mapping ]**
> The Global Load Balancer (which we will build next) needs to know which port to forward traffic to.
> * Setting a "named port" (`http80:80`) creates a key-value pair. Our LB Backend Service will reference this name (`http80`) instead of a hard-coded port number, which is a more robust practice.

**Action:**
Set the named port for the US MIG✈️.

```bash
#Create named ports inside MIG
gcloud compute instance-groups set-named-ports mig-$regionus \
    --named-ports=http80:80 \
    --region=$regionus

```
---

---
### 🌍🇪🇺 Step 2: Provision EU-West4 Compute Resources

#### 2a, b, c, d. 

> **[ RATIONALE: Multi-Region ]**
> * This procedure is an identical mirror of Step 1, deployed to the `$regioneu` region using the corresponding EU variables. This establishes our multi-region footprint for the application.

**Action a:** 
Create the EU Instance Template.

```bash
## EU side
#Create Instance Template
gcloud compute instance-templates create "templatka-${regioneu}-v1" \
  --network="${vpc}" \
  --region="${regioneu}" \
  --subnet="${vpc}-backend-${subneteu}" \
  --machine-type="e2-micro" \
  --tags="${healthchecktag}" \
  --metadata-from-file="user-data=cloud-init.yaml" \
  --create-disk="auto-delete=yes,boot=yes,device-name=persistent-disk-0,image=projects/ubuntu-os-cloud/global/images/ubuntu-minimal-2404-noble-amd64-v20251020,mode=rw,size=10,type=pd-balanced" \
  --shielded-secure-boot \
  --shielded-vtpm \
  --shielded-integrity-monitoring

```
**Action b:**
Create the EU MIG🚀.

```bash
#Create MIG
gcloud compute instance-groups managed create mig-$regioneu \
    --template=templatka-$regioneu-v1 \
    --size=2 \
    --zones=$regioneu-b,$regioneu-c \
    --health-check=$healthcheckname \
    --default-action-on-vm-failure=repair \
    --initial-delay=300

```
**Action c:**
Set the autoscaling policy for the EU MIG🚀.

```bash
#Set autoscaling
gcloud compute instance-groups managed set-autoscaling mig-$regioneu \
    --region=$regioneu \
    --mode=on \
    --min-num-replicas=2 \
    --max-num-replicas=4 \
    --target-cpu-utilization=0.6 \
    --cpu-utilization-predictive-method=none

```
**Action d:**
Set the named port for the EU MIG🚀.

```bash
#Create named ports inside MIG
gcloud compute instance-groups set-named-ports mig-$regioneu \
    --named-ports=http80:80 \
    --region=$regioneu

```
---

<div align="center"> <pre>
⚠️ 🧹 DEPROVISIONING PROCEDURE (CLEANUP) 🧹⚠️
</pre> </div>

<div align="center">
<pre>
+=================================================================+
|                   ! ! ! W A R N I N G ! ! !                     |
+-----------------------------------------------------------------+
These commands will permanently destroy
the resources created in the steps above.
+=================================================================+
</pre>
</div>

> **[ RATIONALE: Deletion Order ]**
> * The Managed Instance Groups (MIGs) must be deleted first, as they hold an active dependency on the Instance Templates.
> * Once the MIGs are terminated, the templates can be safely removed.

**Action:**
```Diff
- Run the following commands to tear down the network infrastructure⚠️.
```
```bash 
##Application
#MIG
gcloud compute instance-groups managed delete mig-${regionus} --region ${regionus} --quiet
gcloud compute instance-groups managed delete mig-${regioneu} --region ${regioneu} --quiet


gcloud compute instance-templates delete "templatka-${regionus}-v1"  --quiet
gcloud compute instance-templates delete "templatka-${regioneu}-v1"  --quiet

```
---
<table align="center"> <tr> <td>

```Diff
+ "My functions are operating perfectly (HAL 9000)."
```
<div align="center">
  <img src="../assets/monolith.png" width="300"/>
</div>
