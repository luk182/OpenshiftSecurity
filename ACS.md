# Protecting Openshift Security Cluster
## Part 2. Deploying Red Hat Advanced Security Cluster and integrating it with Qradar and CP4S SOAR

Red Hat Advanced Security Cluster enables various security use cases: Compliance, Risk Profiling, Vulnerability Management, 1-click network policies and a vast range of customizable security policies to monitor and block risky activities. <br>
Moreover, the solution can be integrated with CICD tooling to block unwanted resources to be deployed in your environment. <br>
We can further extend the value of the tool by integrating it with Qradar to create offenses under certain specific scenarios, or if available, with CP4S SOAR to automatically create cases and asign them to security engineers from your organization SOC.
This guide will cover how to deploy a simple ACS cluster with these integrations.


### Activities
#### 1. Deploying RHASC
#### 2. Integration with Qradar
#### 3. Integration with CP4S SOAR

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
ACS policies can be configured to generate alerts when triggered. These alerts can be routed to a variety of receivers, like Slack or Microsoft Teams for example.
In this example we will configure an integration to forward these alerts to Qradar.
To showcase a valid usecase, we are going to use an out of the box ACS policy which triggers when someone tries to deploy an image with known vulnerabilities with a CVSS over 7 (High and Critical).

<br>

### Adding the Log Source in Qradar
1. From the Qradar console, go to *Admin* > *Log Sources* and launch the app
2. Create a new log source based on the type **Red Hat Advanced Cluster Security**
3. Key paraments to pay attention to: **Protocol** (HTTP or HTTPS), **Port**, And **Log source identifier**. For example: *acs.testcluster.com*. <br> After the Log Source is deployed, Qradar will need to restart its services.
### Configuring ACS integration
4. From the ACS portal, go to *Platform Configuration* > *Integrations*.
6. Click on Generic Webhook Integratoin
7. On endpoint, use https://qradarname:port if you are using http, otherwise same but using http://
8. Copy the Qradar CA certificate content to the *CA certificate*
9. Add a new Header, key is *hostname*, and value is the log source identifier defined earlier. *acs.testcluster.com*
10. Click on *Test* to test the integration. If it works, Save it.

![image](https://user-images.githubusercontent.com/75438200/166304686-cd4ce623-20f8-4d3a-86d2-0c4441273eb3.png)

## 3. Integrating with Cloud Pak for Security SOAR
Raising Offenses in Qradar upon Policy Violations is useful. But raising an incident/case in a SOAR tool for proper investigation is even better.
In this guide we will see how to automatically create cases on CP4S SOAR. As a pre-requisite, we would need to be using the functionality to create cases from an email on CP4S.

<br>

First we will need a mailbox that can be integrated to CP4S. Emails received on this address will automatically create cases under defined circumstances.
For the first few steps of the email integration I'm going to refer you to the official IBM CP4S documentation, for version 1.9

1. [Creating an email Connection](https://www.ibm.com/docs/en/cloud-paks/cp-security/1.9?topic=email-lesson-1-creating-connection)
2. [Assigning email permissions](https://www.ibm.com/docs/en/cloud-paks/cp-security/1.9?topic=email-lesson-2-assigning-permissions)
3. [Configuring a email script](https://www.ibm.com/docs/en/cloud-paks/cp-security/1.9?topic=email-lesson-3-configuring-sample-script) 
On this step we are going to modify the default script. We want the content of the email to be added as *Note* in the newly created case, so the analysts working on the case will have the all the useful information from the alert available to work with.
For this to happen, we will add a single line of code, on the script line **323**: `incident.addNote(emailmessage.body.content)` <br>
You could also modify the logic behind the naming of the incident that is about to be created, in line *532 newIncidentTitle*.

<br>

![image](https://user-images.githubusercontent.com/75438200/166680058-3a8ede66-2ade-442a-a0f4-9587a06b9894.png)

4. CP4S is now connected to a mailbox and checking every email received without taking any action. <br> We are going to define a rule to create cases when receiving emails coming from a certain sender. <br> From the CP4S portal, go to *Application settings* > *Case Management* > *Customization*.
5. Click on the *Rules* tab, and create a new Rule.
6. Add two conditions. One is *Email Message is created*, the other one is *From Address is equal to* the sender address that ACS uses to send email alerts.
7. In Activities, select *Run Script* and choose the script that was created on step 3. Click on *Save and Close*.

![image](https://user-images.githubusercontent.com/75438200/166683247-3c49aaf6-2c3d-4c65-b3ed-28aec67bc30c.png)

8. Now that CP4S can create cases from emails, we will configure the integration at Red Hat Advanced Cluster Security. <br>
From the ACS portal, go to *Platform Configuration* > *Integrations*.
10. Select Email, and create a new integration to your SMTP server. <br> Remember that the **SENDER** address will need to match the one you defined on step 7. You can test the integration by clicking on *Test*. 
11. After the email integration is configured, you can proceed to modify your ACS Policies to **NOTIFY** upon detection, and select this email integration as the notification method. That would be the last step.

![image](https://user-images.githubusercontent.com/75438200/166684850-c8e8145c-3e7d-4219-b3da-5198064f3d4c.png)


![image](https://user-images.githubusercontent.com/75438200/166684250-7fd9fad0-3b0d-4327-ac9e-fac4aa3c8400.png)










