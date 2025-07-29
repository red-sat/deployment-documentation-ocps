This documentation follows the exact advice of:
-	Nabil El Mahyaoui (Principal) 
-	Mahmoud Zridi (Frontend Developer)

Which is: 
-	For an end-to-end project, it’s great to use Azure DevOps platform. Where we can use **Azure Boards** (Sprints, backlogs, Delivery Plans, etc.), **Azure Repos** (for code orchestration), **Azure Pipelines** (for CI/CD pipelines by configuring YAML files in the code repository), etc.
-	We should deploy Azure services (PaaS, aka: Platform as a Service) for every component (Frontend, Backend, Database, file storage, etc.) of the application instead of managing a Virtual Machine (VM or IaaS, aka: Infrastructure as a Service), this latter is considered (in most cases) expensive and demanding in terms of managing infrastructure! 

## Plan of the deployment strategy in azure

Deploying a multi-component application like “AI4KM” to Azure DevOps is a classic and powerful workflow. Let's break down the process step-by-step.

We'll structure this as a non-exhaustive high-level plan, focusing on the key stages and decisions we'll need to make.

Our application has three main parts:
1.  **Backend**: Python FastAPI application.
2.  **Frontend**: Next.js web application.
3.  **Database**: PostgreSQL database.
4.  **File storage**: Previously using MinIO, but we’ll migrate to Azure Blob Storage, stay tuned for next phases. 

Here is a step-by-step guide to deploying it using Azure DevOps.

---

### **Phase 1: Prerequisites**

This phase is about getting Azure and DevOps environments ready.

**1. Set up Azure DevOps:**
*   Create a new **Azure DevOps Project**. This will be the central hub for code, pipelines, and work tracking.

**2. Set up Source Control:**
*   Push the entire codebase into an **Azure Repos Git repository** within new project. This is important because the pipelines will trigger automatically whenever we push new code, for continuous integration processes.

**3. Prepare Azure Subscription:**
*   Ensuring to have an active Azure subscription since we needed permissions to create and manage resources.

---

### **Phase 2: Provisioning Azure Infrastructure**

Next, we'll create the cloud services in Azure that will host AI4KM application. The best practice is to use managed "Platform as a Service" (PaaS) services, as they reduce manual management.

**1. Deploy the Database:**
*   **Service Choice**: **Azure Database for PostgreSQL**.
*   **Why**: It's a fully managed database service. Azure handles backups, patching, and scaling, so we can focus on our application.
*   **Action**: Create an instance of this service. During setup, we defined a server name, admin login, and password. We made sure to configure its **firewall rules** to allow connections from our future backend service. 

**2. Deploy the Backend (FastAPI):**
*   **Service Choice**: Use **Azure App Service for Containers**.
*   **Why**: Containerizing our Backend is the most robust and portable deployment method. Azure App Service can run Docker containers directly.
*   **Action**:
    *   Create an **Azure Container Registry (ACR)** to store our backend's Docker image.
    *   Create an **Azure App Service** configured to run a Linux container. We later pointed it to the image in our ACR.

