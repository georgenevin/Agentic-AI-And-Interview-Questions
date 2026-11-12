# Google Cloud Practical Learning Handbook

## 1. Google Cloud Projects

A Google Cloud project is a fundamental organizing entity for managing
Google Cloud resources, services, APIs, billing associations, and access
control.

A project can contain resources such as virtual machines, databases,
storage buckets, and networks.

### 1.1 Project identifiers

  -----------------------------------------------------------------------
  Identifier              Description             Example
  ----------------------- ----------------------- -----------------------
  Project name            Human-readable display  My Development Project
                          name. It does not need  
                          to be globally unique.  

  Project ID              Globally unique         `my-project-123`
                          identifier used in      
                          commands and APIs.      

  Project number          Automatically generated `123456789012`
                          numeric identifier used 
                          by Google Cloud.        
  -----------------------------------------------------------------------

Important points:

-   Project IDs are globally unique across Google Cloud.
-   Project IDs are selected when creating a project and generally
    cannot be changed later.
-   Project names can be changed.
-   Project numbers are automatically generated and cannot be changed.
-   Project IDs and project numbers both identify projects, but serve
    different purposes.

### 1.2 Practical implementation using gcloud

``` bash
# Display the currently selected project
gcloud config get-value project

# List projects accessible to the current account
gcloud projects list

# Set the active project
gcloud config set project PROJECT_ID

# Display project details
gcloud projects describe PROJECT_ID
```

### 1.3 Real-world example

For an employee policy chatbot, separate projects can be created for
different environments:

-   `policy-bot-dev`
-   `policy-bot-test`
-   `policy-bot-prod`

This helps isolate resources, permissions, billing, and configurations
across environments.

------------------------------------------------------------------------

## 2. Identity and Access Management (IAM)

Google Cloud Identity and Access Management (IAM) controls who can
access Google Cloud resources and what actions they can perform.

An IAM policy binding connects a principal, a role, and a resource.

### 2.1 Three components of an IAM binding

  -----------------------------------------------------------------------
  Component              Description            Example
  ---------------------- ---------------------- -------------------------
  Principal              The identity that      `developer@example.com`
                         needs access.          

  Role                   A collection of        Compute Instance Admin
                         permissions granted to 
                         the principal.         

  Resource               The Google Cloud       A development project
                         resource on which      
                         access is granted.     
  -----------------------------------------------------------------------

The relationship can be represented as:

**Principal → Role → Permissions → Resource**

For example, a developer may receive the Compute Instance Admin role on
a development project to manage virtual machine instances.

### 2.2 Role types

  ---------------------------------------------------------------------
  Role type                          Description
  ---------------------------------- ----------------------------------
  Basic roles                        Broad roles such as Owner, Editor,
                                     and Viewer.

  Predefined roles                   Google-managed roles designed for
                                     specific services and tasks.

  Custom roles                       Roles containing a selected set of
                                     permissions defined by an
                                     organization.
  ---------------------------------------------------------------------

### 2.3 Principle of least privilege

Grant users only the permissions they need to perform their work.

For example, a developer who only needs to view VM instances should not
receive the Project Owner role.

### 2.4 Service accounts

A service account represents an application, virtual machine, or other
workload rather than an individual person.

Example:

A Cloud Run application needs to access Cloud Storage. Assign a service
account to the Cloud Run service and grant that service account only the
required Storage permissions.

### 2.5 Practical IAM commands

``` bash
# Display the IAM policy for a project
gcloud projects get-iam-policy PROJECT_ID

# Grant the Viewer role to a user at project level
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:developer@example.com" \
  --role="roles/viewer"
```

**Important:** IAM permissions can be inherited through the Google Cloud
resource hierarchy. A project-level role binding can apply to resources
within that project.

------------------------------------------------------------------------

## 3. Google Cloud APIs

A Google Cloud API allows applications and services to interact
programmatically with Google Cloud resources.

Examples:

-   **Compute Engine API:** Manage virtual machines.
-   **Cloud Storage API:** Manage storage buckets and objects.
-   **Cloud Run Admin API:** Deploy and manage Cloud Run services.
-   **BigQuery API:** Query and manage BigQuery resources.

Applications can interact with Google Cloud APIs through:

-   REST APIs
-   Client libraries for languages such as Python, Java, .NET, and Go
-   Google Cloud CLI (`gcloud`)

### 3.1 API enablement

Google Cloud APIs generally need to be enabled in a project before the
corresponding services can be used.

For example, to create virtual machines using Compute Engine, enable the
Compute Engine API.

In some training lab environments, APIs may already be enabled. In
projects created independently, you may need to enable the required APIs
yourself.

### 3.2 Practical commands

``` bash
# List enabled APIs
gcloud services list --enabled

# List available APIs
gcloud services list --available

# Enable Compute Engine API
gcloud services enable compute.googleapis.com

# Enable Cloud Storage API
gcloud services enable storage.googleapis.com

# Enable Pub/Sub API
gcloud services enable pubsub.googleapis.com

# Check whether the Compute Engine API is enabled
gcloud services list --enabled \
  --filter="name:compute.googleapis.com"
```

### 3.3 Important distinction: API enablement vs IAM permissions

Enabling an API does not automatically grant users permission to access
it.

-   **API enablement** makes the API available for use in a project.
-   **IAM permissions** determine which identities can perform specific
    actions.

Enabling an API also does not mean the service is free. Usage may incur
charges.

------------------------------------------------------------------------

## 4. Cloud Pub/Sub

Google Cloud Pub/Sub is a fully managed messaging service that enables
asynchronous communication between applications and services.

It follows a **publish-subscribe messaging model**. Applications do not
need to communicate directly with each other; they exchange messages
through topics and subscriptions.

### 4.1 Main components

  ---------------------------------------------------------------------
  Component                          Description
  ---------------------------------- ----------------------------------
  Publisher                          Application that sends messages.

  Topic                              Named resource to which publishers
                                     send messages.

  Subscription                       Resource through which subscribers
                                     receive messages.

  Subscriber                         Application that receives and
                                     processes messages.

  Message                            Data or event sent by a publisher.

  Acknowledgment (ACK)               Confirmation that a message has
                                     been successfully processed.
  ---------------------------------------------------------------------

### 4.2 How Pub/Sub works

1.  A publisher sends a message to a topic.
2.  Pub/Sub stores and distributes the message.
3.  A subscription makes the message available to its subscribers.
4.  A subscriber receives and processes the message.
5.  The subscriber acknowledges successful processing.

### 4.3 Pull vs. push subscriptions

  ---------------------------------------------------------------------
  Pull subscription                  Push subscription
  ---------------------------------- ----------------------------------
  The subscriber requests messages.  Pub/Sub sends messages to a
                                     configured HTTPS endpoint.

  The subscriber controls when to    Pub/Sub initiates delivery.
  retrieve messages.                 

  Useful for background workers and  Useful for HTTP-based services and
  services that control their        endpoints.
  polling.                           
  ---------------------------------------------------------------------

### 4.4 Real-world implementation example

Consider an employee policy chatbot.

When an employee submits a policy-related request:

1.  The chatbot publishes an event to a Pub/Sub topic.
2.  A subscription makes the event available to a background processing
    service.
3.  The service receives and processes the event.
4.  The result can be stored or forwarded to another service.

This decouples the chatbot from background processing, allowing each
component to operate independently.

### 4.5 Practical implementation using gcloud

``` bash
# Create a topic
gcloud pubsub topics create employee-events

# Create a subscription for the topic
gcloud pubsub subscriptions create employee-events-sub \
  --topic=employee-events

# Publish a message
gcloud pubsub topics publish employee-events \
  --message="Employee policy request submitted"

# Pull messages and automatically acknowledge them
gcloud pubsub subscriptions pull employee-events-sub \
  --auto-ack
```

### 4.6 Important Pub/Sub concepts to learn

-   Message acknowledgment and acknowledgment deadlines
-   Message retention
-   Message ordering
-   Dead-letter topics
-   Retry policies
-   Push and pull delivery
-   Exactly-once delivery, where supported and configured
-   At-least-once delivery
-   IAM permissions for publishers and subscribers

**Important:** Pub/Sub generally provides at-least-once delivery.
Subscriber applications should be designed to handle duplicate messages
safely. Exactly-once delivery has specific supported configurations and
limitations; it should not be assumed to apply to every delivery setup.

------------------------------------------------------------------------



# 8. Google Cloud Compute Engine -- Practical Learning Guide

