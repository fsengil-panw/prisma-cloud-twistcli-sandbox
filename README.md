# prisma-cloud-twistcli-sandbox
Of course. Here is a comprehensive `README.md` file that captures all the information for setting up the automated malware analysis pipeline.

-----

# Automated Malware Sandbox Analysis with Jenkins, Prisma Cloud, and AWS EC2 Spot Instances

This guide details how to build a scalable, cost-effective, and secure pipeline for automated malware analysis. It leverages the power of Jenkins for orchestration, Prisma Cloud's `twistcli` for advanced sandbox analysis, and AWS EC2 Spot Instances for ephemeral, low-cost compute environments.

## Overview

The primary goal of this pipeline is to automate the process of analyzing suspicious files in a safe, isolated environment. Instead of maintaining dedicated analysis machines, this solution dynamically provisions a fresh EC2 Spot Instance for each analysis run. Jenkins orchestrates the entire workflow, from creating the instance to running the scan and processing the results. Upon completion, the instance is automatically destroyed, ensuring a clean slate for every analysis and minimizing costs.

### Core Workflow

1.  **Trigger:** A Jenkins pipeline job is started.
2.  **Provision:** Jenkins, via the EC2 Plugin, requests a new build agent. This action provisions a fresh EC2 Spot Instance from a predefined template.
3.  **Execute:** Once the instance is ready, the pipeline runs on it:
      * It checks out the source code, including the file to be analyzed.
      * It securely downloads the `twistcli` utility from your Prisma Cloud Console.
      * It executes `twistcli sandbox <file>`, which uploads the sample to Prisma Cloud's backend for detonation and analysis.
4.  **Report:** The pipeline retrieves the JSON analysis report from `twistcli`, parses the verdict, and archives the report.
5.  **Destroy:** As soon as the job finishes, the Jenkins EC2 Plugin automatically terminates the Spot Instance, ensuring you only pay for the compute time you use and that the contaminated environment is securely destroyed.

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
          * **AWS Credentials:** Select the `aws-ec2-credentials` you just created.
          * **AWS Region:** Choose the region where you want to launch instances (e.g., `us-east-1`).
      * Under **AMIs**, click **Add AMI** to create an agent template:
          * **AMI ID:** Provide a recent Amazon Linux 2 or Ubuntu AMI ID (e.g., `ami-0c55b159cbfafe1f0`).
          * **Instance Type:** Choose a suitable type (e.g., `t3.medium`).
          * **Security group(s):** Assign a security group that allows inbound SSH from your Jenkins controller and necessary outbound access to the internet.
          * **Labels:** Assign a unique label that your `Jenkinsfile` will reference. For example: `ec2-spot-sandbox`.
          * Click **Advanced...** to configure Spot settings:
              * Check the **Use Spot instance** box.
              * You can leave the **Spot max bid price** blank to bid the current On-Demand price, which is the recommended practice.
          * Ensure the **Remote FS root**, **User**, and **SSH key** settings are appropriate for your chosen AMI.
      * **Save** the configuration.

### Step 2: Store Prisma Cloud Credentials in Jenkins

Securely store your Prisma Cloud credentials for the pipeline to use.

1.  **Add Prisma API Keys:**

      * Navigate to **Manage Jenkins \> Credentials**.
      * Add a new credential of the **Username with password** kind.
      * **Username:** Enter your Prisma Cloud Access Key.
      * **Password:** Enter your Prisma Cloud Secret Key.
      * **ID:** Set the ID to `prisma-cloud-credentials`.

2.  **Add Prisma Console URL:**

      * Add another credential, this time of the **String** kind.
      * **Secret:** Enter the full URL to your Prisma Cloud Console (e.g., `https://us-east1.cloud.twistlock.com/my-company`).
      * **ID:** Set the ID to `prisma-console-url`.

-----

## Benefits of this Approach

  * **Cost-Efficiency:** EC2 Spot Instances offer discounts of up to 90% over On-Demand prices, making this solution incredibly cheap to run.
  * **Maximum Security:** Each analysis runs in a completely new, isolated environment. The instance is destroyed after use, eliminating any risk of cross-contamination or persistent threats.
  * **Scalability:** Jenkins can provision dozens of agents concurrently, allowing you to build a high-throughput analysis capability without managing a fixed pool of machines.
  * **Zero Maintenance:** The entire analysis environment is ephemeral and defined as code. There are no dedicated servers to patch or maintain.