# Lab 6 - CST8915 Full-stack Cloud-native Development: Deploy Algonquin Pet Store to Azure Kubernetes Service (AKS)

Welcome to Lab 6 of the **CST8915 Full-stack Cloud-native Development** course. In this lab, you will deploy the Algonquin Pet Store application to **Azure Kubernetes Service (AKS)**. You will create an AKS cluster, deploy the application using a provided YAML file, and expose the services for external access.

You have been building toward this step throughout the previous labs. In **Lab 1**, you ran the Pet Store on an Azure VM. In **Lab 2**, you applied the first four 12-Factor App principles and split the services across separate VMs. In **Lab 3**, you moved the APIs and frontend to managed Azure services. **Lab 4** introduced Docker, and **Lab 5** brought the Pet Store services together with Docker Compose and published your images to Docker Hub.

Now you will take your first step into **Kubernetes (K8s)**, a platform for managing containerized applications across a **cluster** of machines called **nodes**. In Lab 5, Compose described how to run your containers on one Docker host. Here, a Kubernetes YAML file describes how the application should run, and Kubernetes works to maintain that state, including replacing failed containers. You will use provided container images, so you can focus on deploying and inspecting the familiar application with `kubectl`.

> **Note:** This lab will incur an approximate cost of $3 to your Azure credit. Be sure to clean up your resources after completing the lab to avoid additional charges.


---
### What is Azure Kubernetes Service (AKS)?