This chapter expands the Compute Engine lab notes into a practical
guide. It explains the commands, why each step is needed, what happens
behind the scenes, and how the same ideas are used in real projects.

The examples use Linux VMs, Debian, `gcloud`, a frontend/backend
application, and a zonal Managed Instance Group (MIG), as in the
supplied lab notes.

> **Important:** Lab commands often use temporary project IDs, bucket
> names, zones, and simplified settings. Replace these values with the
> ones in your own project. Before running commands that delete, format,
> replace, or expose resources, check the target carefully.

## 8.1 What is Compute Engine?

**Compute Engine** is a Google Cloud service that lets you create and
run virtual machines (VMs) on Google's infrastructure.

A VM behaves like a computer or server. It has:

-   A machine type that determines its virtual CPUs and memory.
-   A boot disk that contains the operating system.
-   Optional additional disks for application data.
-   A network interface for communication.
-   A service account that gives the VM an identity when it accesses
    Google Cloud services.
-   Optional startup scripts that configure the VM when it boots.

You can install software on a VM just as you would on a physical Linux
or Windows server. For example, you can install NGINX, Node.js, .NET,
Java, or a database.

### When would you use a VM?

Use Compute Engine when you need control over the operating system or
the software installed on the server. Examples include:

-   Hosting an application on a Linux or Windows server.
-   Running software that needs operating-system-level access.
-   Running a background worker or scheduled process.
-   Testing server configurations.
-   Hosting software that is not suitable for a fully managed platform.

If you do not need to manage a VM's operating system, a managed service
such as Cloud Run may require less operational work. Compute Engine is
not automatically the best choice for every application.

### Create a basic VM

``` bash
gcloud compute instances create my-vm \
  --zone=asia-east1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

What each part means:

  -----------------------------------------------------------------------
  Part                                Simple explanation
  ----------------------------------- -----------------------------------
  `gcloud compute instances create`   Ask Google Cloud to create a VM.

  `my-vm`                             The name you give the VM.

  `--zone=asia-east1-a`               The zone where the VM will run.

  `--machine-type=e2-medium`          The amount/type of virtual CPU and
                                      memory allocated to it.

  `--image-family=debian-12`          Select a Debian 12 operating-system
                                      image family.

  `--image-project=debian-cloud`      Select the project that publishes
                                      the official Debian images.
  -----------------------------------------------------------------------

Useful commands:

``` bash
# List VMs in the selected project and zone context
gcloud compute instances list

# Display details about one VM
gcloud compute instances describe my-vm --zone=asia-east1-a

# Stop a VM
gcloud compute instances stop my-vm --zone=asia-east1-a

# Start a stopped VM
gcloud compute instances start my-vm --zone=asia-east1-a

# Delete a VM
gcloud compute instances delete my-vm --zone=asia-east1-a
```

Stopping a VM is different from deleting it. A stopped VM can be started
again. Its attached persistent disks generally remain, although some
resources and storage can still incur charges. Deleting a VM removes the
VM resource; check the disk deletion settings before assuming its disks
will also be deleted.

## 8.2 Cloud Shell, a VM terminal, and SSH

These three terms are related, but they do not mean the same thing.

### What is a terminal?

A terminal is a text-based window where you type commands and read the
output. It is a way to interact with a computer using commands instead
of clicking through a graphical interface.

For example:

``` bash
pwd
ls
whoami
```

-   `pwd` shows the current directory.
-   `ls` lists files and folders.
-   `whoami` shows the current Linux user.

### What is SSH?

**SSH means Secure Shell.** It is a secure network protocol that lets
you log in to another computer and run commands on it.

Think of it this way:

-   Your laptop is one computer.
-   A Linux VM in Google Cloud is another computer.
-   SSH creates a secure connection from your laptop or Cloud Shell to
    that VM.
-   Once connected, the commands you type run on the VM, not on your
    laptop.

SSH is commonly used by developers and system administrators to:

-   Install software on a server.
-   Check whether an application is running.
-   Read application logs.
-   Edit configuration files.
-   Restart services.
-   Inspect files and disk space.
-   Troubleshoot a server without physically accessing it.

For example, when you run `sudo apt-get install nginx` inside an SSH
session connected to a VM, NGINX is installed on that VM.

### What is Cloud Shell?

Cloud Shell is a browser-based command-line environment provided by
Google Cloud. It runs in a Google-managed environment and includes tools
such as the Google Cloud CLI (`gcloud`) and Git. The lab notes describe
a persistent 5-GB home directory.

You can use Cloud Shell to manage Google Cloud resources without
installing the Google Cloud CLI on your own computer.

For example, run this in Cloud Shell:

``` bash
gcloud compute instances list
```

This command asks the Google Cloud APIs to list VM instances. It does
not log in to any VM.

### What is the SSH terminal opened from the VM page?

When you click **SSH** beside a VM in the Google Cloud Console, Google
opens a browser-based terminal connected to that specific VM.

Commands entered in that window run inside the VM.

For example:

``` bash
sudo apt-get update
sudo apt-get install -y nginx
```

These commands update the package information and install NGINX on the
VM.

### Cloud Shell vs. SSH terminal connected to a VM

  -----------------------------------------------------------------------
  Feature                 Cloud Shell             VM SSH terminal
  ----------------------- ----------------------- -----------------------
  What is it?             A Google-managed        A terminal session
                          command-line            connected to a
                          environment.            particular VM.

  Where do commands run?  In the Cloud Shell      Inside the target VM.
                          environment.            

  Main purpose            Manage Google Cloud     Administer and
                          resources using tools   troubleshoot the VM.
                          such as `gcloud`.       

  `gcloud` available?     Yes, it is              Not necessarily; it can
                          preinstalled.           be installed if needed.

  Installs NGINX where?   In Cloud Shell, if you  On the connected VM.
                          run the install there.  

  Can it connect to a VM? Yes, using              It is already connected
                          `gcloud compute ssh`.   to that VM.
  -----------------------------------------------------------------------

A useful daily workflow is:

1.  Open Cloud Shell.
2.  Use `gcloud` to create or inspect a VM.
3.  Use `gcloud compute ssh` to connect to the VM.
4.  Run Linux commands inside the VM.
5.  Type `exit` to leave the VM session and return to Cloud Shell.

### Connect to a VM using SSH

From Cloud Shell:

``` bash
gcloud compute ssh gcelab2 \
  --zone=asia-east1-a
```

Explanation:

-   `gcloud compute ssh` asks Google Cloud to establish an SSH
    connection.
-   `gcelab2` is the VM name.
-   `--zone` identifies where that VM is located.

If the active project and default zone are configured in `gcloud`, you
may be able to omit those flags.

To leave the remote VM terminal:

``` bash
exit
```

You return to the shell from which you started the SSH connection.

### Why do we need SSH when attaching a disk?

Attaching a disk in Google Cloud connects the disk resource to the VM.
It does not automatically prepare the disk for storing files inside
Linux.

After attaching it, you normally need to work inside the VM to:

1.  Find the new disk device.
2.  Create a filesystem on it.
3.  Create a directory to use as its mount point.
4.  Mount the filesystem to that directory.
5.  Optionally configure automatic mounting after a restart.

Those are operating-system tasks, so you use SSH to run the Linux
commands inside the VM.

You do **not** need SSH merely to issue the Google Cloud command that
attaches the disk. That attachment command can be run from Cloud Shell.

## 8.3 SSH keys and password prompts

### What is an SSH key?

An SSH key is a digital credential used to prove that you are allowed to
connect to a server.

SSH normally uses a pair of mathematically related keys:

  -----------------------------------------------------------------------
  Key                     Meaning                 Where it should be kept
  ----------------------- ----------------------- -----------------------
  Private key             The secret part used by Keep it private on your
                          your SSH client to      computer or approved
                          prove your identity.    key store.

  Public key              The part that can be    Can be shared with the
                          given to the server or  system that manages
                          an identity service.    access.
  -----------------------------------------------------------------------

A simple analogy:

-   The public key is like a lock that can be placed on the server.
-   The private key is the secret key that proves you are allowed to
    open that lock.

The public and private keys are related. A server can verify proof made
with the private key using the corresponding public key.

**Never share your private key.** Do not upload it to a source-code
repository, send it in email, or place it in a public Cloud Storage
bucket.

### Does Google Cloud create SSH keys automatically?

When you connect using the Google Cloud Console or `gcloud compute ssh`,
Compute Engine can create and manage SSH keys for you. The exact process
depends on whether the VM uses metadata-based SSH keys or OS Login.

For many beginner labs, you do not need to manually create a key before
using the Console's SSH button or `gcloud compute ssh`.

### Why is SSH asking for a password?

There are several different prompts that can look like password
requests:

1.  **SSH key passphrase:** Your computer may ask for a passphrase to
    unlock your encrypted private key. This protects the key file. It is
    not necessarily the VM's Linux password.
2.  **Google account sign-in or authorization:** The browser or `gcloud`
    may ask you to authenticate to Google Cloud.
3.  **Remote Linux password:** A terminal may display
    `username@host's password:`. This is asking for the remote Linux
    account password. Many Compute Engine Linux images do not have a
    password configured by default and use key-based SSH authentication
    instead.
