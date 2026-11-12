# Google Cloud Practical Learning Handbook

## 1. Google Cloud Projects

Think of a [Google Cloud project](https://cloud.google.com/compute/docs/overview) as a main workspace or top-level folder in Google Cloud. Every resource you create—such as virtual machines (VMs), databases, [Cloud Storage](https://cloud.google.com/storage/docs) buckets, and networks—must live inside a project.

A project controls four main things:

* **Resources**: All the VMs, disks, and buckets inside it.

* **Billing**: Which payment account pays for those resources.

* **Permissions (IAM)**: Who is allowed to view or change things inside the project.

* **Enabled APIs**: Which Google Cloud services are turned on inside the project.

### 1.1 Three Ways Google Cloud Identifies a Project

Every project has three different labels. Knowing the difference saves you from errors when running commands:

| Identifier | What It Means in Simple Words | Can You Change It Later? | Example | 
| ----- | ----- | ----- | ----- | 
| **Project name** | A friendly display name you read in the web console ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). Two projects can have the same name. | Yes, anytime. | `My Development Project` | 
| **Project ID** | A unique ID you type in `gcloud` commands and code ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). No two projects in the world can share the same Project ID. | No. Once created, it is permanent. | `my-project-123` | 
| **Project number** | A number that Google Cloud creates automatically behind the scenes for internal tracking ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | No. Google Cloud sets it permanently. | `123456789012` | 

### 1.2 Useful `gcloud` Commands for Projects

```
# Check which project you are currently working in
gcloud config get-value project

# See a list of all projects your account can open
gcloud projects list

# Switch to a specific project (always use the Project ID here, not the Project Name)
gcloud config set project my-project-123

# Show all three identifiers (Name, ID, and Number) for a project
gcloud projects describe my-project-123

```

### 1.3 Real-World Example: Why Create Multiple Projects?

Suppose you are building an employee policy chatbot for your company. Instead of putting test work and live company data into the same project, you create three separate projects:

* **`policy-bot-dev`**: A safe playground where developers write and test new code.

* **`policy-bot-test`**: A testing environment where QA checks the app before release.

* **`policy-bot-prod`**: The live environment used by real employees, locked down with strict security rules.

Keeping these in separate projects ensures that a mistake in `policy-bot-dev` can never accidentally delete live data in `policy-bot-prod`.

## 2. Identity and Access Management (IAM)

Google Cloud **IAM (Identity and Access Management)** is the security system that decides **who** is allowed to do **what** on **which resource**.

### 2.1 The Three Parts of an IAM Rule

Every access rule (called an **IAM policy binding**) connects three things:

**Principal (Who) → Role (What they can do) → Resource (Where they can do it)**