**3. Deploy the Frontend (Next.js):**
*   **Service Choice**: Use **Azure Static Web Apps**.
*   **Why**: This service is made for modern JavaScript frameworks like Next.js. It automatically handles the build process (installations and running commands like for our case : `”npm run dev”). It's very cost-effective.
*   **Action**: Create an instance of Azure Static Web Apps. During setup, we linked it directly to our Azure Repo.

**4. Secure Secrets:**
*   **Service Choice**: Use **Azure Key Vault**.
*   **Why**: It isn’t a good practice to hard-code secrets like database connection strings or JWT keys in code. Key Vault provides a secure place to store them.
*   **Action**: Create a Key Vault and add the database connection string and `SECRET_KEY` as secrets. The Azure App Service can then be configured to securely access them at runtime without them ever being exposed in code.

To secure these secrets, there are two potential options : 
-	Either using Azure Key Vault, or configuring these environment variables. 

---

### **Phase 3: Building the CI/CD Pipelines**

This is the core of automation in Azure DevOps. We created two separate pipelines defined in YAML files that live alongside the code.

**1. Backend CI/CD Pipeline (`azure-pipelines-backend.yml`):**
This pipeline will be responsible for containerizing and deploying the FastAPI application.

*   **Trigger**: It will automatically run whenever a change is pushed to the Master branch
*   **Build Stage**:
    1.  The pipeline job will check out the code.
    2.  It builds a Docker image from the backend's `Dockerfile`.
    3.  It tags the image with ‘lastest’ tag.
    4.  It pushes the newly built image to **Azure Container Registry (ACR)**.
*   **Deploy Stage**:
    1.  This stage runs after the build is successful.
    2.  It connects to **Azure App Service**.
    3.  It tells the App Service to pull the new image from ACR and restart, deploying the new version with latest tag.
    4.  It will also configure the App Service's application settings, pulling the database connection string from **Azure Key Vault**.

**2. Frontend CI/CD Pipeline (`azure-pipelines-frontend.yml`):**
This pipeline will build and deploy Next.js application.

*   **How it's created**: This is the easiest part. When we create the **Azure Static Web App** in the Azure portal and link it to our repository, Azure will **automatically generate and commit** the complete YAML pipeline file for we.
*   **What it does**:
    1.  **Triggers** automatically on any change to the `front_km_ai_logo_title/` directory.
    2.  **Builds** the Next.js application (runs `npm run dev`).
    3.  **Deploys** the resulting static files to the global network of the Static Web Apps service.

---

### **Phase 4: Final Configuration and Going-Live !**

These are the final steps to connect everything and make the application accessible to the world.

**1. Configure Environment Variables:**
*   In the Azure portal, for **Frontend** (Static Web App), we added an application setting for `NEXT_PUBLIC_API_URL`. Set its value to the public URL of the **Backend** (App Service).
*   The same for `NEXT_PUBLIC_WS_API_URL`, using the `ws://` or `wss://` protocol.

**2. Configure CORS:**
*   In the **Backend** App Service, we configured its CORS (Cross-Origin Resource Sharing) settings to explicitly allow requests from the frontend's URL. This is a critical security step that allows the browser to make API calls from frontend to backend.

**3. Test the Deployed Application:**
*   Navigate to the URL of the Azure Static Web App. We should be able to sign up, log in, and use the chat, with all API and WebSocket calls correctly routing to our backend service running in Azure.

This step-by-step plan provides a modern, secure, and greatly automated way to deploy AI4KM application. 

---

#  Detailed Plan for every phase of the deployment strategy

## **Phase 1: Prerequisites**


Here is a detailed, beginner-friendly guide to get the codebase from local machine into Azure Repos.

#### **Prerequisites** : 

**Git Installed**: We need Git installed on our Windows machine. If we don't have it, we can download it from [git-scm.com](https://git-scm.com/). The default installation options are fine.

---

### **Step 1: Create a New Project in Azure DevOps**

First, we need a project in Azure DevOps to hold our repository.

1.  Go to [dev.azure.com](https://dev.azure.com) and sign in.
2.  Click the **"+ New project"** button in the top-right corner.
3.  Give our project a **Project name** (e.g., `ai4km-app`).
4.  Keep the visibility as **Private**. This is the standard for company or personal projects.
5.  Click **"Create"**.

We will be taken to our new project's dashboard.

### **Step 2: Get the Repository URL**

Now, let's get the address of the empty Git repository that Azure DevOps just created for we.

1.  In our project's left-hand menu, click on **"Repos"**.
2.  Since the repo is empty, we'll see a page with instructions. Find the section titled **"Push an existing repository from command line"**.
3.  We will see two commands there. The important part is the URL, which looks like this:
    `https://<OurOrgName>@dev.azure.com/<OurOrgName>/<OurProjectName>/_git/<OurProjectName>`
4.  Click the **copy button** next to this URL. Keep it handy; we'll need it in a moment.



### **Step 3: Prepare Our Local Codebase**

Next, we'll get the local code ready to be pushed. We'll do this from the command line.

1.  Open a terminal. Use **Git Bash** (which comes with Git) or the Windows **Command Prompt**.
2.  Navigate to the root directory of our project:
    ```bash
    cd path\to\full\code
    ```
3.  Initialize a Git repository if we haven't already. Run `git status`. If it says `fatal: not a git repository`, then run this command to create one:
    ```bash
    git init
    ```
4.  **Crucial Step: Create a [.gitignore].** This file tells Git to ignore certain files and folders (like `node_modules`, Python virtual environments, and local secrets) so they aren't accidentally committed to the repository.
    *   Create a new file named [.gitignore] in the root directory.
    *   Copy and paste the content below into it. This is a good starting point for a Python/Node.js project.

    ```gitignore
    # Python
    __pycache__/
    *.pyc
    .env
    venv/
    *.egg-info/

    # Node.js
    node_modules/
    .next/
    npm-debug.log*
    yarn-debug.log*
    yarn-error.log*
    .pnp/
    .pnp.js

    # IDE / OS files
    .vscode/
    .idea/
    *.DS_Store
    ```

5.  Now, stage and commit all our code to our local repository:
    ```bash
    git add .
    git commit -m "Initial commit of the AI4KM application"
    ```

### **Step 4: Connect and Push to Azure Repos**

This is the final step where we connect our local repo to the one in Azure and upload our files.

1.  In our terminal (still in our project directory), run the `git remote add` command we saw on the Azure Repos page. It connects our local repo to the remote one in Azure.
    ```bash
    git remote add origin https://<OurOrgName>@dev.azure.com/<OurOrgName>/<OurProjectName>/_git/<OurProjectName>

    ```
2.  Now, push our code. The `-u` flag sets the Azure repo as the default for future pushes.
    ```bash
    git push -u origin --all
    ```

### **Step 5: Authenticate with Azure DevOps**

When we run the `git push` command, Git will need to prove to Azure that we have permission to add code. A window will likely pop up asking we to sign in.

*   **The Easiest Way (Recommended)**: Use the **Git Credential Manager**. A Microsoft sign-in window should appear. Just sign in with the same account we use for Azure DevOps.
*   **Alternative (Personal Access Token)**: If we are prompted for a password in the terminal, we need to generate a **Personal Access Token (PAT)**.
    *   In Azure DevOps, go to **User settings** (top right corner) -> **Personal Access Tokens**.
    *   Create a new token. Give it a name, set the expiration, and for **Scopes**, select **"Code (Full)"**.
    *   Copy the generated token. **We will only see it once!**
    *   Go back to the terminal. When it asks for a password, paste the PAT.

### **Step 6: Verify Our Success!**

Go back to our browser and refresh the **Repos** page in Azure DevOps. We should now see all our project files!

The code is now securely stored and versioned in Azure Repos, let’s dive into the next phase.


---

## **Phase 2: Provisioning Azure Infrastructure**

### **Step 1: Creating the Production Database in Azure**

Let's start with the most critical piece: the database. Here is the
guide to creating the **Azure Database for PostgreSQL** instance using the Azure Portal.

**1. Go to the Azure Portal**
*   Open our web browser and navigate to [portal.azure.com](https://portal.azure.com). Sign in with our account.

**2. Find the PostgreSQL Service**
*   In the main search bar at the top of the portal, type **`Azure Database for PostgreSQL`** and select it from the list.

**3. Start the Creation Process**
*   On the service page, click the **"+ Create"** button.
*   We will be presented with different options. Choose **"Flexible Server"** and click **"Create"**. The Flexible Server model is the most modern and recommended option.

**4. Configure Our New Database Server**
We will now see a form with several sections. Let's go through the most important ones:

*   **Basics Tab:**
    *   **Subscription:** Choose our Azure subscription.
    *   **Resource group:** Click **"Create new"** and give it a name like `ai4km-database` (for "production resource group"). A resource group is like a folder that holds all the related services for our application.
    *   **Server name:** Pick a **globally unique** name for our database (e.g., `ai-km-prod-db-server-` followed by our initials).
    *   **Region:** Select an Azure region that is physically close to we or our users (e.g., `East US`, `West Europe`).
    *   **PostgreSQL version:** Choose the latest available version (e.g., 16).
    *   **Workload type:** Select **"Development"** for now. This will give we a low-cost server that is perfect for getting started. We can easily scale it up later without any downtime.
    *   **Admin account:** Set our **Administrator username** and **Password**. **IMPORTANT: Save this password somewhere safe like a password manager. We will need it to connect to our database!**

*   **Networking Tab:**
    *   **Connectivity method:** Select **"Public access"**. This means the database will be accessible over the internet (but protected by a firewall).
    *   **Firewall rules:** Click **"Add current client IP address"**. This adds a rule that allows our current computer to connect to the database, which is essential for the initial setup.

**5. Create the Server**
*   Click the **"Review + create"** button at the bottom, and then click **"Create"** again.
*   Azure will now start provisioning our database server. This can take 5-10 minutes. We can watch the progress in the notifications area (the bell icon in the top right).

Once the deployment is complete, we will have a production-ready PostgreSQL server running in Azure. We will then need to connect to it to enable the `pgvector` extension, but creating the server is the biggest step.

The production database server is running. Now, we need to perform a crucial one-time setup: enabling the `pgvector` extension.

In Azure, this is a two-part process:
1.  **Allowlist the Extension**: We first need to tell our Azure PostgreSQL server that it's *allowed* to use the `pgvector` extension. This is a security measure.
2.  **Enable the Extension**: We then connect to the database with a standard tool and run a simple SQL command to actually turn it on for our specific database, for example, we used pgAdmin for visualizing and querying the database.

Let's go through this step-by-step.

---

### **Step 1: Allowlist the `vector` Extension in the Azure Portal**

1.  **Navigate to Our Server**: Open the [Azure Portal](https://portal.azure.com) and go to the PostgreSQL server we just created.
2.  **Find Server Parameters**: In the left-hand menu for our server, scroll down to the **Settings** section and click on **"Server parameters"**.
3.  **Search for the Extension Parameter**: In the search box on the Server parameters page, type `azure.extensions`. This will filter the list to show the correct parameter.
4.  **Add `VECTOR` to the Allowlist**:
    *   We will see a parameter named `azure.extensions`.
    *   Click inside the `VALUE` box. A dropdown list of available extensions will appear.
    *   Scroll through the list and check the box next to **VECTOR**.
    *   Click **"OK"** to close the dropdown. The `VALUE` box should now contain `VECTOR`.
5.  **Save Our Changes**: At the top of the page, click the **"Save"** icon. It may take a minute or two for the change to apply.

We have now given our server permission to use the `pgvector` extension.

---

### **Step 2: Connect and Enable the Extension with a Database Tool**

Now we need to connect to the database and run the final command. The best tool for this if we're a beginner is **pgAdmin**, a free graphical tool for managing PostgreSQL.

**1. Get Our Connection Details from Azure**
*   Go back to the **"Overview"** page for our PostgreSQL server in the Azure Portal.
*   On this page, we will find the following essential details. Keep this page open:
    *   **Server name** (this is the Host)
    *   **Admin username**

**2. Download and Install pgAdmin**
*   If we don't have it, download it from the official website: [pgAdmin Download](https://www.pgadmin.org/download/).
*   Install it on our computer just like any other application.

**3. Connect pgAdmin to Our Azure Database**
*   Open pgAdmin.
*   Right-click on **"Servers"** in the left-hand browser panel and select **"Register"** -> **"Server..."**.
*   A dialog box will appear. Fill it out as follows:
    *   **General Tab:**
        *   **Name:** Give our connection a friendly name, like `Azure Prod DB`.
    *   **Connection Tab:**
        *   **Host name/address:** Copy the **Server name** from the Azure Portal.
        *   **Port:** Leave this as `5432`.
        *   **Maintenance database:** Leave this as `postgres`.
        *   **Username:** Copy our **Admin username** from the Azure Portal.
        *   **Password:** Enter the password we created when we set up the server.
    *   **SSL Tab:**
        *   **SSL mode:** Change this from `Prefer` to **`Require`**. This is mandatory for connecting to Azure.
*   Click **"Save"**. If all the details are correct, pgAdmin will connect to our server and it will appear in the left-hand panel.

**4. Create the Database and Enable the Extension**
*   In pgAdmin, expand our new server connection. We will see a database named `postgres`.
*   Right-click on **"Databases"** and select **"Create"** -> **"Database..."**.
*   In the dialog, for **Database name**, enter `km`. 
*   Click **"Save"**.
*   Now, find our new `km` database in the list, left-click on it, and then click the **"Query Tool"** icon in the top menu bar (it looks like a database with a magnifying glass).    ```


### Step 2 : Frontend deployment through Azure Static Web Apps 

Now, let's get the frontend deployed. We will use **Azure Static Web Apps**, a service that is perfectly designed for modern JavaScript applications like ours (which is built with Next.js).

### **Why Azure Static Web Apps?**

Think of this service as an all-in-one solution for our frontend. When we use it, Azure automatically handles:
*   **Building the Code**: It runs the `npm run build` command for we in the cloud.
*   **Global Distribution**: It places our app on a Content Delivery Network (CDN), so it loads quickly for users anywhere in the world.
*   **Free SSL Certificate**: It automatically secures our site with HTTPS.
*   **CI/CD Automation**: It creates a deployment pipeline for we in Azure DevOps. From now on, whenever we push a change to our frontend code, the app will automatically rebuild and deploy the new version.

---

### **Step-by-Step: Deploying Our Frontend to Azure Static Web Apps**

Here is our beginner-friendly guide to creating the service and deploying our app.

**1. Go to the Azure Portal**
*   Open our web browser and navigate to [portal.azure.com](https://portal.azure.com).

**2. Find the Static Web Apps Service**
*   In the main search bar at the top, type **`Static Web Apps`** and select it from the list.

**3. Start the Creation Process**
*   On the service page, click the **"+ Create"** button.

**4. Configure Our New Static Web App**
We will now see a form with several tabs. This is the most important part. Let's fill it out carefully.

*   **Basics Tab:**
    *   **Subscription:** Choose our Azure subscription.
    *   **Resource group:** Select the **same resource group** we created for our database (e.g., `ai-km-prod-rg`). This keeps all our application's components neatly organized together.
    *   **Name:** Pick a **globally unique** name for our application. This name becomes part of its URL (e.g., `ai-km-prod-app`).
    *   **Plan type:** Select **"Free"**. This is more than enough to get started.
    *   **Region:** Choose a region. For the "Free" plan, this mainly affects where any associated backend functions would run. Pick a region close to we.

*   **Deployment details Tab:**
    *   **Source:** Select **"Azure DevOps"**.
    *   A new section will appear. Fill in the details of our repository:
        *   **Organization:** Select our Azure DevOps organization.
        *   **Project:** Select the project we created (e.g., `ai-km-app`).
        *   **Repository:** Select our repository.
        *   **Branch:** Select `main`.

*   **Build Details Tab:**
    *   This tells Azure *how* to build our Next.js app.
    *   **Build Presets:** From the dropdown, select **"Next.js"**. Azure will automatically fill in the best settings for a Next.js project.
    *   The fields should now look like this:
        *   **App location:** `/` (This needs to be corrected).
        *   **Api location:** `api` (or blank).
        *   **Output location:** `.next`
    *   **CRITICAL CORRECTION:** We must change the **App location**. Our frontend code is not in the root of our repository; it's in a subfolder. Change this value to:
        *   **App location:** `/front_km_ai_logo_title`

**5. Review and Create**
*   Click the **"Review + create"** button at the bottom.
*   Verify that all the settings are correct, especially the **App location**.
*   Click **"Create"**.


Once we click "Create," two things will happen automatically:

1.  **Resource Creation**: Azure will create the Static Web App resource. We can navigate to it in the portal. On its "Overview" page, we will see a URL for our application (it will show a placeholder page for now).

2.  **Pipeline Creation and Execution**: Azure will **automatically commit a new YAML pipeline file** to our Azure DevOps repository. This pipeline defines the build and deploy steps. It will then **immediately trigger its first run**.

### **Step 6: Monitor Our First Deployment**

1.  In the Azure Portal, go to our newly created Static Web App.
2.  In the overview panel, we will see a link to "Azure DevOps" or a banner that says "Click here to check the status of our Azure DevOps build." Click it.
3.  This will take we directly to the pipeline run in Azure DevOps. We can watch in real-time as it checks out our code, installs the `npm` packages, and builds our Next.js application.
4.  The first run might take 5-10 minutes. Once it finishes successfully (we'll see a green checkmark), our application will be live!

### **Step 7: Visit Our Live Application!**

Go back to the "Overview" page of our Static Web App in the Azure Portal and click on the URL. We should see our application's login page, now running live in the cloud.

**Note:** The login won't work yet because the deployed frontend doesn't know the address of our backend. That is the very next step we will tackle after we have this one.

### **For "Api location":**

*   **What we should put:** We should leave this field **blank**.
*   **Why:** The "Api location" field is specifically for a feature of Azure Static Web Apps called "Managed Functions." This feature allows we to deploy a simple, serverless API (written as Azure Functions) *alongside* our frontend, within the same Static Web App resource. Since we are correctly planning to deploy our powerful Python FastAPI backend as a completely separate service (an Azure App Service), we will not be using this integrated API feature. Leaving it blank tells Azure, "I am not deploying a managed API with this service."

### **For "Output location":**

*   **What we should put:** We should put **`.next`**.
*   **Why:** This setting tells the Azure build system where to find the final, compiled files after it finishes building our Next.js project. When we run the standard `npm run build` command on a Next.js application, it generates a highly optimized production version of our site and places all the necessary files into a folder named `.next`. Azure needs to know the name of this folder so it can grab the contents and deploy them to its global CDN.

### **Summary of Our Build Settings**

So, to be perfectly clear, our "Build Details" section should look like this:

*   **Build Presets:** `Next.js`
*   **App location:** `/front_km_ai_logo_title` (This is crucial because our app is in a subfolder)
*   **Api location:** (leave this empty)
*   **Output location:** `.next`

These settings will correctly configure the pipeline to:
1.  Navigate into our `front_km_ai_logo_title` directory.
2.  Run `npm run build`.
3.  Find the compiled output in the `.next` folder.
4.  Deploy those files to the web.
5.  Ignore the API part, because our real API will be deployed elsewhere.


## Step 3 : Backend Python FastAPI application

Now, let’s focus on the final critical piece: the backend. This is what will power the entire application.

### **The Strategy: Containerization**

The best and most modern way to deploy an application like AI4KM is by using **containers**. A container is a lightweight, standalone package that includes everything our application needs to run: the code, its dependencies (like FastAPI and SQLAlchemy).

**Why use containers?**
*   **Consistency:** The container runs exactly the same on our local machine as it does in the cloud, eliminating "it works on my machine" problems.
*   **Portability:** We can run this container on any major cloud provider or on our own servers.
*   **Isolation:** The container keeps our application and its dependencies separate from the underlying server, which is great for security and stability.

### **The Azure Services We Will Use**

To run our containerized backend in Azure, we need two core services:

1.  **Azure Container Registry (ACR):** Think of this as a private, secure library for our Docker images. We built our backend's container image and store it here.
2.  **Azure App Service for Containers:** This is the service that will actually *run* the container. It will pull the image from ACR and host it on the web, giving us a public URL for our API. It also handles scaling, security, and monitoring for we.

### **The Game Plan: Step-by-Step**

Let's outline the process from start to finish.

**Step 1: Review Backend's `Dockerfile`**

A `Dockerfile` is a text file that contains the step-by-step instructions for building the application's container image. The project must have one for the backend. 

---

### **Step 2: Create the `.dockerignore` File**

Now, let's create the `.dockerignore` file. This is just as important as the `Dockerfile`. It lists all the files and folders that should **not** be included in the container. This prevents secrets (like `.env` file) from being baked into the image and keeps the image size small.

### **Why the `.dockerignore` File is Important**

This file tells Docker to ignore things like:
*   Our Python virtual environment (`venv`).
*   Our secret `.env` file.
*   Git history and editor-specific folders.

This is critical for security (it prevents database password from being copied into the image) and for keeping container image small and efficient.

---

### **Next Step: Create a Home for Container Image**

Now we need a place to *store* the finished container image in the cloud. For this, we used the **Azure Container Registry (ACR)**.

Think of ACR as a personal and private, secure version of Docker Hub, hosted in Azure.

### **Step-by-Step: Creating Azure Container Registry**

**1. Go to the Azure Portal**
*   Open our web browser and navigate to [portal.azure.com](https://portal.azure.com).

**2. Find the Container Registry Service**
*   In the main search bar at the top, type **`Container Registries`** and select it from the list.

**3. Start the Creation Process**
*   On the service page, click the **"+ Create"** button.

**4. Configure Our New Registry**
We will now see a form. Let's fill it out.

*   **Basics Tab:**
    *   **Subscription:** Choose our Azure subscription.
    *   **Resource group:** Select the **same resource group** we had been using. This is very important for keeping the project organized.
    *   **Registry name:** This must be **globally unique** across all of Azure. Use something simple and related to our project.
        *   Example: `ai4kmcontainerregistry` (all lowercase, no hyphens).
    *   **Location:** This should automatically match our resource group's location.
    *   **SKU:** Select **`Basic`**. This is the most cost-effective tier and is perfect for our needs.
    *   **Admin user:** Make sure this is **enabled (checked)**. We will need this for an easy way to log in later.

**5. Review and Create**
*   Click the **"Review + create"** button.
*   Verify that all the settings are correct.
*   Click **"Create"**.

The deployment will take a few minutes. Once it's complete, we will have a secure, private registry in Azure, ready to store our backend container image.

We will proceed to the next step: building the image and pushing it to the new registry.

### **Step 1: Get Azure Container Registry (ACR) Credentials**

First, we need to get the "address" and "keys" to the new registry so Docker knows where to send the image and has permission to do so.

1.  Go to the **Azure Portal** ([portal.azure.com](https://portal.azure.com)).
2.  Navigate to the **Container Registry** resource we created (e.g., `ai4kmcontainerresgistry`).
3.  In the left-hand menu, under the "Settings" section, click on **"Access keys"**.
4.  We will see three important pieces of information. Keep this page open, as we will need to copy-paste from it:
    *   **Login server:** This is the address of our registry (e.g., `ai4kmcontainerregistry.azurecr.io`).
    *   **Username:** This is the name of our registry (e.g., `ai4kmcontainerregistry`).
    *   **password:** There are two passwords listed. We can use either one.

### **Step 2: Open a Terminal and Navigate to Our Backend Code**

Now, let's open a command line interface to run the Docker commands.

1.  Open **Command Prompt** or **PowerShell** on our Windows machine.
2.  Navigate to the directory where our `Dockerfile` is located by running this command:
    ```bash
    cd path\to\backend\code
    ```

### **Step 3: Build the Docker Image**

This command reads our `Dockerfile` and builds our application into a self-contained image.

*   Run the following command in our terminal. The `.` at the end is important, it tells Docker to look for the `Dockerfile` in the current directory.
    ```bash
    docker build -t ai4km-backend:v1 .
    ```
*   This process will take several minutes as Docker downloads the base image and installs all the dependencies from [requirements.txt] file. 

### **Step 4: Log in to Our Azure Container Registry**

Now we need to authenticate our local Docker client with our registry in Azure.

*   Run the following command, replacing `<our_login_server>` with the **Login server** value we found in Step 1.
    ```bash
    docker login <our_login_server>
    ```
*   It will prompt we for a **Username**. Enter the username from the "Access keys" page.
*   It will then prompt we for a **Password**. Copy one of the passwords from the "Access keys" page and paste it in.

We should see a "Login Succeeded" message.

### **Step 5: Tag the Image for Azure**

Our image is currently named `ai4km-backend:v1`, which is a local name. We need to give it an "alias" or "tag" that includes the address of our Azure registry.

*   Run the following command, making sure to replace `<our_login_server>` in both places.
    ```bash
    docker tag ai4km-backend:v1 <our_login_server>/ai4km-backend:v1
    ```

### **Step 6: Push the Image to Azure**

This is the final step, where we upload our tagged image from our computer to our private registry in the cloud.

*   Run the following command, again replacing `<our_login_server>`:
    ```bash
    docker push <our_login_server>/ai4km-backend:v1
    ```
*   This will also take a few minutes, depending on our internet connection speed, as it uploads the layers of our image.

### **Step 7: Verify the Upload**

1.  Go back to the **Azure Portal**.
2.  Navigate to our **Container Registry**.
3.  In the left-hand menu, under "Services," click on **"Repositories"**.
4.  We should now see a new repository named **`ai4km-backend`**. If we click on it, we will see the tag **`v1`**.

We have successfully containerized our backend application and stored it securely in the cloud. Our backend is now ready to be deployed and run. The final step is to create a service that will run this container image and make it accessible on the internet.

For this, we will use **Azure App Service**, which is Azure's primary platform for hosting web applications.

### **The Goal: Run our Container as a Live Web App**

Azure App Service will:
*   Pull the `ai4km-backend:v1` image from our Azure Container Registry.
*   Run it as a persistent, managed container.
*   Assign it a public URL (e.g., `https://ai4km-backend-api.azurewebsites.net`).
*   Handle networking, security, and scaling for we.

Let's get it running.

---

### **Step-by-Step: Deploying Backend with Azure App Service**

**1. Go to the Azure Portal**
*   Navigate to [portal.azure.com](https://portal.azure.com).

**2. Find the App Services Service**
*   In the main search bar at the top, type **`App Services`** and select it from the list.

**3. Start the Creation Process**
*   On the App Services page, click the **"+ Create"** button and select **"Web App"** from the dropdown.

**4. Configure Our New Web App (Basics Tab)**
This first tab is for the core details of our service.

*   **Subscription:** Choose our Azure subscription.
*   **Resource Group:** Select the **same resource group** we have been using (`ai4km-resource-group`).
*   **Name:** Pick a **globally unique** name for our backend API. This becomes part of its URL.
    *   Example: `ai4km-backend-api`
*   **Publish:** Select **`Docker Container`**. This is the key setting.
*   **Operating System:** Select **`Linux`**.
*   **Region:** This should automatically match our resource group's location.
*   **Linux Plan (App Service Plan):** This is the underlying virtual machine that will run our container. We need to create a new one.
    *   Click **"Create new"**.
    *   Give it a name, for example: `ai-km-prod-plan`.
    *   Click **"OK"**.
*   **Pricing plan:**
    *   The default may be a high-cost plan. Click **"Change size"** next to the plan name.
    *   Go to the **"Dev / Test"** tab and select the **`B1` (Basic)** tier. This is a great, cost-effective starting point. Click **"Apply"**.

**5. Configure the Container (Docker Tab)**
Now, click the **"Next: Docker >"** button at the bottom to tell the App Service where to find our image.

*   **Options:** Select `Single Container`.
*   **Image Source:** Select `Azure Container Registry`.
*   **Registry:** Use the dropdown to select the container registry we created (e.g., `ai4kmcontainerregistry`).
*   **Image:** Select the `ai4km-backend` image from the dropdown.
*   **Tag:** Select `v1`. (Attention !!! This worked us out a lot, we might choose a tag called `latest` for continuous deployment)
*   **Startup Command:** Leave this **blank**. Our `Dockerfile` already contains the `CMD ["uvicorn", ...]` instruction, and App Service will use that by default.

**6. Review and Create**
*   Click the **"Review + create"** button.
*   Carefully review all the settings to ensure they are correct.
*   Click **"Create"**.

### **What Happens Now?**

Azure is now doing several things in the background:
1.  Provisioning the server (the App Service Plan).
2.  Creating the Web App service to host our container.
3.  Connecting to our Azure Container Registry.
4.  Pulling the `ai4km-backend:v1` image. (Attention !! It should be `ai4km-backend:latest`, it can be reconfigured after deploying, so no problem)
5.  Starting the container.

This process will take about 5-10 minutes.

### **Important Next Step: Configuration**

Once the deployment is finished, we will have a URL for our backend. However, **the application will not work correctly yet.**

Why? Because the container is running in isolation and has no idea how to connect to our Azure PostgreSQL database. The code is expecting a `DATABASE_URL` or different components environment variables of the URL.

**As soon as the deployment is complete, our very next task will be to go into the App Service's "Configuration" settings and securely provide it with the database connection string.** This will bring the backend fully online.

---

### **Part 1: Connecting the Deployed Services**

Right now, the three services (Frontend, Backend, Database, file storage) are sitting in Azure, but they don't know about each other. We need to give them the addresses and credentials to connect. We will do this using **Configuration Settings**, which act as secure environment variables for our cloud services.

#### **Step 1: Connect the Backend to the Database**

Our backend container needs the `DATABASE_URL` to connect to our Azure PostgreSQL server.

1.  **Get Our Database Connection String**:
    *   We should have this from when we connected pgAdmin. If not, reconstruct it. The format is:
        `postgresql+psycopg2://USER:PASSWORD@SERVER_FQDN:5432/km?sslmode=require`
    *   Replace `USER`, `SERVER`, `PASSWORD`, and `SERVER_FQDN` with our actual Azure PostgreSQL credentials. **Be very careful with the password.**

2.  **Add the Connection String to Our Backend App Service**:
    *   In the **Azure Portal**, navigate to our backend **App Service** (e.g., `ai4km-backend-api`).
    *   In the left-hand menu, under "Settings," click on **"Configuration"**.
    *   Under the **"Application settings"** tab, click **"+ New application setting"**.
    *   **Name:** `DATABASE_URL`
    *   **Value:** Paste our full, complete database connection string here.
    *   Click **"OK"** to save the setting.
    *   At the top of the Configuration page, click **"Save"**. Azure will restart our backend container with the new environment variable.

#### **Step 2: Connect the Frontend to the Backend**

Our frontend Static Web App needs to know the URL of our newly deployed backend API.

1.  **Get Our Backend URL**:
    *   In the Azure Portal, navigate to our backend **App Service**.
    *   On the **"Overview"** page, we will find the **URL** (e.g., `https://ai4km-backend-api.azurewebsites.net`). Copy this.

2.  **Add the Backend URL to Our Frontend Static Web App**:
    *   In the Azure Portal, navigate to our **Static Web App** (e.g., `ai-km-prod-app`).
    *   In the left-hand menu, click on **"Configuration"**.
    *   Click **"+ Add"** to create a new setting.
        *   **Name:** `NEXT_PUBLIC_API_URL`
        *   **Value:** Paste the backend URL we just copied.
    *   Click **"+ Add"** again to add the WebSocket URL.
        *   **Name:** `NEXT_PUBLIC_WS_API_URL`
        *   **Value:** Take our backend URL, change `https://` to `wss://`, and add `/ws` to the end. For example: `wss://ai4km-backend-api.azurewebsites.net/ws`
    *   Click **"OK"** on both, and then click **"Save"** at the top.

#### **Step 3: Test Everything!**

Our services are now connected. Open the URL of our live frontend application. We should be able to:
1.  Register a new user.
2.  Log out.
3.  Log back in with the new user.

If we can do this, it means the entire chain is working: the browser is talking to the frontend, the frontend is making API calls to the backend, and the backend is successfully reading/writing to the database.

---

## Phase 3 : Building the Backend CI/CD Pipeline

This will automate the manual `docker build` and `docker push` steps we  did earlier.

#### **Step 1: Create a Service Connection in Azure DevOps**

First, we need to give our Azure DevOps project permission to connect to our Azure subscription. This is done via a "Service Connection."

1.  Go to our **Azure DevOps Project**.
2.  Click on **"Project settings"** in the bottom-left corner.
3.  Under the "Pipelines" section, click on **"Service connections"**.
4.  Click **"Create service connection"**.
5.  Select **"Azure Resource Manager"** and click **"Next"**.
6.  Select **"Service principal (automatic)"** and click **"Next"**.
7.  Choose our **Subscription** and give the service connection a name (e.g., `Azure-Subscription-Connection`). Ensure "Grant access permission to all pipelines" is checked.
8.  Click **"Save"**.

#### **Step 2: Create the Backend Pipeline YAML File**

### **Explanation of the Backend Pipeline**

*   **`trigger`**: This pipeline will automatically run only when we push changes to the [pipeline] directory. This is efficient, since it won't rebuild our backend if we only change the frontend.
*   **`variables`**: This section holds all the important names for our services. **We must edit this file** and replace the placeholder values for `containerRegistry` and `webAppName` with our actual service names. The `azureSubscription` name must also exactly match the service connection we created.
*   **`job: BuildAndPush`**: This first job connects to our Azure Container Registry, builds a new Docker image from our `Dockerfile`, and pushes it to the registry with two tags: a unique build ID and "latest".
*   **`job: DeployToAppService`**: This second job only runs if the first one succeeds. It connects to our Azure subscription, finds our App Service, and tells it to pull the newly created image from our registry and restart.

### **Final Step: Create the Pipeline in Azure DevOps**

Now that the YAML file exists in our repository, we just need to tell Azure DevOps to create a pipeline from it.

1.  Go to our **Azure DevOps Project**.
2.  In the left-hand menu, click on **"Pipelines"**.
3.  Click the **"New pipeline"** button in the top right.
4.  Select **"Azure Repos Git"** (where our code is).
5.  Select our repository.
6.  Select **"Existing Azure Pipelines YAML file"**.
7.  In the "Path" dropdown, select `/azure-pipelines-backend.yml`.
8.  Click **"Continue"**.
9.  **DO NOT CLICK RUN YET.** Click the down arrow next to the "Run" button and select **"Save"**.

We now have two CI/CD pipelines: one for our frontend (created automatically by Static Web Apps) and one for our backend.

From this point forward, whenever we push a code change to the [pipeline] directory, this new pipeline will automatically build and deploy our updated backend to Azure.