4.  **`sudo` password:** Some Linux environments ask for a password when
    a user runs a privileged command. This is separate from SSH
    authentication.

Do not assume that every password prompt is asking for your Google
account password. Read the prompt carefully. Do not repeatedly guess
passwords.

### OS Login

OS Login is a Google Cloud feature that connects VM login access to
Google identities and IAM permissions. It can make access management
easier, especially when multiple developers need access.

Two useful roles are:

-   `roles/compute.osLogin`: permits login without `sudo` access.
-   `roles/compute.osAdminLogin`: permits login with `sudo` access.

OS Login is for Linux VMs. When it is enabled, metadata-based SSH keys
are not used for normal OS Login access.

### SSH in daily development

A typical workflow might look like this:

``` bash
# Connect to the VM
gcloud compute ssh my-vm --zone=asia-east1-a

# Check the current user
whoami

# Check whether NGINX is running
sudo systemctl status nginx

# View recent system logs for NGINX
sudo journalctl -u nginx -n 50

# Leave the VM
exit
```

Use SSH for administration and troubleshooting. For repeatable
application deployments, prefer automated deployment methods such as
startup scripts, instance templates, managed instance groups, or a CI/CD
pipeline rather than manually changing every VM.

## 8.4 Linux commands used in the lab

The lab uses Debian Linux. These commands run inside the VM's SSH
terminal unless stated otherwise.

### Update package information

``` bash
sudo apt-get update
```

-   `sudo` runs the command with administrator privileges.
-   `apt-get` is Debian's package-management command.
-   `update` downloads the latest package lists.
-   It does not upgrade every installed package by itself.

### Install NGINX

``` bash
sudo apt-get install -y nginx
```

-   `install nginx` asks Debian to install the NGINX web server.
-   `-y` automatically answers yes to confirmation prompts.
-   NGINX can serve web pages and act as a reverse proxy.

### Check whether NGINX is running

The lab uses:

``` bash
ps auwx | grep nginx
```

This searches the process list for processes whose command contains
`nginx`. The `grep` process itself can also appear in the output.

A more direct service check is:

``` bash
sudo systemctl status nginx
```

To test whether a local web server responds:

``` bash
curl http://localhost
```

`localhost` means the same VM. This test does not prove that the
application is reachable from the public internet; firewall rules and
external networking must also be configured.

## 8.5 Regions, zones, and resource locations

A **region** is a geographical area where Google Cloud has data centers.
A region contains one or more **zones**.

Example from the lab:

-   Region: `asia-east1`
-   Zone: `asia-east1-a`

A zone is a deployment location inside a region. A VM instance is a
zonal resource: it is created in a particular zone.

The location of a resource matters because some resources must be in
compatible locations to work together.

Examples:

-   A zonal VM and a zonal Persistent Disk that you want to attach to it
    must be in the same zone.
-   A regional static external IP address must be in the same region as
    the VM that uses it.
-   A global load balancer has a different scope from a zonal VM.

### Set the default region and zone

``` bash
gcloud config set compute/region asia-east1
gcloud config set compute/zone asia-east1-a
```

These commands set default properties for the Google Cloud CLI. They do
not move existing resources and do not change the physical location of a
project.

### Use shell variables

``` bash
export REGION=asia-east1
export ZONE=asia-east1-a
```

Now you can reuse the values:

``` bash
gcloud compute instances create demo-vm \
  --zone="$ZONE" \
  --machine-type=e2-medium
```

`export` creates an environment variable for the current shell and its
child processes. If you open a new shell session, you may need to define
it again.

This is different from `gcloud config set`, which stores a CLI
configuration property. In training labs, the project or shell
environment may be reset between sessions, so check the active
configuration rather than assuming it is unchanged.

### Check the active account and project

``` bash
# Show authenticated accounts
gcloud auth list

# Show the active project configuration
gcloud config list project

# Show the active project ID only
gcloud config get-value project
```

Before running commands in a lab, confirm that the active account and
project are the ones you intend to use.

## 8.6 Create and attach a Persistent Disk

### What is a Persistent Disk?

A Persistent Disk is durable block storage that a VM can use like a disk
attached to a computer.

It is useful when you want data to remain available independently of the
VM's running state. For example, an application can store uploaded files
or application data on a separate data disk.

A Persistent Disk is network-attached storage. It is not the same as a
physical disk directly installed inside the server hosting the VM.

### Why create a separate disk?

A separate data disk can make it easier to manage application data
independently from the operating system.

For example:

-   The boot disk contains Debian and installed software.
-   A separate data disk stores application files.
-   You can manage, snapshot, or attach the data disk separately,
    subject to disk and VM compatibility rules.

### Step 1: Create a VM

The lab creates a Debian VM:

``` bash
gcloud compute instances create gcelab \
  --zone="$ZONE" \
  --machine-type=e2-standard-2 \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

This creates a VM named `gcelab` in the selected zone.

### Step 2: Create a Persistent Disk

``` bash
gcloud compute disks create mydisk \
  --size=200GB \
  --zone="$ZONE"
```

Explanation:

-   `disks create` creates a disk resource.
-   `mydisk` is the disk name.
-   `--size=200GB` requests a 200-GB disk.
-   `--zone` places the disk in a specific zone.

The disk is created, but it is not yet attached to the VM.

### Step 3: Attach the disk to the VM

``` bash
gcloud compute instances attach-disk gcelab \
  --disk=mydisk \
  --zone="$ZONE"
```

This connects the disk resource to the VM. Linux should then be able to
detect it as a block device.

### Step 4: Connect to the VM

``` bash
gcloud compute ssh gcelab --zone="$ZONE"
```

You need this connection because the next steps prepare the disk inside
Linux.

### Step 5: Find the new disk

``` bash
ls -l /dev/disk/by-id/
```

You can also inspect disks and filesystems with:

``` bash
lsblk -f
```

-   `/dev` contains device files representing hardware and virtual
    devices.
-   `/dev/disk/by-id/` provides device names based on disk identity.
-   `lsblk` lists block devices.
-   `-f` shows filesystem information where available.

Check the output carefully and identify the new data disk. Do not guess
the device name: selecting the wrong device can destroy data.

### Step 6: Create a mount-point directory

``` bash
sudo mkdir -p /mnt/mydisk
```

A **mount point** is a directory where Linux makes the contents of a
filesystem accessible.

After mounting the disk at `/mnt/mydisk`, you can work with its files
through that directory.

### Step 7: Format the disk

The lab uses `mkfs.ext4`:

``` bash
sudo mkfs.ext4 -F \
  /dev/disk/by-id/DEVICE_ID
```

Replace `DEVICE_ID` with the actual device identifier found on your VM.

Explanation:

-   `mkfs` means make filesystem.
-   `ext4` is a common Linux filesystem format.
-   `-F` forces the operation.

**Warning: Formatting creates a filesystem and destroys existing data on
the selected device.** Only format a new or intentionally disposable
disk. Verify the device name before running this command.

### Step 8: Mount the disk

``` bash
sudo mount -o defaults \
  /dev/disk/by-id/DEVICE_ID \
  /mnt/mydisk
```

This makes the disk's filesystem available at `/mnt/mydisk`.

The lab also uses the `discard` mount option:

``` bash
sudo mount -o discard,defaults \
  /dev/disk/by-id/DEVICE_ID \
  /mnt/mydisk