| Part | Simple Explanation | Example | 
| ----- | ----- | ----- | 
| **Principal** | The person or app asking for access ([OS Login](https://cloud.google.com/compute/docs/oslogin)). | `user:developer@example.com` | 
| **Role** | A bundle of permissions that allows specific actions ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | `Compute Instance Admin` (can create, start, and stop VMs) | 
| **Resource** | The project, storage bucket, or VM where that access is given ([Cloud Storage documentation](https://cloud.google.com/storage/docs)). | Project `policy-bot-dev` | 

### 2.2 Three Types of IAM Roles

| Role Type | Simple Explanation | When to Use It | 
| ----- | ----- | ----- | 
| **Basic roles** | Very broad roles (`Owner`, `Editor`, `Viewer`) that affect the entire project ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | Use only for quick practice or read-only checks (`Viewer`). Avoid `Owner` and `Editor` in real projects because they give too much power. | 
| **Predefined roles** | Ready-made roles created by Google for one specific job (for example, only reading files from a storage bucket) ([Cloud Storage documentation](https://cloud.google.com/storage/docs)). | Use these most of the time in real projects. | 
| **Custom roles** | Roles you build yourself by hand-picking exact permissions ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | Use when none of Google's predefined roles fit your exact security rule. | 

### 2.3 The Principle of Least Privilege

**Least privilege** simply means: *give people and apps only the exact permissions they need to do their job, and nothing extra.*

* **Bad example**: Giving a developer the `Project Owner` role when they only need to check if a VM is running.

* **Good example**: Giving that developer the `Compute Viewer` role so they can look at the VM without accidentally deleting it or changing billing.

### 2.4 What Is a Service Account?

Normal user accounts (`developer@example.com`) are for human beings. A **Service Account** is an account made for a **machine or application**—such as a [Compute Engine](https://cloud.google.com/compute/docs/overview) VM or a Cloud Run app.

For example, if your app running on a VM needs to download a file from a private [Cloud Storage](https://cloud.google.com/storage/docs) bucket, you do not put your personal Google password on the VM. Instead, you create a Service Account, give that Service Account permission to read the bucket, and attach the Service Account to the VM.

### 2.5 Practical IAM & Service Account Commands

Permissions flow downward (**Organization → Folder → Project → Resource**). If you give someone a role on a **Project**, they get that access on every VM and bucket inside the project. If you give a role on a single **Bucket**, they only get access to that one bucket.

```
# 1. See who currently has access to your project
gcloud projects get-iam-policy PROJECT_ID

# 2. Give a developer read-only (Viewer) access to the project
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:developer@example.com" \
  --role="roles/viewer"

# 3. Create a Service Account for an application or VM
gcloud iam service-accounts create policy-bot-sa \
  --display-name="Policy Bot Service Account"

# 4. List all Service Accounts in your project
gcloud iam service-accounts list

# 5. Allow that Service Account to read files from one specific storage bucket
gcloud storage buckets add-iam-policy-binding gs://YOUR_BUCKET_NAME \
  --member="serviceAccount:policy-bot-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

```

## 3. Google Cloud APIs

An **API (Application Programming Interface)** is the control switch behind every Google Cloud service. Whether you click a button in the web console, run a `gcloud` command, or write Python or .NET code, your request talks to a [Google Cloud API](https://cloud.google.com/compute/docs/overview):

* **Compute Engine API (`compute.googleapis.com`)**: Creates and manages virtual machines and load balancers ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)).

* **Cloud Storage API (`storage.googleapis.com`)**: Manages storage buckets and files ([Cloud Storage documentation](https://cloud.google.com/storage/docs)).

* **Cloud Pub/Sub API (`pubsub.googleapis.com`)**: Sends and receives messages between apps.

* **Cloud Run Admin API (`run.googleapis.com`)**: Runs containerized apps.

### 3.1 Why Do You Need to Enable APIs?

In a brand-new project, most APIs are **turned off** by default so you do not accidentally start services or get billed for things you aren't using. Before you can create a VM, you must turn on the Compute Engine API.

### 3.2 Important Difference: Turning on an API vs. IAM Permissions

Beginners often wonder why they get a "Permission Denied" error even after enabling an API. Both of these steps are required:

| Step | What It Does in Simple Words | Who It Affects | Command | 
| ----- | ----- | ----- | ----- | 
| **1. API Enablement** | Flips the master power switch **ON** for that service inside the project ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | The whole project | `gcloud services enable compute.googleapis.com` | 
| **2. IAM Permission** | Gives a specific person or Service Account a badge allowing them to use that service ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | One user or Service Account | `gcloud projects add-iam-policy-binding ...` | 

Also remember: turning on an API is free by itself, but creating resources with that API (like starting a VM) will cost money.

### 3.3 Practical Commands for APIs

```
# See which APIs are currently turned ON in your project
gcloud services list --enabled

# See all APIs available in Google Cloud
gcloud services list --available

# Turn on Compute Engine, Cloud Storage, and Pub/Sub APIs at the same time
gcloud services enable \
  compute.googleapis.com \
  storage.googleapis.com \
  pubsub.googleapis.com

# Check if the Compute Engine API is turned on
gcloud services list --enabled \
  --filter="name:compute.googleapis.com"

```

## 4. Cloud Pub/Sub

**Cloud Pub/Sub** (Publish/Subscribe) is a messaging service that lets different parts of an application talk to each other **without waiting on each other**.

Think of it like a post office drop box:

* Without Pub/Sub: Your frontend app calls the backend directly and freezes while waiting for a slow job to finish. If the backend is down, the request fails.

* With Pub/Sub: Your frontend drops a message into a **Topic** and immediately moves on. The backend picks up the message from a **Subscription** whenever it is ready.

### 4.1 The Six Main Parts of Pub/Sub

| Component | Simple Explanation | Chatbot Example | 
| ----- | ----- | ----- | 
| **Publisher** | The app that sends the message ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | The employee policy chatbot frontend. | 
| **Topic** | The named mailbox where the publisher drops messages. | `employee-events` | 
| **Subscription** | A waiting line attached to a topic that holds messages for a specific worker app. | `employee-events-sub` | 
| **Subscriber** | The worker app that picks up the message and does the background work. | A background service that saves audit logs. | 
| **Message** | The actual text or JSON data sent by the publisher. | `"Employee policy request submitted"` | 
| **Acknowledgment (ACK)** | A "thumbs up" reply from the subscriber telling Pub/Sub: *"I finished the job—you can delete this message from the queue now."* | Sent after the worker saves the log. | 

### 4.2 Step-by-Step: How a Message Travels Through Pub/Sub

1. **Send**: A **Publisher** sends a message to a **Topic**.

2. **Save**: Pub/Sub safely stores the message on disk and forwards it to every **Subscription** attached to that topic.

3. **Receive**: A **Subscriber** app receives the message from its subscription and performs the task.

4. **Confirm (ACK)**: The subscriber sends back an **ACK**. If the subscriber crashes before sending an ACK, Pub/Sub waits for the timeout and sends the message again so no work is lost.

### 4.3 Pull vs. Push Delivery

How does the subscriber get the message from the subscription? There are two ways:

| Feature | Pull Subscription (Subscriber Asks) | Push Subscription (Pub/Sub Sends) | 
| ----- | ----- | ----- | 
| **How it works** | The subscriber app reaches out and asks Pub/Sub: *"Do you have any messages for me right now?"* ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)) | Pub/Sub automatically sends the message straight to your app's HTTPS web address (`POST` request). | 
| **Who controls the speed?** | The subscriber app pulls messages only when it has free CPU capacity. | Pub/Sub pushes messages as they arrive. | 
| **Best used for** | Background workers running on [Compute Engine](https://cloud.google.com/compute/docs/overview) VMs that process heavy batches at their own pace. | Web services or Cloud Run apps that already listen for HTTPS requests. | 

### 4.4 Practical Pub/Sub Commands

```
# 1. Create a topic (the mailbox)
gcloud pubsub topics create employee-events

# 2. Create a pull subscription connected to that topic (the waiting line)
gcloud pubsub subscriptions create employee-events-sub \
  --topic=employee-events

# 3. Publish (send) a message to the topic
gcloud pubsub topics publish employee-events \
  --message="Employee policy request submitted"

# 4. Pull (read) the message and automatically send an ACK so it is marked finished
gcloud pubsub subscriptions pull employee-events-sub \
  --auto-ack

```

### 4.5 Golden Rule for Pub/Sub: Handle Duplicate Messages Safely

By default, Pub/Sub promises **at-least-once delivery**. That means Pub/Sub will never lose your message, but occasionally—if a network delay happens right when your worker sends its ACK—Pub/Sub might deliver the **exact same message twice**.

Always write your subscriber code so that receiving the same message twice does not cause harm (for example, check if `order_id = 501` was already saved before inserting it again). If a broken message keeps crashing your worker, configure a **Dead-Letter Topic** so Pub/Sub moves the bad message to a separate holding topic after a few retries.

## 5. Google Cloud Compute Engine — Practical Guide

[Compute Engine](https://cloud.google.com/compute/docs/overview) lets you create and run Virtual Machines (VMs)—which are complete Linux or Windows computers running inside Google's data centers.

### 5.1 What Is a VM and When Should You Use One?

A VM works just like a physical computer server. It has:

* **vCPUs and Memory (Machine Type)**: How much processing power and RAM it gets.

* **A Boot Disk**: The main hard drive holding the operating system (such as Debian Linux).

* **Extra Data Disks (Optional)**: Extra hard drives you attach to store application files.

* **Networking**: An internal IP address (and optional public IP address) plus firewall rules.

* **A Service Account**: The VM's identity for talking to other Google Cloud services.

* **A Startup Script (Optional)**: A script of commands the VM runs automatically every time it turns on.

**When should you use Compute Engine?**
Use a VM when you need full control over the Linux or Windows operating system, need to attach custom block disks, or want to install specific server software yourself ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). If you just want to run a simple web API without managing Linux updates or disks, a serverless tool like **Cloud Run** is much less work.

#### Creating and Managing a Basic VM

```
# Create a Debian 12 Linux VM named 'my-vm' in the asia-east1-a zone
gcloud compute instances create my-vm \
  --zone=asia-east1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud

```

| Part of the Command | What It Means in Simple Words | 
| ----- | ----- | 
| `gcloud compute instances create my-vm` | Tells [Compute Engine](https://cloud.google.com/compute/docs/overview) to build a new VM named `my-vm`. | 
| `--zone=asia-east1-a` | Places the VM in the `asia-east1-a` data center ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | 
| `--machine-type=e2-medium` | Chooses the computer size (2 shared vCPUs and 4 GB RAM) ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | 
| `--image-family=debian-12` | Installs the latest Debian 12 Linux operating system ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | 
| `--image-project=debian-cloud` | Tells Google to grab the official Debian image maintained by Google ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | 

```
# List all VMs in your project
gcloud compute instances list

# See full details (IP addresses, disks, status) for one VM
gcloud compute instances describe my-vm --zone=asia-east1-a

# Stop (shut down) a VM
gcloud compute instances stop my-vm --zone=asia-east1-a

# Start a stopped VM back up
gcloud compute instances start my-vm --zone=asia-east1-a

# Permanently delete a VM
gcloud compute instances delete my-vm --zone=asia-east1-a

```

> **Good to Know (Stopping vs. Deleting):** When you **stop** a VM, you stop paying for its CPU and RAM, and you can start it again later—but you **still pay** a small storage charge for its attached hard disk. When you **delete** a VM, the VM is gone for good.

### 5.2 Terminal vs. SSH vs. Cloud Shell (What Is the Difference?)

One of the easiest mistakes in a lab is typing a command into the wrong window. Let's make these three terms crystal clear:

* **What is a Terminal?** A text window where you type commands (like `ls` to list files, `pwd` to see what folder you are in, or `whoami` to see your username) instead of clicking with a mouse.

* **What is SSH (Secure Shell)?** [SSH](https://cloud.google.com/compute/docs/connect/standard-ssh) is a secure tunnel that connects your terminal to **another computer** far away. Once you connect to a VM with SSH, every command you type runs **inside that VM**, not on your laptop.

* **What is Cloud Shell?** Cloud Shell is a free mini-computer Google gives you inside your web browser with `gcloud` already installed ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). You use Cloud Shell as your "control room" to manage Google Cloud, and then use `gcloud compute ssh` from Cloud Shell to step inside your VM.

| Question | Cloud Shell | VM SSH Terminal | 
| ----- | ----- | ----- | 
| **What computer am I typing on?** | Google's temporary admin box (comes with 5 GB of personal storage) ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | Your actual [Compute Engine](https://cloud.google.com/compute/docs/overview) VM (`my-vm` or `gcelab`). | 
| **What should I run here?** | `gcloud` commands to create VMs, disks, buckets, and firewall rules ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | Linux commands (`sudo apt-get install nginx`, `lsblk`, `systemctl`) to set up the server inside the VM ([Connect to Linux VMs using SSH](https://cloud.google.com/compute/docs/connect/standard-ssh)). | 
| **What happens if I install NGINX here?** | It installs inside Cloud Shell by mistake (your VM won't have NGINX). | It installs NGINX on your VM where your web server actually runs. | 
| **How do I switch between them?** | Type `gcloud compute ssh VM_NAME` to jump into the VM. | Type `exit` to leave the VM and return to Cloud Shell. | 

#### Everyday Workflow: Jumping from Cloud Shell into a VM

```
# 1. Start in Cloud Shell: Open an SSH connection to your VM
gcloud compute ssh gcelab2 --zone=asia-east1-a

# 2. Now you are INSIDE the VM: Download the latest package list and install NGINX
sudo apt-get update
sudo apt-get install -y nginx

# 3. Still INSIDE the VM: Check if NGINX is running and test the local web page
sudo systemctl status nginx
curl http://localhost

# 4. Leave the VM and return to Cloud Shell
exit

```

### 5.3 SSH Keys, Password Prompts, and OS Login

#### How Do SSH Keys Work?

Instead of typing a regular password, [SSH](https://cloud.google.com/compute/docs/instances/ssh) uses a matching pair of digital files called a **Key Pair**:

| Key File | Simple Analogy | Where It Lives & Rule | 
| ----- | ----- | ----- | 
| **Public Key** | Like a **padlock** you bolt onto the VM's door ([About SSH connections and SSH keys](https://cloud.google.com/compute/docs/instances/ssh)). | Stored on the VM. Anyone can see the padlock; it is safe to share. | 
| **Private Key** | Like the **secret physical key** in your pocket that opens that padlock ([About SSH connections and SSH keys](https://cloud.google.com/compute/docs/instances/ssh)). | Stored privately on your computer or Cloud Shell. **Never share it with anyone.** | 

When you click the **SSH** button in the console or run `gcloud compute ssh`, Google Cloud automatically creates this key pair for you and puts the public key onto the VM behind the scenes.

#### Why Is SSH Asking Me for a Password?

If you see a password prompt, read the exact words before typing:

1. **`Enter passphrase for key ...`**: `gcloud` is asking if you want an extra password to lock your Private Key file in Cloud Shell. In training labs, you can simply press `Enter` (leave it blank).

2. **`username@host's password:`**: The VM is asking for a Linux user password. Because Google Cloud Linux VMs do not use passwords by default, seeing this usually means your SSH key or IAM permission isn't set up properly—do not try guessing your Gmail password here!

3. **`[sudo] password for username:`**: Linux inside the VM is asking to confirm administrator rights when running `sudo`.

#### What Is OS Login?

When a team of 50 developers works on VMs, managing SSH key files by hand gets messy. [OS Login](https://cloud.google.com/compute/docs/oslogin) links Linux SSH login directly to a person's Google Cloud IAM roles:

* **`roles/compute.osLogin`**: Lets the user log into the VM as a normal user ([OS Login](https://cloud.google.com/compute/docs/oslogin)).

* **`roles/compute.osAdminLogin`**: Lets the user log into the VM and run `sudo` admin commands ([OS Login](https://cloud.google.com/compute/docs/oslogin)).

### 5.4 Regions, Zones, and Shell Variables

* **Region**: A geographic area where Google has data centers, such as `asia-east1` (Taiwan) or `us-west1` (Oregon) ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)).

* **Zone**: An individual data center building inside a region, such as `asia-east1-a`, `asia-east1-b`, or `asia-east1-c` ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)).

**Why does location matter?** Because certain items must live in the exact same place to plug into each other. For example, a zonal VM in `asia-east1-a` can only attach a zonal [Persistent Disk](https://cloud.google.com/compute/docs/disks/persistent-disks) that is **also** in `asia-east1-a`.

```
# Check which Google account and project are currently active
gcloud auth list
gcloud config get-value project

# Save a default region and zone in your gcloud settings
gcloud config set compute/region asia-east1
gcloud config set compute/zone asia-east1-a

# Or save them as shell variables ($REGION and $ZONE) for your current terminal window
export REGION=asia-east1
export ZONE=asia-east1-a

```

### 5.5 Adding a Second Hard Drive (Persistent Disk) to a VM

A [Persistent Disk](https://cloud.google.com/compute/docs/disks/persistent-disks) is durable network storage that acts like an external hard drive for your VM. Keeping your application files on a separate data disk (instead of the main boot disk) makes it easy to resize, back up, or move that disk to another VM later.

#### Why Do We Need SSH When Attaching a Disk?

Think of what happens when you buy a brand-new blank USB hard drive for your laptop:

1. **Physical step (Cloud Shell)**: You plug the cable into the computer (`gcloud compute instances attach-disk`).

2. **Operating System step (Inside VM via SSH)**: Even though the drive is plugged in, Linux cannot save files on a raw, blank disk yet. You have to log into the VM via [SSH](https://cloud.google.com/compute/docs/connect/standard-ssh) to **format** the drive (`mkfs.ext4`), create a **folder** for it (`mkdir /mnt/mydisk`), and **connect (mount)** the drive to that folder (`mount`).

#### Step-by-Step Commands to Create, Attach, Format, and Mount a Disk

**Part A — Run these commands in Cloud Shell (outside the VM):**

```
# Step 1: Create a Debian VM named 'gcelab'
gcloud compute instances create gcelab \
  --zone="$ZONE" \
  --machine-type=e2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud

# Step 2: Create a blank 200 GB Persistent Disk in the exact same zone
gcloud compute disks create mydisk \
  --size=200GB \
  --zone="$ZONE"

# Step 3: Plug (attach) the disk into the VM
gcloud compute instances attach-disk gcelab \
  --disk=mydisk \
  --zone="$ZONE"

# Step 4: SSH into the VM so we can format and mount the disk inside Linux
gcloud compute ssh gcelab --zone="$ZONE"

```

**Part B — Run these commands inside the VM's SSH terminal:**

```
# Step 5: Find the exact name of the newly attached disk
ls -l /dev/disk/by-id/
lsblk -f

# Step 6: Create an empty folder (/mnt/mydisk) where the disk's files will show up
sudo mkdir -p /mnt/mydisk

# Step 7: Format the blank disk with the ext4 Linux filesystem
# WARNING: Formatting wipes a disk clean! Double-check the device ID from Step 5 first.
sudo mkfs.ext4 -F /dev/disk/by-id/scsi-0Google_PersistentDisk_persistent-disk-1

# Step 8: Mount (connect) the formatted disk to the /mnt/mydisk folder
sudo mount -o discard,defaults \
  /dev/disk/by-id/scsi-0Google_PersistentDisk_persistent-disk-1 \
  /mnt/mydisk

# Check that the 200 GB disk now appears in your mounted drives list
df -h

```

**Part C — Make the disk reconnect automatically when the VM restarts (`/etc/fstab`):**

If you restart the VM right now, Linux will forget that `/mnt/mydisk` was mounted. To make Linux reconnect it automatically on every boot, add its unique ID (`UUID`) to `/etc/fstab` ([Add a Persistent Disk to a VM](https://cloud.google.com/compute/docs/disks/add-persistent-disk)):

```
# Step 9: Find the unique UUID of your new filesystem
sudo blkid

# Step 10: Open /etc/fstab in the nano text editor
sudo nano /etc/fstab

# Add a single line at the bottom of /etc/fstab (replace with your real UUID):
# UUID=YOUR-FILESYSTEM-UUID /mnt/mydisk ext4 discard,defaults,nofail 0 2
# ('nofail' tells Linux to still boot normally even if the disk is ever unplugged)

# Step 11: Test your /etc/fstab file safely right now before rebooting
sudo umount /mnt/mydisk
sudo mount -a
df -h

```

### 5.6 Persistent Disk vs. Local SSD

| Feature | Persistent Disk (Durable Storage) | Local SSD (Super-Fast Scratch Space) | 
| ----- | ----- | ----- | 
| **Where is it physically located?** | Connected over Google's fast internal network ([Persistent Disk](https://cloud.google.com/compute/docs/disks/persistent-disks)). | Physically wired inside the exact host computer running your VM ([Local SSD](https://cloud.google.com/compute/docs/disks/local-ssd)). | 
| **Is my data safe if the VM stops?** | **Yes.** Your data stays safe even if you stop or delete the VM ([Persistent Disk](https://cloud.google.com/compute/docs/disks/persistent-disks)). | **No.** While data survives a simple OS reboot, stopping the VM or a hardware crash can **wipe the disk completely** ([Local SSD](https://cloud.google.com/compute/docs/disks/local-ssd)). | 
| **What should I store on it?** | Operating systems, databases, and important application files ([Persistent Disk](https://cloud.google.com/compute/docs/disks/persistent-disks)). | Temporary caches or scratch files that you can easily recreate if lost ([Local SSD](https://cloud.google.com/compute/docs/disks/local-ssd)). | 

### 5.7 Storing Code in a Cloud Storage Bucket

> **Note on Zones:** From this point forward, the lab switches from `asia-east1-a` to `us-west1-b` to build the multi-VM online store (`fancy-fe` and `fancy-be`).

#### What Is a Cloud Storage Bucket?

A [Cloud Storage](https://cloud.google.com/storage/docs) bucket is online file storage (like a cloud folder for your project) accessed over an API. It is **not** a hard drive attached to a single VM.

#### Why Do We Copy Code and Startup Scripts to a Bucket?

Soon, we will create a group of VMs that automatically adds new VMs when traffic spikes. You don't want to log into every new VM by hand to copy your website code! Instead:

1. You upload your website code and a setup script (`startup-script.sh`) into **one central bucket**.

2. Every time a new VM turns on, it automatically grabs the script and code from that bucket and starts the app by itself.

#### Why Delete `node_modules` Before Uploading?

In Node.js apps, the `node_modules` folder contains thousands of tiny library files downloaded for the computer where they were first installed. Deleting `node_modules` before uploading makes the upload much faster and lets each VM install fresh, clean packages that match its own Linux system.

```
# 1. Create a bucket (bucket names must be globally unique across all of Google Cloud)
gcloud storage buckets create gs://YOUR_UNIQUE_BUCKET_NAME

# 2. Delete old local node_modules folders so we only upload clean source code
# (Be careful with rm -rf: always double-check the folder path before pressing Enter!)
rm -rf ~/monolith-to-microservices/*/node_modules

# 3. Copy the startup script and the code folder into your bucket
gcloud storage cp ~/monolith-to-microservices/startup-script.sh \
  gs://YOUR_UNIQUE_BUCKET_NAME/startup-script.sh

gcloud storage cp -r ~/monolith-to-microservices \
  gs://YOUR_UNIQUE_BUCKET_NAME/

```

### 5.8 Startup Scripts & Giving VMs Permission to Read the Bucket

A [startup script](https://cloud.google.com/compute/docs/instances/startup-scripts) is a bash script that [Compute Engine](https://cloud.google.com/compute/docs/overview) runs automatically every time a VM boots up. It installs software (like NGINX or Node.js), downloads your code from the bucket, and starts your server.

**Security tip:** Never make your code bucket public to the whole internet just so your VM can read `startup-script.sh`. Instead, create a **Service Account**, give it permission to read that bucket (`roles/storage.objectViewer`), and attach it to your VMs:

```
# 1. Create a Service Account for your store VMs
gcloud iam service-accounts create fancy-vm-sa \
  --display-name="Fancy Store VM Service Account"

# 2. Allow that Service Account to read files from your private bucket
gcloud storage buckets add-iam-policy-binding gs://YOUR_UNIQUE_BUCKET_NAME \
  --member="serviceAccount:fancy-vm-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# 3. Create the 'backend' VM with the Service Account, 'backend' tag, and startup script
gcloud compute instances create backend \
  --zone=us-west1-b \
  --machine-type=e2-standard-2 \
  --service-account=fancy-vm-sa@PROJECT_ID.iam.gserviceaccount.com \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --tags=backend \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script-url=gs://YOUR_UNIQUE_BUCKET_NAME/startup-script.sh

# 4. Create the 'frontend' VM with the Service Account, 'frontend' tag, and startup script
gcloud compute instances create frontend \
  --zone=us-west1-b \
  --machine-type=e2-standard-2 \
  --service-account=fancy-vm-sa@PROJECT_ID.iam.gserviceaccount.com \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --tags=frontend \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script-url=gs://YOUR_UNIQUE_BUCKET_NAME/startup-script.sh

```

**Good habits for startup scripts:**

* Write the script so that if the VM reboots and runs the script a second time, it won't crash or duplicate files (this is called making the script **idempotent**).

* Never hardcode database passwords or secret API keys in plain text inside a startup script—use **Secret Manager** instead.

### 5.9 Network Tags and Firewall Rules

Even after your startup script starts your web app on port `8080`, nobody from the outside can reach it until you open that port in the **VPC Firewall**.

Instead of writing firewall rules for individual IP addresses, we use **Network Tags** (`--tags=frontend` and `--tags=backend`):

1. A **network tag** is just a sticky label on a VM (`frontend` or `backend`). By itself, a tag does nothing.

2. A **firewall rule** says: *"Allow traffic on port 8080 to any VM wearing the `frontend` sticky label."*

```
# Open port 8080 for any VM tagged 'frontend'
gcloud compute firewall-rules create fw-fe \
  --allow=tcp:8080 \
  --target-tags=frontend

# Open ports 8081 (Orders) and 8082 (Products) for any VM tagged 'backend'
gcloud compute firewall-rules create fw-be \
  --allow=tcp:8081-8082 \
  --target-tags=backend

```

### 5.10 Instance Templates (Saved VM Blueprints)

Suppose you want 4 identical frontend VMs. Typing `gcloud compute instances create` 4 times by hand is slow and easy to mess up.

An [Instance Template](https://cloud.google.com/compute/docs/instance-templates) is a **saved blueprint** that records everything about a VM (machine size, Linux image, network tags, Service Account, and startup script URL).

* **Important rule — Templates cannot be edited**: Once you create an instance template, it is locked forever ([Instance templates](https://cloud.google.com/compute/docs/instance-templates)). If you want to change the VM size later, you simply create a **new** template (like `fancy-fe-new`) and tell your group of VMs to switch to the new blueprint.

```
# 1. Stop the two prototype VMs before taking their blueprint
gcloud compute instances stop frontend --zone=us-west1-b
gcloud compute instances stop backend --zone=us-west1-b

# 2. Create blueprints (Instance Templates) from the stopped VMs
gcloud compute instance-templates create fancy-fe \
  --source-instance=frontend \
  --source-instance-zone=us-west1-b

gcloud compute instance-templates create fancy-be \
  --source-instance=backend \
  --source-instance-zone=us-west1-b

# 3. View your saved templates
gcloud compute instance-templates list

```

### 5.11 Managed Instance Groups (MIGs) and Named Ports

#### What Is a Managed Instance Group (MIG)?

A [Managed Instance Group (MIG)](https://cloud.google.com/compute/docs/instance-groups) uses your Instance Template blueprint to run a team of identical VMs automatically.

Why use a MIG instead of standalone VMs?

* **Self-healing (Autohealing)**: If one VM crashes, the MIG deletes it and builds a fresh replacement automatically.

* **Autoscaling**: When website traffic jumps, the MIG automatically adds more VMs, and removes them when traffic quiets down.

* **Easy updates**: You can roll out a new template across all VMs smoothly without taking your whole site down.

```
# 1. Create the frontend MIG with 2 VMs using the 'fancy-fe' blueprint
gcloud compute instance-groups managed create fancy-fe-mig \
  --zone=us-west1-b \
  --base-instance-name=fancy-fe \
  --size=2 \
  --template=fancy-fe

# 2. Create the backend MIG with 2 VMs using the 'fancy-be' blueprint
gcloud compute instance-groups managed create fancy-be-mig \
  --zone=us-west1-b \
  --base-instance-name=fancy-be \
  --size=2 \
  --template=fancy-be

# 3. See the actual VMs running inside the frontend MIG
gcloud compute instance-groups managed list-instances fancy-fe-mig \
  --zone=us-west1-b

```

| MIG Setting / Term | Simple Explanation | 
| ----- | ----- | 
| `--base-instance-name=fancy-fe` | The starter name for VMs in the group (Google adds random letters like `fancy-fe-9x2k`) ([Managed instance groups](https://cloud.google.com/compute/docs/instance-groups)). | 
| `--size=2` | Keeps 2 VMs running in the group at all times. | 
| **Zonal vs. Regional MIG** | A **Zonal MIG** puts all VMs in one zone (`us-west1-b`). A **Regional MIG** spreads VMs across multiple zones in a region so your app stays online even if an entire data center building goes dark ([Managed instance groups](https://cloud.google.com/compute/docs/instance-groups)). | 

#### What Are Named Ports?

A **Named Port** is simply a nickname for a port number on a MIG ([External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)). Instead of making the Load Balancer memorize raw port numbers, we label the ports on our MIGs:

* On `fancy-fe-mig`: `frontend` means port `8080`.

* On `fancy-be-mig`: `orders` means port `8081`, and `products` means port `8082`.

```
# Give port 8080 the nickname 'frontend' on the frontend MIG
gcloud compute instance-groups set-named-ports fancy-fe-mig \
  --zone=us-west1-b \
  --named-ports=frontend:8080

# Give port 8081 the nickname 'orders' and port 8082 the nickname 'products' on the backend MIG
gcloud compute instance-groups set-named-ports fancy-be-mig \
  --zone=us-west1-b \
  --named-ports=orders:8081,products:8082

```

### 5.12 Health Checks and Autohealing

Sometimes a VM is turned on (`RUNNING`), but the Node.js app inside it has frozen or crashed. A **Health Check** is an automated ping that visits a web path (like `/` or `/api/orders`) every few seconds to check if the app answers with a healthy `200 OK` status.

Google Cloud uses health checks in two different places:

| Health Check Type | What Question Does It Ask? | What Happens If the VM Fails the Check? | 
| ----- | ----- | ----- | 
| **Load-Balancer Health Check** | *"Should I send live customer traffic to this VM right now?"* ([External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)) | The Load Balancer temporarily stops sending users to that VM until it recovers. It does **not** delete the VM. | 
| **MIG Autohealing Health Check** | *"Is this VM completely broken and in need of replacement?"* ([Managed instance groups](https://cloud.google.com/compute/docs/instance-groups)) | The MIG deletes the broken VM and builds a brand-new VM from the Instance Template. | 

```
# 1. Create Autohealing health checks (checks every 30s; 3 failures in a row marks it broken)
gcloud compute health-checks create http fancy-fe-hc \
  --port=8080 \
  --check-interval=30s \
  --healthy-threshold=1 \
  --timeout=10s \
  --unhealthy-threshold=3

gcloud compute health-checks create http fancy-be-hc \
  --port=8081 \
  --request-path=/api/orders \
  --check-interval=30s \
  --healthy-threshold=1 \
  --timeout=10s \
  --unhealthy-threshold=3

# 2. Open the firewall so Google's health-check servers can reach ports 8080, 8081, and 8082
gcloud compute firewall-rules create allow-health-check \
  --allow=tcp:8080-8082 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --network=default

# 3. Turn on Autohealing on both MIGs (--initial-delay=300 gives a new VM 5 minutes to finish its startup script before health checks judge it)
gcloud compute instance-groups managed update fancy-fe-mig \
  --zone=us-west1-b \
  --health-check=fancy-fe-hc \
  --initial-delay=300

gcloud compute instance-groups managed update fancy-be-mig \
  --zone=us-west1-b \
  --health-check=fancy-be-hc \
  --initial-delay=300

```

### 5.13 Load Balancing (Putting Everything Behind One Public IP)

#### What Is a Load Balancer?

If 1,000 shoppers visit your online store at once, sending all of them to a single VM will overload it. An [External Application Load Balancer](https://cloud.google.com/load-balancing/docs/https) sits in front of your MIGs, gives your website **one single public IP address**, and spreads visitors evenly across your healthy VMs.

Even better, it can read the URL path a shopper visits and send the request to the right service:

* Shopper visits `/` → goes to the **Frontend Service** (`fancy-fe-mig` on port `8080`)

* Shopper visits `/api/orders` → goes to the **Orders Service** (`fancy-be-mig` on port `8081`)

* Shopper visits `/api/products` → goes to the **Products Service** (`fancy-be-mig` on port `8082`)

#### How the Load Balancer Parts Fit Together

Think of a request arriving from a user's browser and passing through four doors in order:

**User Browser → 1. Forwarding Rule → 2. Target HTTP Proxy → 3. URL Map → 4. Backend Service → Healthy MIG VMs**

| Load Balancer Part | What It Does in Simple Words | Name in Our Commands | 
| ----- | ----- | ----- | 
| **1. Forwarding Rule** | The front door: holds the public IP address and listens on port `80` ([External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)). | `fancy-http-rule` | 
| **2. Target HTTP Proxy** | The receptionist: accepts the HTTP request from the front door and opens the routing directory (URL Map) ([Target proxies](https://cloud.google.com/load-balancing/docs/target-proxies)). | `fancy-proxy` | 
| **3. URL Map** | The routing directory: looks at the URL path (`/api/orders`, `/api/products`, or `/`) and picks the matching Backend Service ([URL maps](https://cloud.google.com/load-balancing/docs/url-map-concepts)). | `fancy-map` | 
| **4. Backend Service** | The team manager: connects the route to the actual MIG, checks its health check, and knows which named port (`frontend`, `orders`, or `products`) to use. | `fancy-fe-frontend`, `fancy-be-orders`, `fancy-be-products` | 

#### Complete Commands to Build the Load Balancer

Notice how `fancy-be-orders` and `fancy-be-products` are two separate Backend Services, but both point to the **same** `fancy-be-mig` group using its two different named ports (`orders` and `products`):

```
# STEP 1: Create Load-Balancer health checks for all three services (8080, 8081, and 8082)
gcloud compute health-checks create http fancy-fe-frontend-hc \
  --request-path=/ \
  --port=8080

gcloud compute health-checks create http fancy-be-orders-hc \
  --request-path=/api/orders \
  --port=8081

gcloud compute health-checks create http fancy-be-products-hc \
  --request-path=/api/products \
  --port=8082

# STEP 2: Create the three Backend Services using the named ports we set in Section 5.11
gcloud compute backend-services create fancy-fe-frontend \
  --health-checks=fancy-fe-frontend-hc \
  --port-name=frontend \
  --global

gcloud compute backend-services create fancy-be-orders \
  --health-checks=fancy-be-orders-hc \
  --port-name=orders \
  --global

gcloud compute backend-services create fancy-be-products \
  --health-checks=fancy-be-products-hc \
  --port-name=products \
  --global

# STEP 3: Connect the MIGs to each Backend Service
gcloud compute backend-services add-backend fancy-fe-frontend \
  --instance-group=fancy-fe-mig \
  --instance-group-zone=us-west1-b \
  --global

gcloud compute backend-services add-backend fancy-be-orders \
  --instance-group=fancy-be-mig \
  --instance-group-zone=us-west1-b \
  --global

gcloud compute backend-services add-backend fancy-be-products \
  --instance-group=fancy-be-mig \
  --instance-group-zone=us-west1-b \
  --global

# STEP 4: Create the URL Map and add rules for /api/orders and /api/products
gcloud compute url-maps create fancy-map \
  --default-service=fancy-fe-frontend

gcloud compute url-maps add-path-matcher fancy-map \
  --default-service=fancy-fe-frontend \
  --path-matcher-name=api-routes \
  --new-hosts="*" \
  --path-rules="/api/orders=fancy-be-orders,/api/products=fancy-be-products"

# STEP 5: Create the Target HTTP Proxy and the Global Forwarding Rule on port 80
gcloud compute target-http-proxies create fancy-proxy \
  --url-map=fancy-map

gcloud compute forwarding-rules create fancy-http-rule \
  --global \
  --target-http-proxy=fancy-proxy \
  --ports=80

# STEP 6: Find the public IP address of your new Load Balancer
gcloud compute forwarding-rules list --global

```

### 5.14 SSL/TLS Certificates (HTTP vs. HTTPS)

Why do real websites use `https://` instead of `http://`?

* **HTTP (Port 80)**: Sends information as plain, readable text across the internet. Anyone on the network path could snoop on passwords or credit card numbers ([External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)).

* **HTTPS (Port 443)**: Uses an **SSL/TLS Certificate** to prove the website really belongs to your domain and scrambles (encrypts) all data between the user's browser and your Load Balancer ([Target proxies](https://cloud.google.com/load-balancing/docs/target-proxies)).

| Feature | HTTP Load Balancer (Used in Lab) | HTTPS Load Balancer (Used in Production) | 
| ----- | ----- | ----- | 
| **Is traffic encrypted?** | No (plain text). | Yes (encrypted with TLS). | 
| **Standard port** | Port `80` | Port `443` | 
| **Proxy used** | `target-http-proxies` ([Target proxies](https://cloud.google.com/load-balancing/docs/target-proxies)) | `target-https-proxies` + an SSL Certificate ([Target proxies](https://cloud.google.com/load-balancing/docs/target-proxies)) | 

The lab uses HTTP on port `80` only because creating an SSL certificate requires owning a real domain name (like `example.com`).

### 5.15 Autoscaling and Cloud CDN

#### What Is Autoscaling?

**Autoscaling** lets your MIG automatically add more VMs when traffic gets busy, and delete extra VMs when traffic drops so you save money ([Managed instance groups](https://cloud.google.com/compute/docs/instance-groups)).

> **Watch out for this lab trap:** If your MIG already runs `2` VMs and you set `--max-num-replicas=2`, the autoscaler can never add a 3rd VM! Always set `--max-num-replicas` higher than your starting size (for example, `5`) if you want it to scale out:

```
# Automatically scale fancy-be-mig between 2 and 5 VMs when load exceeds 60%
gcloud compute instance-groups managed set-autoscaling fancy-be-mig \
  --zone=us-west1-b \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-load-balancing-utilization=0.60

```

#### What Is Cloud CDN?

Suppose 10,000 users around the world open your store homepage and download the same product photos and CSS files. Without caching, every single photo request has to travel all the way to your VMs in `us-west1-b`.

[Cloud CDN (Content Delivery Network)](https://cloud.google.com/cdn/docs/overview) saves temporary copies of your static images and files at Google edge locations close to your users:

* **Cache Hit**: Google already has a copy of the image nearby and sends it straight to the user immediately (fast for the user, zero work for your VM) ([Cloud CDN overview](https://cloud.google.com/cdn/docs/overview)).

* **Cache Miss**: Google does not have the file cached yet, so it fetches it from your VM once and saves a copy for the next visitor ([Cloud CDN overview](https://cloud.google.com/cdn/docs/overview)).

```
# Turn on Cloud CDN for the frontend Backend Service
gcloud compute backend-services update fancy-fe-frontend \
  --enable-cdn \
  --global

```

### 5.16 Updating Your App vs. Updating Your VM Template

When you want to update your running website, ask yourself: **Did I change the application code, or did I change the VM's hardware/settings?**

| What Changed? | Simple Example | How to Roll It Out | 
| ----- | ----- | ----- | 
| **Case A: Only your app code changed** | You changed text on the homepage or updated Node.js code inside the bucket ([Startup scripts](https://cloud.google.com/compute/docs/instances/startup-scripts)). | Copy the new code to your bucket, then tell the MIG to **replace** its VMs (`rolling-action replace`) so fresh VMs boot up and download the new code ([Rolling updates for managed instance groups](https://cloud.google.com/compute/docs/instance-groups/rolling-out-updates-to-managed-instance-groups)). | 
| **Case B: The VM settings changed** | You want a smaller machine type (`e2-small` instead of `e2-standard-2`) or a new Linux image ([Instance templates](https://cloud.google.com/compute/docs/instance-templates)). | Because templates cannot be edited, create a **new Instance Template** (`fancy-fe-new`) and run `rolling-action start-update` on the MIG ([Rolling updates for managed instance groups](https://cloud.google.com/compute/docs/instance-groups/rolling-out-updates-to-managed-instance-groups)). | 

```
# CASE A: Upload updated code to the bucket, then replace the VMs in fancy-fe-mig
# (Note: --max-unavailable=100% replaces all VMs at once so the lab finishes fast.
# In real projects, keep --max-unavailable=0 so users never see downtime!)
cd ~
rm -rf monolith-to-microservices/*/node_modules
gcloud storage cp -r monolith-to-microservices gs://YOUR_UNIQUE_BUCKET_NAME/

gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
  --zone=us-west1-b \
  --max-unavailable=100%

# CASE B: Change the stopped 'frontend' VM to e2-small, save a new template, and update the MIG
gcloud compute instances set-machine-type frontend \
  --zone=us-west1-b \
  --machine-type=e2-small

gcloud compute instance-templates create fancy-fe-new \
  --source-instance=frontend \
  --source-instance-zone=us-west1-b

gcloud compute instance-groups managed rolling-action start-update fancy-fe-mig \
  --zone=us-west1-b \
  --version=template=fancy-fe-new

# Watch the Load Balancer health and MIG rollout live every 2 seconds (Press Ctrl+C to stop)
watch -n 2 gcloud compute backend-services get-health fancy-fe-frontend --global
watch -n 2 gcloud compute instance-groups managed list-instances fancy-fe-mig --zone=us-west1-b

```

### 5.17 Why Does the Lab Use SSH to Kill a Node.js Process?

To prove that **Autohealing** works, labs sometimes ask you to log into a VM with [SSH](https://cloud.google.com/compute/docs/connect/standard-ssh), find the running Node.js app, and stop (`kill`) it on purpose. Because the Node.js process is running inside the VM's Linux operating system, you must step inside the VM via SSH to stop it:

```
# 1. From Cloud Shell: SSH into one of the frontend VMs
gcloud compute ssh FANCY_FE_INSTANCE_NAME --zone=us-west1-b

# 2. Inside the VM: Find the Process ID (PID) of the running Node.js app
ps aux | grep node

# 3. Inside the VM: Stop that process politely using its PID number
kill PID

```

Once you stop the Node.js process, the health check on port `8080` starts failing, and after 3 failures in a row, the MIG automatically replaces the VM!

**Do you need SSH to restart apps in real life?**
No. In production, you configure a Linux service manager called **`systemd`** (`sudo systemctl restart myapp`) so Linux restarts a crashed process automatically within seconds, or you deploy new versions through an automated CI/CD pipeline.

### 5.18 Putting It All Together: The Complete Picture

Here is how all 15 steps in the lab connect from start to finish:

1. **Store code centrally**: Put your app code and `startup-script.sh` into a [Cloud Storage](https://cloud.google.com/storage/docs) bucket.

2. **Set up permissions**: Create a **Service Account** (`fancy-vm-sa`) that can read that bucket.

3. **Build prototype VMs**: Create `frontend` and `backend` VMs that run the startup script on boot.

4. **Open firewall ports**: Add network tags (`frontend`, `backend`) and firewall rules for ports `8080`, `8081`, and `8082`.

5. **Save blueprints**: Stop the prototype VMs and save them as immutable [Instance Templates](https://cloud.google.com/compute/docs/instance-templates) (`fancy-fe` and `fancy-be`).

6. **Launch MIGs**: Create two [Managed Instance Groups](https://cloud.google.com/compute/docs/instance-groups) (`fancy-fe-mig` and `fancy-be-mig`) from those templates and label their **Named Ports** (`frontend:8080`, `orders:8081`, `products:8082`).

7. **Turn on Autohealing**: Attach health checks to the MIGs and allow Google's health-check IPs (`130.211.0.0/22`, `35.191.0.0/16`) through the firewall.

8. **Add a Load Balancer**: Create three **Backend Services** (`fancy-fe-frontend`, `fancy-be-orders`, `fancy-be-products`), connect them to a [URL Map](https://cloud.google.com/load-balancing/docs/url-map-concepts) (`fancy-map`), and expose them on port `80` with a **Target HTTP Proxy** and **Forwarding Rule**.

9. **Turn on Autoscaling & CDN**: Enable **Autoscaling** so VMs scale with traffic and [Cloud CDN](https://cloud.google.com/cdn/docs/overview) so static files load fast from nearby edge caches.

### 5.19 Simple Troubleshooting Checklist

When something does not work in a lab, check this table first:

| Problem You See | Most Likely Cause | Command to Check It | 
| ----- | ----- | ----- | 
| **Cannot create a VM** | Wrong project/zone selected, Compute Engine API is off, or missing IAM permission ([Compute Engine overview](https://cloud.google.com/compute/docs/overview)). | `gcloud config list` and `gcloud services list --enabled` | 
| **Cannot SSH into a VM** | The VM is stopped, port `22` is blocked by a firewall rule, or you lack the [OS Login](https://cloud.google.com/compute/docs/oslogin) IAM role. | `gcloud compute instances list` | 
| **App did not start on the VM** | The startup script URL had a typo, or the VM's Service Account cannot read the bucket ([Startup scripts](https://cloud.google.com/compute/docs/instances/startup-scripts)). | `sudo journalctl -u google-startup-scripts.service` (inside VM) | 
| **Load Balancer says `UNHEALTHY`** | The `allow-health-check` firewall rule doesn't cover ports `8080-8082`, the named port doesn't match, or the startup script is still running ([External Application Load Balancer overview](https://cloud.google.com/load-balancing/docs/https)). | `gcloud compute backend-services get-health SERVICE_NAME --global` | 
| **Autoscaler never adds a 3rd VM** | `--max-num-replicas` was set to `2` (the same as the current size) instead of a higher number ([Managed instance groups](https://cloud.google.com/compute/docs/instance-groups)). | `gcloud compute instance-groups managed describe MIG_NAME --zone=ZONE` | 

## 6. Quick Command Reference

| What You Want to Do | Command to Run | 
| ----- | ----- | 
| **Check logged-in Google account** | `gcloud auth list` | 
| **Check current Project ID** | `gcloud config get-value project` | 
| **Switch to another Project ID** | `gcloud config set project PROJECT_ID` | 
| **Set default zone and region** | `gcloud config set compute/zone ZONE` and `gcloud config set compute/region REGION` | 
| **Turn on an API** | `gcloud services enable compute.googleapis.com` | 
| **Give a user Viewer access** | `gcloud projects add-iam-policy-binding PROJECT_ID --member="user:EMAIL" --role="roles/viewer"` | 
| **Create a Service Account** | `gcloud iam service-accounts create SA_NAME --display-name="NAME"` | 
| **Create a Pub/Sub topic & subscription** | `gcloud pubsub topics create TOPIC` and `gcloud pubsub subscriptions create SUB --topic=TOPIC` | 
| **List all VMs** | `gcloud compute instances list` | 
| **Create a basic Debian VM** | `gcloud compute instances create VM_NAME --zone=ZONE --machine-type=e2-medium --image-family=debian-12 --image-project=debian-cloud` | 
| **SSH into a VM** | `gcloud compute ssh VM_NAME --zone=ZONE` | 
| **Stop / Start a VM** | `gcloud compute instances stop VM_NAME --zone=ZONE` / `gcloud compute instances start VM_NAME --zone=ZONE` | 
| **Create & attach a Persistent Disk** | `gcloud compute disks create DISK --size=200GB --zone=ZONE` and `gcloud compute instances attach-disk VM_NAME --disk=DISK --zone=ZONE` | 
| **Copy a file or folder to a bucket** | `gcloud storage cp FILE gs://BUCKET/` and `gcloud storage cp -r FOLDER gs://BUCKET/` | 
| **Create an Instance Template** | `gcloud compute instance-templates create TEMPLATE --source-instance=VM_NAME --source-instance-zone=ZONE` | 
| **Create a Managed Instance Group (MIG)** | `gcloud compute instance-groups managed create MIG --zone=ZONE --base-instance-name=BASE --size=2 --template=TEMPLATE` | 
| **Set Named Ports on a MIG** | `gcloud compute instance-groups set-named-ports MIG --zone=ZONE --named-ports=orders:8081,products:8082` | 
| **Check Load Balancer backend health** | `gcloud compute backend-services get-health BACKEND_SERVICE --global` | 

## 7. Official Google Cloud Documentation Links

* [Compute Engine Overview](https://cloud.google.com/compute/docs/overview)

* [Connect to Linux VMs Using SSH](https://cloud.google.com/compute/docs/connect/standard-ssh)

* [About SSH Connections and SSH Keys](https://cloud.google.com/compute/docs/instances/ssh)

* [Manage VM Access with OS Login](https://cloud.google.com/compute/docs/oslogin)

* [Persistent Disk Overview](https://cloud.google.com/compute/docs/disks/persistent-disks)

* [Add a Persistent Disk to a VM](https://cloud.google.com/compute/docs/disks/add-persistent-disk)

* [Local SSD Documentation](https://cloud.google.com/compute/docs/disks/local-ssd)

* [Use Startup Scripts on Linux VMs](https://cloud.google.com/compute/docs/instances/startup-scripts)

* [Instance Templates Documentation](https://cloud.google.com/compute/docs/instance-templates)

* [Managed Instance Groups (MIGs)](https://cloud.google.com/compute/docs/instance-groups)

* [Rolling Out Updates to Managed Instance Groups](https://cloud.google.com/compute/docs/instance-groups/rolling-out-updates-to-managed-instance-groups)

* [External Application Load Balancer Overview](https://cloud.google.com/load-balancing/docs/https)

* [URL Maps and Path Matchers](https://cloud.google.com/load-balancing/docs/url-map-concepts)

* [Target Proxies for Load Balancing](https://cloud.google.com/load-balancing/docs/target-proxies)

* [Cloud Storage Documentation](https://cloud.google.com/storage/docs)

* [Cloud CDN Overview](https://cloud.google.com/cdn/docs/overview)
