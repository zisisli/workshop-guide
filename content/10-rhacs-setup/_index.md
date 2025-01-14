+++
title = "Install and Configure ACS"
weight = 15
+++

During the workshop you went through the OpenShift developer experience starting from software development using Quarkus and `odo`, moving on to automating build and deployment using Tekton pipelines and finally using GitOps for production deployments.

Now it's time to add another extremely important piece to the setup; enhancing application security in a containerized world. Using a recent addition to the OpenShift portfolio: **Red Hat Advanced Cluster Security for Kubernetes**!

## Install RHACS

### Install the Operator

Install the "Advanced Cluster Security for Kubernetes" operator from OperatorHub:

- Switch **Update approval** to `Manual`
- Apart from this use the default settings
- Approve the installation when asked

{{% notice info %}}
Red Hat recommends installing the Red Hat Advanced Cluster Security for Kubernetes Operator in the **rhacs-operator** namespace. This will happen by default..
{{% /notice %}}

### Install the main component **Central**

{{% notice info %}}
You must install the ACS Central instance in its own project and not in the **rhacs-operator** and **openshift-operator** projects, or in any project in which you have installed the ACS Operator!
{{% /notice %}}

- Navigate to **Operators → Installed Operators**
- Select the ACS operator
- You should now be in the **rhacs-operator** project the Operator created, create a new OpenShift **Project** for the **Central** instance:
  - Select **Project: rhacs-operator → Create project**
  - Create a new project called **stackrox** (Red Hat recommends using **stackrox** as the project name.)
- In the Operator view under **Provided APIs** on the tile **Central** click **Create Instance**
- Accept the name **stackrox-central-services**
- Adjust the memory limit of the central instance to `6Gi` (**Central Component Settings->Resources->Limits>Memory**).
- Click **Create**

After deployment has finished (**Status** `Conditions: Deployed, Initialized` in the Operator view on the tab **Central**) it can take some time until the application is completely up and running. One easy way to check the state is to switch to the **Developer** console view at the upper left. Then make sure you are in the **stackrox** project and open the **Topology** map. You'll see the three deployments of an **Central** instance:

- **scanner-db**
- **scanner**
- **centrals**

Wait until all Pods have been scaled up properly.

**Verify the Installation**

Switch to the **Administrator** console view again. Now to check the installation of your **Central** instance, access the **ACS Portal**:

- Look up the **central-htpasswd** secret that was created to get the password

{{% notice info %}}
If you access the details of your **Central** instance in the Operator page you'll find the complete commandline using `oc` to retrieve the password from the secret under `Admin Credentials Info`. Just sayin... ;)
{{% /notice %}}

- Look up and access the route **central** which was also generated automatically.

This will get you to the **ACS Portal**, accept the self-signed certificate and login as user **admin** with the password from the secret.

Now you have a **Central** instance that provides the following services in an
RHACS setup:

- The application management interface and services. It handles data persistence, API interactions, and user interface access. You can use the same **Central** instance to **secure multiple** OpenShift or Kubernetes clusters.

- Scanner, which is a vulnerability scanner for scanning container images. It analyzes all image layers to check known vulnerabilities from the Common Vulnerabilities and Exposures (CVEs) list. Scanner also identifies vulnerabilities in packages installed by package managers and in dependencies for multiple programming languages.

To actually do and see anything you need to add a **SecuredCluster** (be it the same or another OpenShift cluster). For effect go to the **ACS Portal**, the Dashboard should by pretty empty, click on the **Compliance** link in the menu to the left, lots of zero's and empty panels, too.

This is because you don't have a monitored and secured OpenShift cluster yet.

### Prepare to add Secured Clusters

First you have to generate an init bundle which contains certificates and is used to authenticate a **SecuredCluster** to the **Central** instance, again regardless if it's the same cluster as the Central instance or a remote/other cluster.

In the **ACS Portal**:

- Navigate to **Platform Configuration → Integrations**.
- Under the **Authentication Tokens** section, click on **Cluster Init Bundle**.
- Click **Generate bundle**
- Enter a name for the cluster init bundle and click **Generate**.
- Click **Download Kubernetes Secret File** to download the generated bundle.