```

`discard` enables discard/TRIM requests where supported. It is not
required for every disk or workload.

Verify the mount:

``` bash
df -h
lsblk -f
```

`df -h` displays mounted filesystem capacity in a human-readable format.

### Step 9: Make the mount persistent across VM restarts

A manual `mount` command does not automatically create a permanent mount
configuration. Linux uses `/etc/fstab` to define filesystems that should
be mounted during startup.

Find the filesystem UUID:

``` bash
sudo blkid
```

Open the file:

``` bash
sudo nano /etc/fstab
```

Add an entry using the correct UUID and mount point. Example:

``` text
UUID=YOUR-FILESYSTEM-UUID /mnt/mydisk ext4 defaults,nofail 0 2
```

What the fields mean:

1.  `UUID=...`: identifies the filesystem.
2.  `/mnt/mydisk`: directory where it should be mounted.
3.  `ext4`: filesystem type.
4.  `defaults,nofail`: standard mount options; `nofail` lets the VM
    continue booting if the disk is unavailable.
5.  `0`: do not use the legacy dump backup utility.
6.  `2`: filesystem check order for this non-root filesystem.

Test the entry before rebooting:

``` bash
sudo umount /mnt/mydisk
sudo mount -a
df -h
```

If `mount -a` reports an error, fix `/etc/fstab` before restarting. A
bad mount entry can cause boot problems.

### Why does the lab use SSH for disk setup?

The Google Cloud command attaches the disk to the VM at the
infrastructure level. Linux still needs to recognize, format, and mount
the device. SSH provides access to the VM so you can perform those
operating-system steps.

## 8.7 Persistent Disk vs. Local SSD

The lab introduces Local SSD as another storage option.

### Persistent Disk

-   Durable network block storage.
-   Designed to keep data beyond the VM's normal stop/start lifecycle.
-   Can be managed separately from the VM.
-   Useful for application data that must be retained.

### Local SSD

-   High-performance temporary block storage physically attached to the
    host machine running the VM.
-   Offers very low latency and high I/O performance.
-   Intended for temporary or reconstructable data such as caches,
    scratch data, or temporary processing files.
-   Data can be lost in failure or lifecycle scenarios; do not treat it
    as the only copy of important data.

  -----------------------------------------------------------------------
  Feature                 Persistent Disk         Local SSD
  ----------------------- ----------------------- -----------------------
  Main purpose            Durable storage         Very fast temporary
                                                  storage

  Attachment              Network-attached block  Physically attached to
                          storage                 the host

  Data durability         Designed for durable    Not a durable data
                          storage                 store

  Typical use             Application data, boot  Cache, scratch data,
                          disks, retained files   temporary processing

  Main trade-off          Network storage         High performance with
                          characteristics         data-loss risk
  -----------------------------------------------------------------------

**Important correction:** Local SSD data is not guaranteed to disappear
on every VM reboot. However, it can be lost in certain stop,
termination, host-failure, or other lifecycle scenarios. Its exact
persistence behavior depends on the VM configuration and event. Always
keep important data on durable storage and consult the current Local SSD
documentation before relying on a specific persistence behavior.

## 8.8 Cloud Storage buckets and why the lab copies code there

### What is Cloud Storage?

Cloud Storage is an object storage service. It stores objects (files)
inside buckets.

A bucket is a container for objects. For example, a bucket can hold:

-   Application build files.
-   Startup scripts.
-   Images and documents.
-   Logs or exported files.
-   Deployment artifacts.

A Cloud Storage bucket is not the same as a VM disk. You access objects
through Cloud Storage APIs and tools rather than mounting the bucket as
a normal local disk by default.

### Why copy application files to a bucket?

In the lab, the frontend and backend VMs need to obtain the application
code and startup script.

Instead of manually copying files separately to every VM:

1.  Store the application files in one bucket.
2.  Store the startup script in the bucket.
3.  Configure each VM to retrieve the required files when it starts.
4.  When a new VM is created by a MIG, it can use the same deployment
    process.

This helps because:

-   Multiple VMs can retrieve the same deployment artifact.
-   The code is stored independently of an individual VM.
-   Recreated instances can be configured automatically.
-   You avoid manually uploading the same files to each VM.

A bucket does not automatically make a deployment secure or repeatable.
Give VMs only the permissions they need, control who can change the
artifacts, and use versioned or immutable build artifacts for production
deployments.

### Create a bucket

``` bash
gcloud storage buckets create gs://YOUR_UNIQUE_BUCKET_NAME
```

Bucket names must be globally unique. In real projects, choose the
location and other bucket settings to match your data residency,
availability, and cost requirements.

### Copy a file to the bucket

``` bash
gcloud storage cp startup-script.sh \
  gs://YOUR_BUCKET_NAME/startup-script.sh
```

### Copy a directory to the bucket

``` bash
gcloud storage cp -r monolith-to-microservices \
  gs://YOUR_BUCKET_NAME/
```

-   `cp` copies files.
-   `-r` copies a directory and its contents recursively.
-   `gs://` identifies a Cloud Storage location.

### Why remove `node_modules` before copying?

The lab removes Node.js dependency folders before uploading:

``` bash
rm -rf monolith-to-microservices/*/node_modules
```

`node_modules` can contain many files and can be large. The lab wants to
avoid uploading those existing local dependencies. The startup process
installs dependencies on the VM.

**Warning:** `rm -rf` deletes files without asking for confirmation.
Check the directory and pattern carefully before running it. Never run a
destructive command with an uncertain path.

## 8.9 Startup scripts

### What is a startup script?

A startup script is a file containing commands that Compute Engine runs
when a VM boots.

It can automate initial VM configuration, for example:

-   Install NGINX or Node.js.
-   Download application files.
-   Install application dependencies.
-   Create configuration files.
-   Start application processes.

Without a startup script, an administrator might need to connect to
every new VM and repeat these steps manually.

### Why use a startup script with a MIG?

A MIG can create replacement VMs whenever it needs to maintain the
desired number of instances. If each VM needs the same software and
configuration, a startup script can configure each new VM in the same
way.

The intended sequence is:

1.  A VM starts.
2.  The guest environment reads the startup-script metadata.
3.  The startup script runs.
4.  The script downloads the application artifacts.
5.  The script installs dependencies and configures the application.
6.  The application starts.

The guest environment on the VM is responsible for running
metadata-based startup scripts. Public Compute Engine images normally
include the required guest environment.

### Create a startup-script file

In Cloud Shell:

``` bash
mkdir -p ~/monolith-to-microservices
touch ~/monolith-to-microservices/startup-script.sh
```

Open the Cloud Shell Editor and add the commands needed to prepare and
start the application.

A simplified illustrative script might look like this:

``` bash
#!/bin/bash
set -e

apt-get update
apt-get install -y nginx

# Add application download, dependency installation,
# configuration, and service-start commands here.
```

This is only a small example. A real application startup script must
include the correct download location, runtime installation, dependency
installation, configuration, and process management steps.

### Upload the startup script

``` bash
gcloud storage cp \
  ~/monolith-to-microservices/startup-script.sh \
  gs://YOUR_BUCKET_NAME/startup-script.sh
```

### Create a VM that uses the startup script

The lab uses metadata:

``` bash
gcloud compute instances create backend \
  --zone=us-west1-b \
  --machine-type=e2-standard-2 \
  --tags=backend \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --metadata=startup-script-url=https://storage.googleapis.com/YOUR_BUCKET_NAME/startup-script.sh
```

Important parts:

-   `--tags=backend` adds a network tag to the VM. Firewall rules can
    target VMs with this tag.
-   `--metadata=startup-script-url=...` tells the VM where to find the
    startup script.
-   The startup script runs when the VM boots; it is not simply a
    command that runs in Cloud Shell.

### Important permissions

If a startup script is stored in a private Cloud Storage bucket, the VM
must be able to read it. Assign an appropriate service account to the VM
and grant it the Storage Object Viewer role on the bucket (or a narrower
equivalent permission where appropriate).

Do not make a bucket public just to make a startup script work.

### Production best practices for startup scripts

-   Make scripts idempotent: running them more than once should not
    damage the system or create duplicate configuration.
-   Handle errors so a failed installation does not silently look
    successful.
-   Pin application and dependency versions.
-   Do not put passwords, API keys, or other secrets directly in the
    script or plain metadata.
-   Use Secret Manager or another approved secret-management approach.
-   Log script progress and check startup-script logs when
    troubleshooting.
-   Prefer immutable, versioned application artifacts instead of
    downloading an unspecified "latest" build.

## 8.10 Network tags and firewall rules

### What is a network tag?

A network tag is a label attached to a VM. Firewall rules can use target
tags to select which VMs a rule applies to.

For example:

-   Frontend VMs receive the `frontend` tag.
-   Backend VMs receive the `backend` tag.

