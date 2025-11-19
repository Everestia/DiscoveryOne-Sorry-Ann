<div align="center">

<pre>
=============================================================================
   // DEPLOYMENT PROCEDURE: NETWORK (TASK A) //
=============================================================================
</pre>

<p><em>This document outlines the procedure for provisioning the core network infrastructure.</em></p>

</div>

---

### 🌌 Step 1: Provision VPC (Virtual Private Cloud)
> **[ RATIONALE: VPC Configuration ]**
> * **Custom Subnet Mode:** We are explicitly creating a `custom` mode VPC. This is a security best practice, as it avoids the default `auto` mode which provisions insecure, pre-defined subnets in every region. This gives us full control over our IP address management (IPAM) and network segmentation.
> * **Regional BGP Routing:** We've selected `regional` routing. For this mission's scope, we do not require global dynamic routing, which would add unnecessary complexity and cost. Regional routing is sufficient for our multi-region, non-complex topology.

**Action:**
Create the primary VPC.

```bash
##Create VPC network
#short name for convnience, long description for documentation
gcloud compute networks create ${vpc} \
    --subnet-mode=custom \
    --description="Custom Regional VPC Network" \
    --bgp-routing-mode=regional

```
---

### 🌐 Step 2: Create Subnets
> **[ RATIONALE: Network Segmentation ]** 
> We are provisioning four subnets to create a `multi-region, two-tier architecture (frontend and backend)`.
> * **Frontend Subnets:** `Intended for public-facing components`, like our Load Balancer and web servers.
> * **Backend Subnets:** `Intended for internal services` (e.g., databases, microservices) that should not be directly exposed to the internet. This segmentation is fundamental for applying granular firewall rules and securing our application.

**Action:**
Create subnets in both US and EU regions.

```bash
##Create frontend subnets
#not sure if needed
#US subnet
gcloud compute networks subnets create ${vpc}-frontend-$subnetus \
    --description="Frontend subnet in $regionus" \
    --range=10.10.1.0/24 \
    --stack-type=IPV4_ONLY \
    --network=$vpc \
    --region=$regionus
#EU subnet
gcloud compute networks subnets create ${vpc}-frontend-$subneteu \
    --description="Frontend subnet in $regioneu" \
    --range=10.10.2.0/24 \
    --stack-type=IPV4_ONLY \
    --network=$vpc \
    --region=$regioneu

##Create backend subnets
#US subnet
gcloud compute networks subnets create ${vpc}-backend-$subnetus \
    --description="Backend subnet in $regionus" \
    --range=10.20.1.0/24 \
    --stack-type=IPV4_ONLY \
    --network=$vpc \
    --region=$regionus
#EU subnet
gcloud compute networks subnets create ${vpc}-backend-$subneteu \
    --description="Backend subnet in $regioneu" \
    --range=10.20.2.0/24 \
    --stack-type=IPV4_ONLY \
    --network=$vpc \
    --region=$regioneu

```
---
### 🛡️ Step 3: Configure Firewall Rules
> **[ RATIONALE: Ingress Access Control ]** 
> We are defining the initial "allow" rules for our VPC.
> * **`allow-icmp`:** This rule is opened for troubleshooting and diagnostics (e.g., `ping`).
> * **`allow-lb-health-checks`:** This is a *critical* rule. It allows Google's global health checking systems (from the official IP ranges `130.211.0.0/22` and `35.191.0.0/16`) to reach our web servers on port 80. We must create this manually because GCP's built-in rule only applies to the `default` network.

<div align="center">
<pre>
+=================================================================+
|                   ! ! ! W A R N I N G ! ! !                     |
+-----------------------------------------------------------------+
</pre>
</div>

<table align="center">
<tr>
<td>

```diff
-                        Significant Security Vulnerability

- The `allow-ssh` rule with source -->> 0.0.0.0/0 <<-- opens SSH access to the entire Internet.
- This configuration is for >>TEMPORARY SETUP AND TROUBLESHOOTING ONLY<<.
- It MUST be replaced with a more restrictive rule before any production use.
```
</td>
</tr>
</table>

**Action:**
Create the firewall rules.

```bash
##Create Firewall Rules
#FW rule for ssh (troubleshooting)
gcloud compute firewall-rules create ${vpc}-allow-ssh \
    --network=$vpc \
    --allow=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --description="Allow SSH from anywhere (TEMPORARY)"

#FW rule for ICMP (troubleshooting)
gcloud compute firewall-rules create ${vpc}-allow-icmp \
    --network=$vpc \
    --allow=icmp \
    --source-ranges=0.0.0.0/0 \
    --description="Allow ICMP from anywhere (TEMPORARY)"

#FW rule for LB healthchecks
gcloud compute firewall-rules create ${vpc}-allow-lb-health-checks \
    --network=$vpc \
    --description="Allows traffic from Google Cloud health checking systems over port 80" \
    --direction=INGRESS \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --action=ALLOW \
    --rules=tcp:80 \
    --target-tags=$healthchecktag

```
> **💫[!NOTE]💫👩‍🚀👩‍🚀👩‍🚀👩‍🚀👩‍🚀**
```Diff
+ Operator's Log:
```
> * **TODO-1:** _Test if the open internal-all rule (as seen in Ania's example) is strictly necessary for communication between frontend and backend tiers, or if more granular rules can beapplied_.
> * **TODO-2:** _Allow unrestricted SSH until I figure something better_.
---
### ❤️ Step 4: Define Global Health Check

> **[ RATIONALE: Instance Health ]**
> * This `http` health check will be used by our Global Load Balancer. It defines the criteria for a "healthy" instance. The LB will periodically send a request to port 80 of each VM; if it fails to receive a successful response within the defined threshold, it will stop sending user traffic to that instance (and autohealing will eventually replace it).

**Action:**
Create the global health check resource.

```bash
##Healthcheck
gcloud compute health-checks create http $healthcheckname --port=80 --check-interval=10 --timeout=10 --unhealthy-threshold=3

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
> * Resources must be deleted in the reverse order of creation to respect dependencies.
> * You must delete health checks and firewall rules before deleting the subnets, and delete all subnets before deleting the VPC network itself. 
> * The `--quiet` flag is added to suppress interactive confirmation prompts.

**Action:**
```Diff
- Run the following commands to tear down the network infrastructure⚠️.
```
```bash
### Network
## Delete healtcheck
gcloud compute health-checks delete $healthcheckname --quiet

## Delete Firewall Rules
gcloud compute firewall-rules delete $vpc-allow-ssh --quiet
gcloud compute firewall-rules delete $vpc-allow-icmp --quiet
gcloud compute firewall-rules delete $vpc-allow-lb-health-checks --quiet

## Delete subnets
gcloud compute networks subnets delete $vpc-frontend-$subnetus --quiet
gcloud compute networks subnets delete $vpc-frontend-$subneteu --quiet
gcloud compute networks subnets delete $vpc-backend-$subnetus --quiet
gcloud compute networks subnets delete $vpc-backend-$subneteu --quiet

## Delete Network
gcloud compute networks delete $vpc --quiet

```
---
<table align="center"> <tr> <td>

```Diff
+ "All anomalous elements have been purged (HAL 9000)."
````
<div align="center">
  <img src="../assets/monolith.png" width="300"/>
</div>
