## DNS & IP

### Reserve static IP in GCP

gcloud compute addresses create lb-external-static-ip  --global

gcloud compute addresses describe lb-external-static-ip --global


### Reserve hostname on afraid.org

Create an account on afraid.org
![alt text](image.png)
Confirm your account (email).


Navigate to https://freedns.afraid.org/domain/registry/ to find domian you would like to use. Make sure to select one marked as public, otherwise you will have to wait for approval:
![alt text](image-1.png)
Click on domain name, then on the next page put: 
- Subdomain: your preferred hostname
- Destination: external IP you reserved for your loadbalancer
![alt text](image-2.png)

Wait 15-30 minutes for your domain to become globally reachable.

## Loadbalancer setup
In GCP web console navigate to Network Services / Load balancing and press Create load balancer:
![alt text](image-3.png)


Select Application Load Balancer (HTTP/HTTPS):
![alt text](image-4.png)


Select external:

![](image-5.png)


Select global:
![alt text](image-6.png)

Select Global external Application Load Balancer:
![alt text](image-7.png)

Then press Configure.

In the next page, provide:
- name for your loadbalancer
- name for your frontend
- Protocol: set to HTTPS
- IP address: Select IP we selected before.  


![alt text](image-10.png)

In certificate selection field, press "Create a new certificate: 
![alt text](image-8.png)


On next page, provide name for your cert, domain you registered on afraid.org and select "Create Google-managed certificate", then press Create.

![alt text](image-9.png)


Almost there, now click to Backend configuration and "Create a backend service":

![alt text](image-11.png)

Then:
 - add name for your backend service
 - select existing global healthcheck
 - Select existing MIG in Backends/Instance group:

![alt text](image-13.png)

Use existing named prot when prompted:
![alt text](image-12.png)

Press Add a backend and select second MIG:
![alt text](image-14.png) 

Finally, uncheck "Enable Cloud CDN":

![alt text](image-15.png)

and press Create on Create backend service subpage.


Press final "Create":

![alt text](image-16.png)

After a few minutes, your loadbalancer should be active.