A tag does not itself open a port. A matching firewall rule is needed.

### Create a firewall rule for the frontend

The lab allows TCP port 8080 to VMs tagged `frontend`:

``` bash
gcloud compute firewall-rules create fw-fe \
  --allow=tcp:8080 \
  --target-tags=frontend
```

This creates an ingress firewall rule allowing traffic to TCP port 8080
on VMs with the `frontend` tag, subject to the rule's network and source
settings.

### Create a firewall rule for the backend

``` bash
gcloud compute firewall-rules create fw-be \
  --allow=tcp:8081-8082 \
  --target-tags=backend
```

This allows traffic to TCP ports 8081 and 8082 on VMs tagged `backend`,
subject to the rule's network and source settings.

### Why use different tags and ports?

The example separates the frontend and backend roles.

-   The frontend application listens on port 8080.
-   The backend services listen on ports 8081 and 8082.
-   Firewall rules can target the appropriate group of VMs.

This follows the idea of limiting network access to only the services
that need it.

**Security note:** The lab commands do not specify `--source-ranges`, so
the rule's source scope follows the applicable firewall-rule defaults.
For production, explicitly decide which sources should be allowed. Do
not expose internal backend ports to the entire internet unless there is
a clear requirement.

## 8.11 Instance templates

### What is an instance template?

An instance template is a reusable definition of a VM's configuration.

It can describe settings such as:

-   Machine type.
-   Boot disk image.
-   Network and subnet.
-   Network tags.
-   Service account.
-   Metadata and startup script.
-   Other VM configuration.

A template is a blueprint. It is not itself a running VM.

### Why do we need an instance template?

Suppose you need four identical frontend VMs. Creating each one manually
can lead to differences in machine type, operating system, tags, or
startup configuration.

An instance template records the configuration once. A Managed Instance
Group can then use it to create matching VM instances.

### Create a template from an existing VM

The lab first stops the frontend and backend VMs, then creates templates
from them:

``` bash
gcloud compute instances stop frontend --zone=us-west1-b
gcloud compute instances stop backend --zone=us-west1-b
```

``` bash
gcloud compute instance-templates create fancy-fe \
  --source-instance=frontend \
  --source-instance-zone=us-west1-b

gcloud compute instance-templates create fancy-be \
  --source-instance=backend \
  --source-instance-zone=us-west1-b
```

The first command creates a template based on the frontend VM. The
second creates a template based on the backend VM.

You can also define a template directly with flags rather than deriving
it from an existing VM. This is often easier to reproduce and review in
production.

### List templates

``` bash
gcloud compute instance-templates list
```

### Important: templates are immutable

After an instance template is created, you cannot edit that template in
place.

If you need to change its machine type, startup script, image, or other
settings:

1.  Create a new instance template.
2.  Tell the MIG to use the new template.
3.  Roll out the update to existing instances using an appropriate
    update policy.

This is why the lab creates `fancy-fe-new` rather than editing
`fancy-fe`.

## 8.12 Managed Instance Groups (MIGs)

### What is a Managed Instance Group?

A Managed Instance Group is a group of VM instances that Compute Engine
manages as one unit.

The instances are created from an instance template. The group can
maintain a target number of VMs and can support:

-   Automatic scaling.
-   Autohealing and repair.
-   Rolling updates.
-   Integration with load balancing.

### Why use a MIG?

Imagine that your application needs two frontend VMs.

If one VM fails, you do not want to manually create a replacement. A MIG
can detect the failure through its configured repair/autohealing
mechanism and recreate the VM.

If traffic increases, autoscaling can add instances, within the
configured limits.

### Create a zonal frontend MIG

``` bash
gcloud compute instance-groups managed create fancy-fe-mig \
  --zone=us-west1-b \
  --base-instance-name=fancy-fe \
  --size=2 \
  --template=fancy-fe
```

Explanation:

  -----------------------------------------------------------------------
  Option                              Meaning
  ----------------------------------- -----------------------------------
  `instance-groups managed create`    Create a Managed Instance Group.

  `fancy-fe-mig`                      Name of the group.

  `--zone=us-west1-b`                 Zone in which this zonal MIG
                                      operates.

  `--base-instance-name=fancy-fe`     Prefix used for generated VM names.

  `--size=2`                          Start with two VM instances.

  `--template=fancy-fe`               Use the `fancy-fe` instance
                                      template.
  -----------------------------------------------------------------------

Create the backend MIG:

``` bash
gcloud compute instance-groups managed create fancy-be-mig \
  --zone=us-west1-b \
  --base-instance-name=fancy-be \
  --size=2 \
  --template=fancy-be
```

### List MIGs and their instances

``` bash
# List instance groups
gcloud compute instance-groups list

# List managed instance groups
gcloud compute instance-groups managed list

# List VMs in one MIG
gcloud compute instance-groups managed list-instances fancy-fe-mig \
  --zone=us-west1-b
```

### Zonal vs. regional MIG

-   A zonal MIG manages VMs in one zone.
-   A regional MIG can distribute VMs across multiple zones in a region.

A regional MIG can help reduce the impact of a single-zone failure,
provided the application and its dependencies are designed for that
distribution.

The supplied lab uses zonal MIGs. Do not confuse a regional MIG with a
global resource: a regional MIG is still associated with one Google
Cloud region.

## 8.13 Named ports

### What is a named port?

A named port maps a readable name to a port number on an instance group.

For example:

-   `frontend:8080`
-   `orders:8081`
-   `products:8082`

A backend service can refer to a named port instead of requiring the
port number to be repeated in every related configuration.

### Configure named ports

For the backend MIG:

``` bash
gcloud compute instance-groups set-named-ports fancy-be-mig \
  --zone=us-west1-b \
  --named-ports=orders:8081,products:8082
```

For the frontend MIG:

``` bash
gcloud compute instance-groups set-named-ports fancy-fe-mig \
  --zone=us-west1-b \
  --named-ports=frontend:8080
```

The group name must be the actual MIG name. In this example, it is
`fancy-fe-mig`, not the template name `fancy-fe`.

## 8.14 Health checks and autohealing

### What is a health check?

A health check is a test that Google Cloud performs to see whether an
application or VM is responding.

For an HTTP health check, Google Cloud sends an HTTP request to a
configured port and, optionally, a path.

For example:

-   Port: 8080
-   Path: `/`
-   Expected result: a successful response from the application

A VM can be running while its application is broken. A health check
helps detect this difference.

### Load-balancing health checks vs. autohealing health checks

These two health-check uses have different purposes.

  -----------------------------------------------------------------------
  Load-balancing health check         MIG autohealing health check
  ----------------------------------- -----------------------------------
  Determines whether a backend should Determines whether a VM should be
  receive user traffic.               repaired or recreated.

  Helps route traffic away from       Helps restore the group to a
  unhealthy backends.                 healthy state.

  Should detect traffic-serving       Should be conservative enough to
  problems quickly, with suitable     avoid replacing a VM because of a
  thresholds.                         brief or temporary problem.
  -----------------------------------------------------------------------

An unhealthy load-balancer backend is not automatically deleted. The
load balancer can stop sending it traffic. A MIG autohealing policy can
trigger repair when its configured health check reports a failure for
long enough.

### Create an HTTP health check for the frontend

``` bash
gcloud compute health-checks create http fancy-fe-hc \
  --port=8080 \
  --check-interval=30s \
  --healthy-threshold=1 \
  --timeout=10s \
  --unhealthy-threshold=3
```

Explanation:

-   `--port=8080`: send the health-check request to port 8080.
-   `--check-interval=30s`: run checks at 30-second intervals.
-   `--timeout=10s`: wait up to 10 seconds for a response.
-   `--healthy-threshold=1`: one successful check is enough to mark it
    healthy under this configuration.
-   `--unhealthy-threshold=3`: three consecutive failures are needed to
    mark it unhealthy under this configuration.

Create a backend health check:

``` bash
gcloud compute health-checks create http fancy-be-hc \
  --port=8081 \
  --request-path=/api/orders \
  --check-interval=30s \
  --healthy-threshold=1 \
  --timeout=10s \
  --unhealthy-threshold=3
```

The application must actually respond successfully on the configured
port and path. A health check cannot make an application healthy if the
application is not listening or the path returns an error.

### Allow health-check traffic through the firewall

The lab uses:

``` bash
gcloud compute firewall-rules create allow-health-check \
  --allow=tcp:8080-8081 \
  --source-ranges=130.211.0.0/22,35.191.0.0/16 \
  --network=default
```

