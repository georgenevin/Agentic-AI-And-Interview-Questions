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

  -------------------------------------------------------------------------
  Component               Description             Example
  ----------------------- ----------------------- -------------------------
  Principal               The identity that needs `developer@example.com`
                          access.                 

  Role                    A collection of         Compute Instance Admin
                          permissions granted to  
                          the principal.          

  Resource                The Google Cloud        A development project
                          resource on which       
                          access is granted.      
  -------------------------------------------------------------------------

The relationship can be represented as:

**Principal → Role → Permissions → Resource**

For example, a developer may receive the Compute Instance Admin role on
a development project to manage virtual machine instances.

### 2.2 Role types

  -----------------------------------------------------------------------
  Role type                           Description
  ----------------------------------- -----------------------------------
  Basic roles                         Broad roles such as Owner, Editor,
                                      and Viewer.

  Predefined roles                    Google-managed roles designed for
                                      specific services and tasks.

  Custom roles                        Roles containing a selected set of
                                      permissions defined by an
                                      organization.
  -----------------------------------------------------------------------

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

  -----------------------------------------------------------------------
  Component                           Description
  ----------------------------------- -----------------------------------
  Publisher                           Application that sends messages.

  Topic                               Named resource to which publishers
                                      send messages.

  Subscription                        Resource through which subscribers
                                      receive messages.

  Subscriber                          Application that receives and
                                      processes messages.

  Message                             Data or event sent by a publisher.

  Acknowledgment (ACK)                Confirmation that a message has
                                      been successfully processed.
  -----------------------------------------------------------------------

### 4.2 How Pub/Sub works

1.  A publisher sends a message to a topic.
2.  Pub/Sub stores and distributes the message.
3.  A subscription makes the message available to its subscribers.
4.  A subscriber receives and processes the message.
5.  The subscriber acknowledges successful processing.

### 4.3 Pull vs. push subscriptions

  -----------------------------------------------------------------------
  Pull subscription                   Push subscription
  ----------------------------------- -----------------------------------
  The subscriber requests messages.   Pub/Sub sends messages to a
                                      configured HTTPS endpoint.

  The subscriber controls when to     Pub/Sub initiates delivery.
  retrieve messages.                  

  Useful for background workers and   Useful for HTTP-based services and
  services that control their         endpoints.
  polling.                            
  -----------------------------------------------------------------------

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