[**Azure Kubernetes Service (AKS)**](https://learn.microsoft.com/en-us/azure/aks/what-is-aks) is a managed Kubernetes service provided by Microsoft Azure, designed to simplify the deployment, management, and scaling of containerized applications using Kubernetes. With AKS, Azure handles critical tasks such as provisioning, scaling, upgrading, and monitoring the Kubernetes cluster, allowing you to focus on deploying and managing your applications. AKS supports industry-standard Kubernetes, ensuring that the applications you build are portable and compatible across different environments.

Using AKS, you can:
- **Deploy and manage** applications with ease using Kubernetes’ declarative approach.
- **Scale applications** automatically, based on demand, with Kubernetes' built-in scaling mechanisms.
- **Monitor and secure** your clusters with integrated Azure security and monitoring tools.
- **Integrate with other Azure services**, such as networking, storage, and identity services, enhancing the capabilities of your applications.

---

In this lab, you’ll gain hands-on experience deploying microservices to AKS, connecting to the cluster using `kubectl`, and exposing the application to the internet.

---
## Step 1: Install `kubectl`

1. **What is `kubectl`?**
   - `kubectl` is a command-line tool that allows you to communicate with and manage Kubernetes clusters. You will use `kubectl` to deploy applications, configure clusters, and inspect resources.

2. **Installing `kubectl`:**
   - Follow the official installation guide to install `kubectl` on your system. The guide provides detailed instructions for various operating systems.
   - [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/)

3. **Verify `kubectl` Installation:**
   - After installing, confirm that `kubectl` is properly set up by running:
     ```bash
     kubectl version --client
     ```
   - You should see the client version information displayed, confirming a successful installation.

---
## Step 2: Create an Azure Kubernetes Cluster (AKS)

1. **Log in to Azure Portal:**
   - Go to [https://portal.azure.com](https://portal.azure.com) and log in with your Azure account.

2. **Create a Resource Group:**
   - In the Azure Portal, search for **Resource Groups** in the search bar.
   - Click **Create** and fill in the following:
     - **Resource group name**: `AlgonquinPetStoreRG`
     - **Region**: `Canada`.
   - Click **Review + Create** and then **Create**.

3. **Create an AKS Cluster:**
   - In the search bar, type **Kubernetes services** and click on it.
   - Click **Create** and select **Kubernetes cluster**
   - In the `Basics` tap fill in the following details:
     - **Subscription**: Select your subscription.
     - **Resource group**: Choose `AlgonquinPetStoreRG`.
     - **Cluster preset configuration**: Choose `Dev/Test`.
     - **Kubernetes cluster name**: `AlgonquinPetStoreCluster`.
     - **Region**: Same as your resource group (e.g., `Canada`).
     - **Availability zones**: `None`.
     - **AKS pricing tier**: `Free`.
     - **Kubernetes version**: `Default`.
     - **Automatic upgrade**: `Disabled`.
     - **Automatic upgrade scheduler**: `No schedule`.
     - **Node security channel type**: `None`.
     - **Security channel scheduler**: `No schedule`.
     - **Authentication and Authorization**: `Local accounts with Kubernetes RBAC`.
   - In the `Node pools` tap fill in the following details:
     - Select **agentpool**. Optionally change its name to `masterpool`. This nodes will have the controlplane.
        - Set **node size** to `D2as_v4`.
        - **Scale method**: `Manual`
        - **Node count**: `1`
        - Click `update`
     - Click on **Add node pool**:
        - **Node pool name**: `workerspool`.
        - **Mode**: `User` 
        - Set **node size** to `D2as_v4`.
        - **Scale method**: `Manual`
        - **Node count**: `1`
        - Click `add`
   - Click **Review + Create**, and then **Create**. The deployment will take a few minutes.

4. **Connect to the AKS Cluster:**
   - Once the AKS cluster is deployed, navigate to the cluster in the Azure Portal.
   - In the overview page, click on **Connect**. 
   - Select **Azure CLI** tap. You will need Azure CLI. If you don't have it: [**Install Azure CLI**](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
   - Login to your azure account using the following command:
      ```
      az login
      ```
   - Set the cluster subscription using the command shown in the portal (it will look something like this):
      ```
      az account set --subscription 'subscribtion-id'
      ```

   - Copy the command shown in the portal for configuring `kubectl` (it will look something like this):
     ```
     az aks get-credentials --resource-group AlgonquinPetStoreRG --name AlgonquinPetStoreCluster
     ```
      **Understanding the Command:**
      - The command `az aks get-credentials` pulls the necessary configuration files to enable `kubectl` to access your AKS cluster. Here’s a breakdown:
     - `--resource-group` specifies the resource group where your AKS cluster resides.
     - `--name` specifies the name of your AKS cluster.
     - `--overwrite-existing` can be used to overwrite any existing Kubernetes configuration files for the same cluster. This is useful if you’ve connected to the cluster before or if multiple configurations exist for it.
   - Verify Cluster Access:
      - Test your connection to the AKS cluster by listing all nodes:
        ```
        kubectl get nodes
        ```
        You should see details of the nodes in your AKS cluster if the connection is successful.
---

## Step 3: Deploy the Algonquin Pet Store Application

### Understanding the Provided YAML File

Open [algonquin-pet-store-all-in-one.yaml](algonquin-pet-store-all-in-one.yaml). Like the Compose file in Lab 5, it describes how the application should run. In Kubernetes, this configuration is called a **manifest**. This file defines eight resources: a Deployment and a Service for each of RabbitMQ, Order Service, Product Service, and Store Front. The `---` lines separate resources, and indentation groups related settings.

Each resource has four main fields:

| Field | Purpose |
| --- | --- |
| `apiVersion` | The Kubernetes API version used for this resource. |
| `kind` | The resource type: `Deployment` or `Service` in this file. |
| `metadata` | Identifying information, such as the resource's `name`. |
| `spec` | The desired configuration for that resource. |

- A **Deployment** manages **Pods**, which are the units Kubernetes runs. Each Pod in this lab contains one container. `replicas: 1` requests one Pod for that component. Within the Pod template, `image` selects the container image, `env` supplies environment variables, and `resources` sets CPU and memory requests and limits.
- A **Service** provides a stable network name and address for reaching Pods. Its `selector` matches the Pods' labels. RabbitMQ and the backend APIs use internal Services; the Store Front's `type: LoadBalancer` provides external access to the website.

For this lab, focus on recognizing these parts and deploying the provided file. **Lab 7** will explore YAML structure, `kubectl` operations, and creating Pods, Deployments, and Services in more detail.

1. **Apply the YAML file to the AKS cluster:**
   - In this step, use the K8s deployment YAML file provided: `algonquin-pet-store-all-in-one.yaml`.
   - The manifest uses the `lab6` tag in the existing service repositories: `ramymohamed/order-service:lab6`, `ramymohamed/product-service:lab6`, and `ramymohamed/store-front:lab6`.
   - Open the terminal and navigate to the file directory.
   - Run the following command to apply the YAML configuration and deploy the application to AKS:
      ```
      kubectl apply -f algonquin-pet-store-all-in-one.yaml
      ```

2. **Verify the deployment:**
   - After the command executes, verify that the pods are running by using the following command:
     ```bash
     kubectl get pods
     ```

3. **Check services:**
   - Confirm that all services are up and running:
     ```bash
     kubectl get services
     ```

4. **Access the Store Front Application:**
   - The **Store Front** service is configured as a LoadBalancer, which exposes the application to the internet.
   - In the Azure Portal, go to **Kubernetes Services** > **AlgonquinPetStoreCluster** > **Services and ingresses**.
   - Locate the **store-front** service, and note the **EXTERNAL-IP** address.

5. **Test the Store Front:**
   - Open a web browser and enter the external IP address to access the Store Front (You may need to wait about one minute).

6. **Verifying the backend services**:
   - `order-service` is accessible at:
      ```
      http://<EXTERNAL-IP>/orders
      ``` 
   - `product-service` is accessible at:
      ```
      http://<EXTERNAL-IP>/products
      ``` 
   - `RabbitMQ Management Dashboard` is accessible at:
      ```
      http://<EXTERNAL-IP>/rabbitmq
      ``` 
      - Use the following credentials to log in:
        - Username: myuser
        - Password: mypassword

7. **Explore Kubernetes in Action:**

   Before cleaning up, spend a few minutes experimenting with the running Store Front. You will see where Kubernetes runs your Pods, scale the website, and watch Kubernetes replace a deleted Pod.

   **A. Find your Pods and their nodes**

   ```bash
   kubectl get pods -o wide
   kubectl get pods -l app=store-front --show-labels
   ```

   The `-o wide` option shows each Pod's IP address and node. The `-l` option selects Pods with the label `app=store-front`, matching the selector used by the Store Front Service. Compare the node names: are all application Pods running on the same node?

   **B. Scale the website from one Pod to three**

   ```bash
   kubectl scale deployment/store-front --replicas=3
   kubectl rollout status deployment/store-front --timeout=120s
   kubectl get pods -l app=store-front -o wide
   kubectl get service store-front
   ```

   You should now see three Store Front Pods. Refresh the website using the same external IP address. The Service keeps its address while its selector lets it route traffic to the additional Pods. Scaling creates more Pods on the existing nodes; it does not add nodes to the cluster. See the [Kubernetes scaling tutorial](https://kubernetes.io/docs/tutorials/kubernetes-basics/scale/scale-intro/).

   **C. Delete one Pod and watch Kubernetes replace it**

   In your first terminal, watch the Store Front Pods:

   ```bash
   kubectl get pods -l app=store-front --watch
   ```

   In a second terminal connected to the same cluster and namespace, run the following command. Replace `STORE_FRONT_POD_NAME` with the name of **one** Store Front Pod from the previous output:

   ```bash
   kubectl delete pod STORE_FRONT_POD_NAME
   ```

   Watch the first terminal: the deleted Pod disappears and a new Pod with a different name is created. Kubernetes is maintaining the Deployment's desired count of three replicas. This is an example of **self-healing**: deleting a Pod does not delete its Deployment. Press `Ctrl+C` to stop watching; this only stops the command, not the Pods. See [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

   **D. Look at the frontend logs**

   ```bash
   kubectl logs deployment/store-front --tail=20
   ```

   This prints recent container output from one of the Store Front Pods. Logs help you investigate application behavior and errors without logging in to a node.

   **E. Return to one replica**

   ```bash
   kubectl scale deployment/store-front --replicas=1
   kubectl rollout status deployment/store-front --timeout=120s
   kubectl get pods -l app=store-front
   ```

   Kubernetes removes the extra Pods to match the new desired count. The `scale` commands changed the live Deployment; the YAML file still specifies `replicas: 1`. Allow any Pods marked `Terminating` to finish shutting down before continuing to cleanup.

8. **Clean Up Kubernetes Resources:**

   - Open the terminal and navigate to the `algonquin-pet-store-all-in-one.yaml` file directory.
   - Run the following command to delete all resources defined in the YAML file.
      ```
      kubectl delete -f algonquin-pet-store-all-in-one.yaml
      ``` 

9. **Clean Up Azure Resources:**

   - Delete the Primary Resource Group (AlgonquinPetStoreRG)
   - Delete the Managed Cluster Resource Group (MC_AlgonquinPetStoreRG_AlgonquinPetStoreRG_canadacentral)
   - Delete the Monitoring Resource Group (MA_defaultazuremonitorworkspace-cca_canadacentral_managed)
   - Delete the Network Watcher Resource Group (NetworkWatcherRG):
---

## Lab Tasks: Updating API Endpoints
1. Read `kubectl` documentation: https://kubernetes.io/docs/reference/kubectl/
2. Test the essential `kubectl` commands in the provided `Kubectl-Cheat-Sheet.md` file.
3. The application’s various services (such as `product-service`, `order-service`, and `RabbitMQ Management Console`) are all accessible via the same IP address, differentiated by paths (`/products`, `/orders`, `/rabbitmq`). 
    - Figure out how this setup is possible.
    - `Store-front` was slightly modified to achieve that. Examine the updated `store-front` repo: https://github.com/ramymohamed10/store-front-L6.
    - **Hint**: start by investigating `nginx.conf` file. 


---
## Submission

### What to Submit
1. **Demo Video (Max 5 minutes)**  
   - Record a short demo video for the lab showing the deployed app on your AKS cluster.