This creates an ingress rule allowing TCP traffic from the listed Google
health-check source ranges to the selected network. The exact source
ranges and firewall requirements depend on the load-balancer and
health-check configuration. Check the documentation for the
load-balancer type you deploy.

### Apply an autohealing policy to a MIG

``` bash
gcloud compute instance-groups managed update fancy-fe-mig \
  --zone=us-west1-b \
  --health-check=fancy-fe-hc \
  --initial-delay=300
```

``` bash
gcloud compute instance-groups managed update fancy-be-mig \
  --zone=us-west1-b \
  --health-check=fancy-be-hc \
  --initial-delay=300
```

`--initial-delay=300` gives a newly created VM time to start its
application before autohealing health checks are used to decide whether
the instance needs repair. Choose a delay that reflects the real startup
time of the application.

## 8.15 Load balancing

### What is a load balancer?

A load balancer receives requests and distributes them across multiple
backend servers.

Imagine that 1,000 users visit your application at the same time. If all
requests go to one VM, that VM may become overloaded.

A load balancer can distribute requests across several healthy VMs. If
one backend is unhealthy, the load balancer can avoid sending new
traffic to it.

### Why use a load balancer?

-   Distributes traffic across multiple VM instances.
-   Provides a common frontend IP address.
-   Works with health checks to avoid unhealthy backends.
-   Supports routing rules, such as sending `/api/orders` to one backend
    and `/api/products` to another.
-   Can work with managed instance groups as backends.

### Main components of an external HTTP load balancer

The lab's HTTP load-balancer flow can be understood as:

**Client → Forwarding rule → Target HTTP proxy → URL map → Backend
service → Healthy backend VM(s)**

  -----------------------------------------------------------------------
  Component                           Simple explanation
  ----------------------------------- -----------------------------------
  Forwarding rule                     Receives traffic on an IP address
                                      and port, such as TCP port 80.

  Target HTTP proxy                   Receives HTTP requests and uses the
                                      URL map to decide where they should
                                      go.

  URL map                             Contains routing rules, such as
                                      path-based rules.

  Backend service                     Defines the backend group(s),
                                      health checks, and traffic-related
                                      settings.

  Backend                             The actual serving resources, such
                                      as a MIG.

  Health check                        Tests whether the backend
                                      application is responding.
  -----------------------------------------------------------------------

### Example of path-based routing

Suppose one public application uses these URLs:

  Request path      Destination
  ----------------- ------------------
  `/`               Frontend service
  `/api/orders`     Orders service
  `/api/products`   Products service

The URL map checks the request path and selects the matching backend
service.

This lets users access different application components through one
public load-balancer IP address.

### Important gap in the supplied lab commands

Your notes create the frontend backend service named
`fancy-fe-frontend`, but the path matcher refers to `fancy-be-orders`
and `fancy-be-products`.

Those two backend services must also exist before the path matcher can
successfully refer to them. The supplied excerpt does not include their
creation commands. Treat this as a missing step in the notes, not as a
command that can be skipped.

A backend service generally needs: - A health check. - A named port that
matches the instance group's named port. - The correct instance group
added as a backend.

### Create the frontend health check

``` bash
gcloud compute health-checks create http fancy-fe-frontend-hc \
  --request-path=/ \
  --port=8080
```

This health check requests `/` on port 8080.

### Create a frontend backend service

``` bash
gcloud compute backend-services create fancy-fe-frontend \
  --health-checks=fancy-fe-frontend-hc \
  --port-name=frontend \
  --global
```

This creates a global backend service that uses the `frontend` named
port and the frontend health check.

### Add the frontend MIG as a backend

``` bash
gcloud compute backend-services add-backend fancy-fe-frontend \
  --instance-group=fancy-fe-mig \
  --instance-group-zone=us-west1-b \
  --global
```

This tells the backend service to use the frontend MIG's VM instances as
backends.

The backend services for orders and products need equivalent
configuration, using the appropriate health checks, named ports, and
backend group. Create those services before adding path rules that
reference them.

### Create a URL map

``` bash
gcloud compute url-maps create fancy-map \
  --default-service=fancy-fe-frontend
```

The default service handles requests that do not match a more specific
rule.

### Add path-based rules

Once the orders and products backend services have been created:

``` bash
gcloud compute url-maps add-path-matcher fancy-map \
  --default-service=fancy-fe-frontend \
  --path-matcher-name=api-routes \
  --path-rules="/api/orders=fancy-be-orders,/api/products=fancy-be-products"
```

The two named backend services must exist. The URL map sends requests
for those paths to the corresponding service.

### Create a target HTTP proxy

``` bash
gcloud compute target-http-proxies create fancy-proxy \
  --url-map=fancy-map
```

The target HTTP proxy connects the HTTP load-balancing frontend to the
URL map.

### Create a global forwarding rule

``` bash
gcloud compute forwarding-rules create fancy-http-rule \
  --global \
  --target-http-proxy=fancy-proxy \
  --ports=80
```

This creates a global forwarding rule that directs HTTP traffic on port
80 to the target HTTP proxy.

List the forwarding rules:

``` bash
gcloud compute forwarding-rules list --global
```

Use the resulting frontend IP address to test the load balancer.

**Production note:** This is an HTTP example. For public production
applications, HTTPS is normally recommended. An HTTPS load balancer uses
a target HTTPS proxy and a certificate, and generally receives client
HTTPS traffic on port 443.

## 8.16 SSL/TLS certificates

### What is an SSL certificate?

An SSL certificate is a digital certificate used to prove the identity
of a website or server.

The modern protocol is TLS (Transport Layer Security). People still
commonly say "SSL certificate," but current secure HTTPS connections use
TLS.

### Why is a certificate needed?

When a user opens:

``` text
https://example.com
```

the browser needs to establish an encrypted connection to the correct
website.

A certificate helps the browser:

1.  Check that the server is authorized to represent the domain.
2.  Verify the server's identity through the certificate chain.
3.  Establish encryption keys for the secure connection.
4.  Protect data sent between the browser and the server from being read
    or modified by an attacker on the network.

A certificate is not a replacement for application authentication. It
secures the connection and helps authenticate the server's identity.

### HTTP vs. HTTPS

  -----------------------------------------------------------------------
  HTTP                                HTTPS
  ----------------------------------- -----------------------------------
  Does not encrypt the HTTP           Protects the connection using TLS.
  connection by itself.               

  Commonly uses port 80.              Commonly uses port 443.

  Not suitable for sensitive public   Recommended for public production
  application traffic without another web applications.
  secure transport layer.             
  -----------------------------------------------------------------------

### How this relates to the load balancer

-   An HTTP load balancer uses a target HTTP proxy.
-   An HTTPS load balancer uses a target HTTPS proxy and a certificate
    configuration.
-   The certificate is presented to the client during the TLS handshake.
-   The load balancer then routes the request according to its URL map.

The lab uses HTTP to avoid certificate setup during the exercise. That
simplifies the lab but is not a production security recommendation.

## 8.17 Rolling updates and replacing instances

### What is a rolling update?

A rolling update changes the VM instances in a MIG gradually or
according to a configured update policy.

For example, suppose a frontend MIG has four VMs. You create a new
instance template with an updated application configuration. A rolling
update can replace the old VMs with new VMs in a controlled way.

This avoids manually changing each VM.

### Rolling replace command in the lab

``` bash
gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
  --zone=us-west1-b \
  --max-unavailable=100%
```

This tells the MIG to replace instances. The `--max-unavailable=100%`
setting allows all instances to be unavailable during the operation.

**Important:** This setting can cause downtime. It is used in the lab
for speed, not as a safe production default. For production, choose
`max-unavailable` and `max-surge` values that preserve enough serving
capacity while the update runs.

### Check backend health

``` bash
watch -n 2 \
  gcloud compute backend-services get-health fancy-fe-frontend --global
```

This repeatedly checks the health status of the backend service.

-   `watch` repeats a command.
-   `-n 2` repeats it every two seconds.
-   `get-health` reports backend health information.

Press `Ctrl+C` to stop `watch`.

If instances remain unhealthy, check:

1.  Is the application process running?
2.  Is it listening on the expected port?
3.  Does the health-check path return a successful response?
4.  Does the firewall allow health-check traffic?
5.  Did the startup script finish successfully?
6.  Can the application be reached directly from an allowed network
    path?

An instance can be running at the VM level while the application inside
it is not ready.

## 8.18 Autoscaling

