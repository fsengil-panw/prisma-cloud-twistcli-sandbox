# prisma-cloud-twistcli-sandbox

Here is the updated `README.md` that reflects the new image-based analysis workflow.

# Automated Container Image Sandbox Analysis with Jenkins, Prisma Cloud, and AWS EC2 Spot Instances

This guide details how to build a scalable, cost-effective, and secure pipeline for automated **container image analysis**. It leverages the power of Jenkins for orchestration, Prisma Cloud's `twistcli` for advanced sandbox analysis, and AWS EC2 Spot Instances for ephemeral, low-cost compute environments.

This pipeline is designed to be integrated into your CI/CD process, automatically scanning newly built container images for malware and suspicious behavior before they are pushed to a registry.

### Core Workflow

1.  **Trigger:** A Jenkins pipeline job is started, typically after a code commit.
2.  **Provision:** Jenkins, via the EC2 Plugin, provisions a fresh **EC2 Spot Instance** from a pre-defined template. This instance must have Docker installed.
3.  **Build:** The pipeline checks out the source code and uses a `Dockerfile` in the repository to build a new container image.
4.  **Analyze:**
      * It securely downloads the `twistcli` utility from your Prisma Cloud Console.
      * It executes `twistcli sandbox --analysis-type image <your-image-name>`, which submits the image to Prisma Cloud's backend for detonation and analysis.
5.  **Report:** The pipeline retrieves the JSON analysis report from `twistcli`, parses the verdict (e.g., malware, suspicious, benign), and archives the full report.
6.  **Destroy:** As soon as the job finishes, the Jenkins EC2 Plugin automatically **terminates the Spot Instance**, ensuring you only pay for the compute time you use and that the environment is securely destroyed.

-----

## Prerequisites

Before you begin, ensure you have the following:

  * **Prisma Cloud Compute Edition:** An active subscription with API access.
      * Prisma Cloud Console URL
      * Prisma Cloud Access Key
      * Prisma Cloud Secret Key
  * **AWS Account:** An AWS account with an IAM user or role that has permissions to create and manage EC2 instances.
      * AWS Access Key ID
      * AWS Secret Access Key
  * **Jenkins Server:** A running Jenkins instance with administrator access.
  * **A `Dockerfile`** in your Git repository.
  * **Required Jenkins Plugins:**
      * **[Amazon EC2 Plugin](https://plugins.jenkins.io/ec2/)**

-----

## Configuration Steps

### Step 1: Configure the Jenkins EC2 Plugin

This setup connects Jenkins to your AWS account, allowing it to dynamically create agents.

1.  **Add AWS Credentials to Jenkins:**

      * Navigate to **Manage Jenkins \> Credentials**.
      * Add a new credential of the **AWS Credentials** kind.
      * Enter the **Access Key ID** and **Secret Access Key** for your AWS IAM user.
      * Set the ID to `aws-ec2-credentials`.

2.  **Configure the EC2 Cloud:**

      * Navigate to **Manage Jenkins \> Clouds**.
      * Click **Add a new cloud** and select **Amazon EC2**.
      * Fill in the cloud details:
          * **AWS Credentials:** Select `aws-ec2-credentials`.
          * **AWS Region:** Choose your desired region (e.g., `us-east-1`).
      * Under **AMIs**, click **Add AMI** to create an agent template:
          * **AMI ID:** Provide an AMI that has Docker pre-installed. The latest **Amazon Linux 2** AMIs are a good choice.
          * **Instance Type:** Choose a suitable type (e.g., `t3.medium`).
          * **Security group(s):** Assign a security group that allows inbound SSH from your Jenkins controller and necessary outbound access.
          * **Labels:** Assign a unique label. For example: `ec2-spot-sandbox`.
          * Click **Advanced...** to configure Spot settings:
              * Check the **Use Spot instance** box.
              * Leave the **Spot max bid price** blank to bid the On-Demand price.
      * **Save** the configuration.

### Step 2: Store Prisma Cloud Credentials in Jenkins

Securely store your Prisma Cloud credentials for the pipeline to use.

1.  **Add Prisma API Keys:**

      * Navigate to **Manage Jenkins \> Credentials**.
      * Add a new credential of the **Username with password** kind.
      * **Username:** Your Prisma Cloud Access Key.
      * **Password:** Your Prisma Cloud Secret Key.
      * **ID:** `prisma-cloud-credentials`.

2.  **Add Prisma Console URL:**

      * Add another credential of the **String** kind.
      * **Secret:** The full URL to your Prisma Cloud Console.
      * **ID:** `prisma-console-url`.

-----

## Pipeline Implementation

Place the following `Jenkinsfile` in the root of your Git repository.

* **Zero Maintenance:** The entire analysis environment is ephemeral and defined as code. There are no dedicated servers to patch or maintain.