The init bundle needs to be applied on all OpenShift clusters you want to secure & monitor.

### Prepare the Secured Cluster

For this workshop we run **Central** and **SecuredCluster** on one OpenShift cluster. E.g. we monitor and secure the same cluster the central services live on.

**Apply the init bundle**

- Use the `oc` command to log in to the OpenShift cluster as `cluster-admin`.
  - The easiest way might be to use the **Copy login command** link from the UI
- Switch to the **Project** you installed **ACS Central** in, it should be `stackrox`.
- Run `oc create -f <init_bundle>.yaml -n stackrox` pointing to the init bundle you downloaded from the Central instance and the Project you created.
- This will create a number of secrets:

```
secret/collector-tls created
secret/sensor-tls created
secret/admission-control-tls created
```

### Add the Cluster as **SecuredCluster** to **ACS Central**

You are ready to install the **SecuredClusters** instance, this will deploy the secured cluster services:

- In the **OpenShift Web Console** go to the **ACS Operator** in **Operators->Installed Operators**
- Using the Operator create an instance of the **Secured Cluster** type **in the Project you created** (should be stackrox)
- Change the **Cluster Name** for the cluster if you want, it'll appear under this name in the **ACS Portal**
- And most importantly for **Central Endpoint** enter the address and port number of your **Central** instance, this is the same as the **ACS Portal**.
  - If the **ACS Portal** is available at `https://central-stackrox.apps.cluster-65h4j.65h4j.sandbox1803.opentlc.com/` the endpoint is `central-stackrox.apps.cluster-65h4j.65h4j.sandbox1803.opentlc.com:443`.
- Under **Admission Control Settings** make sure
  - **listenOnCreates**, **listenOnUpdates** and **ListenOnEvents** is enabled
  - Set **Contact Image Scanners** to **ScanIfMissing**
- Click **Create**

Now go to your **ACS Portal** again, after a couple of minutes you should see you secured cluster under **Platform Configuration->Clusters**. Wait until all **Cluster Status** indicators become green.

### Create a serviceaccount to scan the internal OpenShift registry

The integrations to the internal registry were created automatically. But to enable scanning of images in the internal registry, you'll have to configure valid credentials, so this is what you'll do:

- add a serviceaccount
- assign it the needed privileges
- configure the **Integrations** in ACS with the new credentials

But the first step is to disable the auto-generate mechanism, otherwise your updated credentials would be set back automatically:

- In the **OpenShift Web Console**, switch to the project **stackrox**, go to **Installed Operators->Advanced Cluster Security for Kubernetes**
- Open your Central instance `stackrox-central-services`
- Switch to the YAML view, under `spec:` add the following YAML snippet (one indent):

```
customize:
    envVars:
      - name: ROX_DISABLE_AUTOGENERATED_REGISTRIES
        value: 'true'
```

- Click **Save**

**Create ServiceAccount to read images from Registry**

- In the **OpenShift Web Console** make sure you are still in the `stackrox` **Project**
- **User Management -> ServiceAccounts -> Create ServiceAccount**
- Replace the example name in the YAML with `acs-registry-reader` and click **Create**
- In the new ServiceAccount, under **Secrets** click one of the `acs-registry-reader-token-...` secrets
- Under **Data** copy the Token
- Using `oc` give the ServiceAccount the right to **read** images from all projects:

```
oc adm policy add-cluster-role-to-user 'system:image-puller' system:serviceaccount:stackrox:acs-registry-reader -n stackrox
```

**Configure Registry Integrations in ACS**

Access the **ACS Portal** and configure the already existing integrations of type **Generic Docker Registry**. Go to **Platform Configuration -> Integrations -> Generic Docker Registry**. You should see a number of autogenerated (from existing pull-secrets) entries.

You have to change four entries pointing to the internal registry, you can easily recognize them by the placeholder **Username** `serviceaccount`.

For each of the four local registry integrations click **Edit integration** using the three dots at the right:

- Put in `acs-registry-reader` as **Username**
- Paste the token you copied from the secret into the **Password** field
- Select Disable TLS certificate validation
- Press the **Test** button to validate the connection and press **Save** when the test is successful.

**ACS is now able to scan images in the internal registry!**