### What is autoscaling?

Autoscaling automatically changes the number of VM instances in a MIG
based on configured signals and limits.

For example:

-   When demand increases, the MIG can add VMs.
-   When demand decreases, the MIG can remove extra VMs, subject to the
    configured minimum and other rules.

Autoscaling helps the application respond to changing demand without an
administrator manually changing the group size.

### Example: autoscaling based on load-balancing utilization

The lab uses:

``` bash
gcloud compute instance-groups managed set-autoscaling fancy-be-mig \
  --zone=us-west1-b \
  --max-num-replicas=2 \
  --target-load-balancing-utilization=0.60
```

This configures autoscaling based on load-balancing utilization, with a
maximum of two replicas.

**Important limitation in the lab value:** If the group already has two
instances and the maximum is two, the autoscaler cannot add a third
instance. To demonstrate scale-out, configure a maximum greater than the
current group size.

For example, an illustrative configuration is:

``` bash
gcloud compute instance-groups managed set-autoscaling fancy-be-mig \
  --zone=us-west1-b \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-load-balancing-utilization=0.60
```

This sets a minimum of two and a maximum of five instances. The target
is a scaling signal, not a guarantee that the group will always have
exactly the number of VMs needed for every traffic pattern.

Autoscaling can use different signals, such as CPU utilization or
load-balancing utilization. Choose a signal that reflects the
application's actual bottleneck and configure appropriate limits.

## 8.19 Cloud CDN

### What is Cloud CDN?

Cloud CDN is a content delivery service that caches eligible content
closer to users.

A cache is a temporary copy of content kept so it can be served again
without always requesting it from the original backend.

### Why use Cloud CDN?

Imagine that users in different countries repeatedly request the same
images, JavaScript files, stylesheets, or other cacheable content.

Without a cache, every request may need to reach the backend. With Cloud
CDN, eligible content can be served from a cache location closer to the
user.

Benefits can include:

-   Lower response latency for cache hits.
-   Less repeated work for the backend.
-   Reduced data transfer from the backend for cached content.

### Enable Cloud CDN on a backend service

The lab uses:

``` bash
gcloud compute backend-services update fancy-fe-frontend \
  --enable-cdn \
  --global
```

This enables Cloud CDN for the specified global backend service.

### Cache hit and cache miss

-   **Cache hit:** The requested object is already available in the
    cache and can be served from there.
-   **Cache miss:** The object is not available in the cache, so the
    request goes to the backend. If the response is eligible for
    caching, it may be stored for later requests.

Not every response is automatically cacheable. Cache behavior depends on
the backend configuration and HTTP caching rules and headers.

## 8.20 Updating the application and VM template

The lab describes two different types of change. Keep them separate.

### Case A: The application code changes

If the startup script downloads application code from Cloud Storage:

1.  Update the application code.
2.  Build or prepare the new application artifact.
3.  Upload the new artifact to Cloud Storage.
4.  Ensure the startup process will retrieve the new artifact.
5.  Restart or replace the VM instances so the new startup process runs.
6.  Check the application and backend health.

The lab includes a command to remove existing Node.js dependency folders
before copying the code:

``` bash
cd ~
rm -rf monolith-to-microservices/*/node_modules

gcloud storage cp -r monolith-to-microservices \
  gs://YOUR_BUCKET_NAME/
```

Be careful about the destination and object names. A production
deployment should use a versioned artifact or a distinct release path so
that new VMs do not accidentally retrieve a partially uploaded or
unexpected version.

### Case B: VM configuration changes

Examples include:

-   Changing the machine type.
-   Changing the boot image.
-   Changing network tags.
-   Changing the service account.
-   Changing startup-script metadata.

These are VM configuration changes. The correct pattern is to create a
new instance template and roll it out to the MIG.

### Why create a new template?

Instance templates are immutable. You cannot edit an existing template
in place.

The lab changes the machine type of an existing VM:

``` bash
gcloud compute instances set-machine-type frontend \
  --zone=us-west1-b \
  --machine-type=e2-small
```

Then it creates a new template from that VM:

``` bash
gcloud compute instance-templates create fancy-fe-new \
  --source-instance=frontend \
  --source-instance-zone=us-west1-b
```

Finally, it starts a MIG update:

``` bash
gcloud compute instance-groups managed rolling-action start-update fancy-fe-mig \
  --zone=us-west1-b \
  --version=template=fancy-fe-new
```

This tells the MIG to roll out the new template version.

Check the rollout:

``` bash
watch -n 2 \
  gcloud compute instance-groups managed list-instances fancy-fe-mig \
  --zone=us-west1-b
```

**Important:** Changing one VM that belongs to a MIG is not the normal
way to manage the whole group. The MIG is designed to keep instances
consistent with its template. For repeatable changes, create a new
template and update the MIG. The lab uses an existing VM as a convenient
way to capture a new template, but production teams often define
templates directly and keep their configuration in source control.

## 8.21 Why does the lab use SSH to kill a Node.js process?

Your notes ask why killing a Node.js process requires an SSH connection.

The reason is that the Node.js process is running inside the VM. To
inspect or stop that process manually, you need to run commands on the
VM's operating system.

You can connect from Cloud Shell:

``` bash
gcloud compute ssh frontend --zone=us-west1-b
```

Inside the VM, inspect Node.js processes:

``` bash
ps aux | grep node
```

This lists processes and searches for the word `node`.

To stop a process, first identify its correct process ID (PID). Then:

``` bash
kill PID
```

Use the actual PID in place of `PID`.

`kill` normally sends a termination signal asking the process to stop.
Avoid using `kill -9` unless necessary because it forcefully terminates
the process and does not give it a chance to shut down cleanly.

### Do you always need SSH to stop an application?

No.

SSH is one way to administer the VM manually. In a real application, it
is usually better to run the application under a service manager such as
`systemd`, or use a process manager or deployment platform that can
start, stop, and restart it consistently.

For example, if the application is configured as a `systemd` service:

``` bash
sudo systemctl status myapp
sudo systemctl restart myapp
sudo systemctl stop myapp
```

Replace `myapp` with the actual service name.

A process that is managed by `systemd` can be restarted after a VM
reboot according to its service configuration. A process started
manually in a terminal may stop when the session ends or may not restart
automatically.

## 8.22 Troubleshooting checklist for this lab

Use this checklist when a VM, application, MIG, or load balancer does
not work.

### VM creation problems

``` bash
gcloud config list
gcloud compute instances list
gcloud compute instances describe VM_NAME --zone=ZONE
```

Check:

-   Is the correct project selected?
-   Is the requested zone correct?
-   Is the Compute Engine API enabled?
-   Does the account have the required IAM permissions?
-   Is quota available for the requested machine type and region?

### SSH connection problems

Check:

-   Is the VM running?
-   Is the project and zone correct?
-   Does the user have the required IAM access?
-   Is SSH access allowed by the VPC firewall on TCP port 22?
-   Is OS Login enabled, and does the user have the required OS Login
    role?
-   Is the VM using metadata-based SSH keys instead?
-   Is the correct SSH key available?

Useful command:

``` bash
gcloud compute ssh VM_NAME --zone=ZONE
```

### Application not responding

Inside the VM:

``` bash
sudo systemctl status nginx
ps aux | grep node
sudo journalctl -u nginx -n 50
curl http://localhost:8080
```

Check:

-   Did the startup script finish?
-   Is the application process running?
-   Is it listening on the expected port?
-   Does the health-check path return a successful response?
-   Are environment variables and dependencies correct?

### Startup script problems

Check:

-   Is the script URL correct?
-   Does the VM's service account have permission to read the object?
-   Is the guest environment installed and running?
-   Does the script have a valid shebang such as `#!/bin/bash`?
-   Are commands failing because of missing packages or incorrect paths?
-   Does the script handle repeated execution safely?

### Load balancer reports an unhealthy backend

Check:

-   Does the backend service use the correct named port?
-   Is the health check using the correct port and path?
-   Is the application listening on that port?
-   Does the firewall allow the required health-check traffic?
-   Is the correct MIG attached to the backend service?
-   Has the application finished starting?

### MIG does not update as expected

Check:

-   Does the new instance template exist?
-   Is the MIG configured to use the new template version?
-   Is the update still in progress?
-   Are instances failing startup or health checks?
-   Is the update policy allowing enough capacity during replacement?

### Autoscaling does not add instances

Check:

