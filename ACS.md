# Protecting Openshift Security Cluster
## Part 2. Deploying Red Hat Advanced Security Cluster and integrating it with Qradar and CP4S SOAR

Red Hat Advanced Security Cluster enables various security use cases: Compliance, Risk Profiling, Vulnerability Management, 1-click network policies and a vast range of customizable security policies to monitor and block risky activities. <br>
Moreover, the solution can be integrated with CICD tooling to block unwanted resources to be deployed in your environment. <br>
We can further extend the value of the tool by integrating it with Qradar to create offenses under certain specific scenarios, or if available, with CP4S SOAR to automatically create cases and asign them to security engineers from your organization SOC.
This guide will cover how to deploy a simple ACS cluster with these integrations.


### Activities
#### 1. Deploying RHASC
- 1.1 Creating Log Sources
- 1.2 Increasing Event size limit
#### 2. Configuring Qradar integration
- 2.1 Enabling Audit logs
- 2.2 Installing Logging Operators
- 2.3 Deploying an instance of Logging Operator
- 2.4 Deploying an instance of Logging Forwarder Operator
#### 3. Configuring a CP4S SOAR integration
- 3.1 Refine DSM
- 3.2 Estimate EPS


<br>

## 1. Deploying RHASC
[System requirements for ACS](https://docs.openshift.com/acs/3.69/installing/prerequisites.html) <br>

[Architecture and components](https://docs.openshift.com/acs/3.69/architecture/acs-architecture.html)

The ACS operator has two *deployments*: Central and Secured Cluster.

In a nutshell *Central* is the management portal and Secured Cluster the component that provides protection to the cluster. Each protected cluster needs a *Secured Cluster*, but you could have a single *Central* component connected to every *Secured Cluster*. <br>
*Central* and *Secured Cluster* can of course be deployed in the same cluster, just as we will do in this guide.

<br>

1. From the Operator Hub, search and deploy the *Advanced Cluster Security for Kubernetes* Operator, you can leave the default options, this will take a few minutes
2. Once deployed, go to *Installed Operators* and select *Advanced Cluster Security for Kubernetes*. Click on *Central* and deploy an instance of it. You can leave the default options
3. Now we need to log into the RHACS portal. You can obtain the URL from the *central* route in the *rhacs-operator* namespace. Username is **admin** and the password can be found inside the *central-htpasswd* secret in the *rhacs-operator* namespace. Is the field called *password*.
4. Next we are going to export a yaml from ACS that will allow us to connect the *Secured Cluster* we are about to deploy, to our *Central*. From the ACS console, go to *Platform Configuration > Clusters*.
5. Click on *Manage Tokens* in the top right corner. Select *Cluster Init Bundle*. Click on *Generate Bundle* and Download the Kubernetes secrets file (any name will do).
6. From the shell, apply the yaml you just downloaded with  `oc create -f secretfile.yaml -n rhacs-operator`
7. Go back to *Installed Operators* and select *Advanced Cluster Security for Kubernetes*. Click on *Secured Cluster* and deploy an instance of it.
8. Switch to form view. The *Central Endpoint* field requires the route for *central*, ie central.example.com:443. In our case since we are deploying in the same cluster we can it leave empty. We can leave the remaining options as default, and click Create.
9. Once the deployment is finalized, you can return to the ACS portal and confirm that the application is working properly. You should be seeing for example results from vulnerabilty scans, like below:

![image](https://user-images.githubusercontent.com/75438200/166227544-cce99568-20e8-45da-be26-21d4f928ac73.png)

## 2. Integrating with Qradar
You can configure your ACS policies to generate alerts when triggered. The alerts can go to many 3rd party systems for different uses, and in this case we will explore forwarding these alerts to Qradar. For example: Alert when someone tries to deploy an image with known vulnerabilities with a CVSS over 7 (High and Critical).
Triggering such policy would send an event to Qradar, and in Qradar we would have created a rule to raise an "Offense" upon receiving that sort of event.
Let's look into that.

1.