-   Is autoscaling enabled on the correct MIG?
-   Is the maximum replica count greater than the current size?
-   Is the selected utilization signal actually reaching the target?
-   Is the MIG in the expected zone or region?
-   Are quota and capacity available?

## 8.23 Important corrections and gaps found in the supplied lab notes

This section is important because the source notes contain some
incomplete commands and a few details that could lead to confusion.

  -----------------------------------------------------------------------
  Original note or issue              Review and correction
  ----------------------------------- -----------------------------------
  "Cloud Shell is a VM"               Cloud Shell is a managed shell
                                      environment running in Google
                                      Cloud. It is separate from the VM
                                      you administer through SSH.

  Cloud Shell configuration must      Shell variables created with
  always be reset                     `export` are session-specific.
                                      `gcloud config set` stores CLI
                                      properties, although lab
                                      environments can reset state. Check
                                      the active configuration.

  SSH asks for a password             It may be asking for a private-key
                                      passphrase, Google authentication,
                                      a remote Linux password, or a
                                      `sudo` password. These are
                                      different things.

  SSH is needed to attach a disk      SSH is needed for the Linux-side
                                      tasks such as finding, formatting,
                                      and mounting the disk. The Google
                                      Cloud attach command can run from
                                      Cloud Shell.

  `mkfs.ext4` formats a disk          Correct, but it destroys existing
                                      data on the selected device. Verify
                                      the device before running it.

  Add a disk entry to `/etc/fstab`    Prefer a filesystem UUID and test
                                      with `sudo mount -a` before
                                      rebooting.

  Local SSD is always lost after      Too broad. Persistence depends on
  reboot                              the event and configuration. It is
                                      still temporary storage and should
                                      not hold the only copy of important
                                      data.

  Bucket is used to hold code and     Correct. The VM also needs
  startup scripts                     permission to read private bucket
                                      objects. Do not make the bucket
                                      public as a shortcut.

  Startup script automatically        Correct when the script is
  configures every new VM             configured and can run
                                      successfully. It should be
                                      repeatable and should log failures.

  Instance template can be edited     It cannot be edited in place.
                                      Create a new template and roll it
                                      out to the MIG.

  `fancy-fe` is the frontend MIG      `fancy-fe` is the template name in
                                      the example. The MIG name is
                                      `fancy-fe-mig`. Use the correct
                                      resource name in commands.

  Path matcher uses `fancy-be-orders` The supplied notes do not show
  and `fancy-be-products`             creation of these backend services.
                                      They must be created and connected
                                      to the correct backend group and
                                      named port before the URL map can
                                      reference them.

  Autohealing and load-balancer       They have different purposes.
  health checks are the same          Load-balancer checks control
                                      traffic routing; MIG autohealing
                                      checks can trigger VM repair.

  `--max-unavailable=100%` is a       It can make every instance
  normal rolling update setting       unavailable during replacement. It
                                      is suitable for a quick lab
                                      exercise, not a high-availability
                                      production rollout.

  Autoscaling max replicas is 2 while The group cannot scale above two.
  the group has 2 instances           Use a higher maximum to demonstrate
                                      scale-out.

  Changing one VM updates the whole   It does not. The MIG uses its
  MIG                                 instance template. Create a new
                                      template and update the group.

  HTTP is used to avoid certificate   Correct for the lab, but public
  setup                               production applications should
                                      normally use HTTPS.
  -----------------------------------------------------------------------

## 8.24 Practical end-to-end picture: how all the pieces work together

The lab is demonstrating how to run a small application on Compute
Engine and make it more scalable.

The high-level flow is:

1.  Prepare the application code and startup script.
2.  Upload the code and startup script to Cloud Storage.
3.  Create frontend and backend VM instances.
4.  Configure the VMs to run the startup script.
5.  Add network tags and firewall rules.
6.  Create instance templates from the VM configurations.
7.  Create frontend and backend MIGs from those templates.
8.  Configure named ports and health checks.
9.  Configure autohealing for the MIGs.
10. Create backend services and connect the MIGs.
11. Create a URL map to route requests.
12. Create a target proxy and forwarding rule to expose the application.
13. Configure autoscaling and Cloud CDN where appropriate.
14. When the application or VM configuration changes, publish the new
    artifact or create a new template and roll it out.
15. Check application health and troubleshoot any failing components.

### Architecture in simple terms

``` text
                         Users
                           |
                           v
                  Public Load Balancer
                           |
                           v
                        URL Map
                    /              \
                   /                \
                  v                  v
           Frontend Service     Backend Services
                  |              /          \
                  v             v            v
             Frontend MIG   Orders MIG   Products MIG
                  |             |            |
             Multiple VMs   Multiple VMs  Multiple VMs
                  \             |            /
                   \            |           /
                    \           v          /
                     Application resources
                              |
                              v
                       Cloud Storage
                 (code and startup scripts)
```

This is a conceptual view of the lab. The exact backend services and
MIGs must be created and connected as described in the command sections.

------------------------------------------------------------------------


------------------------------------------------------------------------

## 10. Quick command reference

  --------------------------------------------------------------------------------------------------------------------
  Task                                Command
  ----------------------------------- --------------------------------------------------------------------------------
  Show active account                 `gcloud auth list`

  Show active project                 `gcloud config get-value project`

  Set project                         `gcloud config set project PROJECT_ID`

  Set default zone                    `gcloud config set compute/zone ZONE`

  Set default region                  `gcloud config set compute/region REGION`

  List VMs                            `gcloud compute instances list`

  Create VM                           `gcloud compute instances create VM_NAME --zone=ZONE --machine-type=e2-medium`

  Connect to VM                       `gcloud compute ssh VM_NAME --zone=ZONE`

  Stop VM                             `gcloud compute instances stop VM_NAME --zone=ZONE`

  Start VM                            `gcloud compute instances start VM_NAME --zone=ZONE`

  Create disk                         `gcloud compute disks create DISK_NAME --size=SIZE --zone=ZONE`

  Attach disk                         `gcloud compute instances attach-disk VM_NAME --disk=DISK_NAME --zone=ZONE`

  List templates                      `gcloud compute instance-templates list`

  List MIGs                           `gcloud compute instance-groups managed list`

  List MIG instances                  `gcloud compute instance-groups managed list-instances MIG_NAME --zone=ZONE`

  List health checks                  `gcloud compute health-checks list`

  List firewall rules                 `gcloud compute firewall-rules list`

  Copy file to bucket                 `gcloud storage cp FILE gs://BUCKET/`

  Copy directory to bucket            `gcloud storage cp -r DIRECTORY gs://BUCKET/`
  --------------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 11. Official Google Cloud references

Use official documentation when command flags, machine families, disk
behavior, IAM requirements, or load-balancer configuration may have
changed.

-   [Compute Engine
    overview](https://cloud.google.com/compute/docs/overview)
-   [Connect to Linux VMs using
    SSH](https://cloud.google.com/compute/docs/connect/standard-ssh)
-   [About SSH connections and SSH
    keys](https://cloud.google.com/compute/docs/instances/ssh)
-   [OS Login](https://cloud.google.com/compute/docs/oslogin)
-   [Persistent
    Disk](https://cloud.google.com/compute/docs/disks/persistent-disks)
-   [Add a Persistent Disk to a
    VM](https://cloud.google.com/compute/docs/disks/add-persistent-disk)
-   [Local SSD](https://cloud.google.com/compute/docs/disks/local-ssd)
-   [Startup
    scripts](https://cloud.google.com/compute/docs/instances/startup-scripts)
-   [Instance
    templates](https://cloud.google.com/compute/docs/instance-templates)
-   [Managed instance
    groups](https://cloud.google.com/compute/docs/instance-groups)
-   [Rolling updates for managed instance
    groups](https://cloud.google.com/compute/docs/instance-groups/rolling-out-updates-to-managed-instance-groups)
-   [External Application Load Balancer
    overview](https://cloud.google.com/load-balancing/docs/https)
-   [URL
    maps](https://cloud.google.com/load-balancing/docs/url-map-concepts)
-   [Target
    proxies](https://cloud.google.com/load-balancing/docs/target-proxies)
-   [Cloud Storage documentation](https://cloud.google.com/storage/docs)
-   [Cloud CDN overview](https://cloud.google.com/cdn/docs/overview)

**Learning objective:** After completing this chapter, you should be
able to explain how a VM is created, how to access and administer it,
how disks and startup scripts work, and how instance templates, MIGs,
health checks, load balancing, autoscaling, and Cloud CDN fit together
in a practical application deployment.
