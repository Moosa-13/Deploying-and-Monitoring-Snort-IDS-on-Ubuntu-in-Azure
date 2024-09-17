## Cyber Security Project: Deploying and Monitoring Snort IDS on Ubuntu in Azure

### Table of Contents

1. Introduction
2. Project Overview
3. Prerequisites
4. Step-by-Step Instructions
   1. Setting Up the Environment in Azure
   2. Deploying Snort IDS
   3. Configuring Snort Rules
   4. Simulating Brute-Force SSH Attack
   5. Monitoring and Logging Snort Alerts
   6. Installing Additional Tools for Testing
   7. Downloading and Configuring Snort Rules
5. Troubleshooting and Fine-Tuning
6. Conclusion
7. Future Work

---

### Introduction

As cyber threats grow in complexity and volume, organizations must employ sophisticated defenses to monitor and protect their networks. One such defense is an **Intrusion Detection System (IDS)**, which monitors network traffic for malicious activities and generates alerts when suspicious events are detected. This project focuses on deploying and configuring the popular open-source IDS, **Snort**, on an Ubuntu server within the Azure cloud infrastructure.

In this project, we will simulate a network environment with three machines: 
- a **Snort IDS Server** on Ubuntu, 
- a **Target Machine** running vulnerable services on Ubuntu, 
- and an **Attacker Machine** (Kali Linux) designed to launch attacks on the target.

We will configure Snort to detect common network attacks, particularly focusing on SSH brute-force attempts. By the end of this project, you will understand how to deploy and configure Snort, write custom rules, simulate attacks, and monitor alerts. This is a foundational setup that will prepare you for more advanced network security tasks.

---

### Project Overview

- **Snort IDS Server**: An Ubuntu Server running Snort IDS to monitor network traffic and detect malicious activities.
- **Target Machine**: An Ubuntu Server running vulnerable services that will serve as the victim machine.
- **Attacker Machine**: A Kali Linux VM used to simulate different types of attacks (such as brute-force attacks on SSH).
- **Azure Environment**: All virtual machines are hosted in Microsoft Azure within the same virtual network for seamless communication between machines.

This setup allows for controlled testing of security tools and attack simulations, providing valuable insights into how Snort works in a real-world environment.

---

### Prerequisites

Before diving into the detailed steps, ensure you have the following:

- **Azure Subscription**: An active Azure account with sufficient credits to deploy multiple VMs.
- **Basic Networking Knowledge**: Understanding of IP addresses, subnets, and basic networking concepts.
- **Familiarity with Linux**: You should know how to use basic Linux commands, navigate the terminal, and install software packages.
- **Attack Simulation Tools**: We will be using tools like **Hydra** for brute-force attacks from the Kali Linux VM.
- **Snort Account**: You will need to register an account on [Snort’s website](https://www.snort.org/) to download the latest rule sets.

---

### Step-by-Step Instructions

#### Setting Up the Environment in Azure

##### 1. Create a Resource Group

To organize our resources in Azure, we first create a **Resource Group**. Resource groups allow you to manage related resources, like virtual machines and networks, under a single umbrella.

- **Step 1**: Log into your [Azure Portal](https://portal.azure.com).
- **Step 2**: Navigate to **Resource Groups** from the left-hand menu.
- **Step 3**: Click on **Add** and create a new resource group named `CyberSecProjectRG`.
- **Step 4**: Choose a region closest to your location for optimal performance (e.g., East US, West Europe).
- **Step 5**: Click **Review + Create** and then **Create**.

##### 2. Create a Virtual Network (VNet)

Next, we need to set up a virtual network that will house our virtual machines. This network will allow our VMs to communicate with each other privately.

- **Step 1**: Navigate to **Virtual Networks** and click **Add**.
- **Step 2**: Name your VNet `CyberSecVNet`.
- **Step 3**: Set the **Address Space** to `10.0.0.0/16` (this will give us enough IP addresses to work with).
- **Step 4**: Under **Subnets**, add a subnet named `DefaultSubnet` with an address range of `10.0.0.0/24`.
- **Step 5**: Assign the virtual network to the resource group you created earlier (`CyberSecProjectRG`).
- **Step 6**: Click **Create** to finalize your VNet.

##### 3. Deploy the Ubuntu Target Machine

The **Target Machine** will be a vulnerable Ubuntu server that will serve as the victim in our attack simulations. 

- **Step 1**: Go to **Virtual Machines** and click **Add**.
- **Step 2**: Name your VM `TargetVM` and select **Ubuntu Server 20.04 LTS** as the image.
- **Step 3**: Choose a VM size that suits your budget (e.g., Standard B1s).
- **Step 4**: Configure the VM to be deployed in the **CyberSecVNet** you created earlier.
- **Step 5**: Enable **SSH access** by configuring the Network Security Group (NSG) to allow SSH on port **22**.
- **Step 6**: Complete the deployment by clicking **Create**.

##### 4. Deploy the Snort IDS Server

The **Snort IDS Server** will monitor network traffic to detect malicious activities. We will also use **Ubuntu Server 20.04 LTS** for the Snort IDS server.

- **Step 1**: Repeat the process from step 3, but this time name the VM `SnortIDSVM`.
- **Step 2**: Ensure the VM is in the same **CyberSecVNet** and subnet.
- **Step 3**: Configure SSH access on port **22** and complete the deployment.

##### 5. Deploy the Kali Linux Attacker Machine

The **Attacker Machine** will be a **Kali Linux** VM used for launching brute-force attacks and other exploits against the **TargetVM**.

- **Step 1**: Navigate to **Virtual Machines** > **Add**.
- **Step 2**: Search for **Kali Linux** in the Azure Marketplace and select the official image.
- **Step 3**: Name the VM `KaliAttackerVM` and ensure it is deployed in the same **CyberSecVNet**.
- **Step 4**: Complete the deployment, ensuring SSH access is allowed for external connections.

---

#### Deploying Snort IDS

Once the environment is set up, we can install and configure **Snort** on the `SnortIDSVM` to monitor network traffic. Snort will be set up in **IDS mode** (as opposed to **IPS mode**) to detect attacks without blocking them.

##### 1. Update the System

Start by updating the package repositories and upgrading any existing software to the latest versions:

```bash
sudo apt update && sudo apt upgrade -y
```

This ensures that the system is secure and all packages are up-to-date.

##### 2. Install Required Dependencies

Snort requires several dependencies for compiling and running effectively. Install the necessary libraries and tools:

```bash
sudo apt install -y build-essential libpcap-dev libpcre3-dev libdumbnet-dev bison flex zlib1g-dev liblzma-dev openssl libssl-dev pkg-config
```

These libraries include development tools (like `build-essential`) and libraries for network packet capture (`libpcap`), pattern matching (`libpcre3`), and cryptography (`openssl`).

##### 3. Install DAQ (Data Acquisition Library)

The **Data Acquisition Library (DAQ)** is required by Snort to process packet data. Install DAQ using the following steps:

```bash
wget https://www.snort.org/downloads/snort/daq-2.0.7.tar.gz
tar -xzvf daq-2.0.7.tar.gz
cd daq-2.0.7
./configure && make && sudo make install
```

This will download, extract, and install DAQ.

##### 4. Install Snort

Now it’s time to download and install **Snort** itself. We will use Snort version 2.9.20 in this setup:

```bash
wget https://www.snort.org/downloads/snort/snort-2.9.20.tar.gz
tar -xzvf snort-2.9.20.tar.gz
cd snort-2.9.20
./configure --enable-sourcefire && make && sudo make install
```

##### 5. Create Directories for Snort

Snort requires specific directories for rules and log files. Let’s create those directories and set appropriate permissions:

```bash
sudo mkdir -p /etc/snort/rules /var/log/snort /usr/local/lib/snort_dynamicrules
sudo touch /etc/snort/rules/white_list.rules /etc/snort/rules/black_list.rules
sudo chown -R snort:snort /etc/snort /var/log/snort
```

These commands set up the necessary directories for Snort to store its configuration, rule files, and logs.

##### 6. Download Snort Rules

Snort uses **rules** to identify different types of network traffic and classify them as malicious or benign. You can download the community rule set to get started:

```bash
wget https://www.snort.org/downloads/community/community-rules.tar.gz
sudo tar -xzvf community-rules.tar.gz -C /etc/snort/rules
```

You can also register for an account on the [Snort website](https://www.snort.org/) to download the latest paid rules.

##### 7. Configure snort.conf

The Snort configuration file (`snort.conf`) is where we define paths to the rule files and specify network variables. Open the configuration file for editing:

```bash
sudo nano /etc/snort/snort.conf
```

Set the path for the rule files:

```bash
var RULE_PATH /etc/snort/rules
```

Include the necessary rules:

```bash
include $RULE_PATH/local.rules
include $RULE_PATH/community.rules
```

Save the changes and exit the editor.

##### 8. Test Snort Configuration

Before running Snort, it’s a good idea to test the configuration to ensure there are no errors:

```bash
sudo snort -T -c /etc/snort/snort.conf
```

If everything is set up correctly, you should see the message **"Snort successfully validated the configuration!"**.

---

#### Configuring Snort Rules

To detect malicious traffic, we need to create and configure rules. Snort comes with pre-configured rules for various types of attacks, but we can also create custom rules based on our specific use case.

##### 1. Create or Modify Local Rules

The **`local.rules`** file is where we can define custom rules. For example, let’s create a rule to detect SSH brute-force attempts:

```bash
sudo nano /etc/snort/rules/local.rules
```

Add the following rule to detect multiple failed login attempts on SSH (port 22):

```bash
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force Attempt Detected"; flags:S; threshold:type threshold, track by_src, count 3, seconds 60; sid:1000001; rev:1;)
```

This rule tracks incoming connection attempts to port 22 and generates an alert if there are three or more attempts within 60 seconds.

##### 2. Include local.rules in snort.conf

Ensure that `local.rules` is included in the **`snort.conf`** file:

```bash
include $RULE_PATH/local.rules
```

##### 3. Verify and Reload Snort

After adding the rule, test the configuration again to make sure there are no errors:

```bash
sudo snort -T -c /etc/snort/snort.conf
```

If there are no errors, restart Snort in IDS mode.

---

#### Simulating Brute-Force SSH Attack

Now that Snort is configured and running, we can simulate a brute-force attack from the **Kali Linux** attacker machine.

##### 1. Run Snort in IDS Mode

To monitor network traffic and generate alerts, start Snort in **IDS mode**:

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

This command will start Snort and display alerts directly in the console.

##### 2. Simulate SSH Brute-Force Attack Using Hydra

From the **Kali Linux** VM, use the **Hydra** tool to simulate a brute-force attack on the SSH service running on the **TargetVM**. Replace `10.0.0.4` with the IP address of your **TargetVM**:

```bash
hydra -l saalim -P /usr/share/wordlists/rockyou.txt -t 4 ssh://10.0.0.4
```

This command will use a common password list (`rockyou.txt`) to attempt multiple SSH login attempts against the `saalim` user on the target machine.

---

#### Monitoring and Logging Snort Alerts

##### 1. Check for Alerts in the Console

If Snort detects the brute-force attack, it will output alerts in real-time in the console where Snort is running.

##### 2. Check Snort Log Files

If Snort is logging to files, you can check the **alert log** for a record of all detected incidents:

```bash
sudo tail -f /var/log/snort/alert
```

##### 3. Example Alert

For the SSH brute-force attack, you should see an alert similar to the following:

```
[**] [1:1000001:1] SSH Brute Force Attempt Detected [**]
{TCP} <source_ip>:<source_port> -> <target_ip>:22
```

This indicates that Snort successfully detected multiple login attempts to the SSH service.

---

#### Installing Additional Tools for Testing

##### 1. Installing Wireshark

Wireshark is a popular packet capture tool that can be used alongside Snort for deeper traffic analysis. Install Wireshark on the **SnortIDSVM**:

```bash
sudo apt install wireshark
```

During installation, choose **"Yes"** when prompted to allow non-superusers to capture packets.

---

#### Downloading and Configuring Snort Rules

##### 1. Download Latest Snort Rules

You can download the latest Snort rules from [Snort’s official website](https://www.snort.org). For registered users, Snort provides the most up-to-date rule sets. Here’s how to download and extract them:

```bash
wget https://www.snort.org/rules/snortrules-snapshot-29190.tar.gz
sudo tar -xzvf snortrules-snapshot-29190.tar.gz -C /etc/snort/rules
```

##### 2. Modify the snort.conf File

After downloading the rules, make sure they are included in the `snort.conf` file:

```bash
include $RULE_PATH/snort.rules
```

---

### Troubleshooting and Fine-Tuning

While working with Snort, you may encounter several issues, such as rules not triggering or Snort not seeing traffic. Here are some common troubleshooting steps:

##### 1. Verifying Network Interface Monitoring

Make sure Snort is listening on the correct network interface. You can verify the interface name using:

```bash
ip a
```

Ensure that you are using the correct interface in the Snort command:

```bash
sudo snort -A console -q -c /etc/snort/snort.conf -i eth0
```

##### 2. Checking Rule Files

If Snort isn’t detecting attacks, ensure the rule files are correctly loaded. Run Snort in test mode to validate the rules:

```bash
sudo snort -T -c /etc/snort/snort.conf
```

Check for any missing or misconfigured rule files.

##### 3. Analyzing Traffic with tcpdump

Use **tcpdump** to check if the traffic is reaching the Snort IDS:

```bash
sudo tcpdump -i eth0 port 22
```

This will display SSH traffic and help verify that the network is working as expected.

---

### Conclusion

In this project, we successfully deployed Snort IDS on an Ubuntu server in Azure, configured detection rules, and simulated a brute-force attack using Hydra from a Kali Linux VM. Snort successfully detected the attack and generated alerts based on the rules configured.

The project demonstrates the power of Snort as an IDS for monitoring network traffic and detecting suspicious activities in real-time. This setup provides a foundation for further exploration into network security and IDS technologies.

---

### Future Work

While this project focused on deploying and configuring Snort for basic attack detection, there is much more that can be explored:

1. **Advanced Rule Writing**: Create more complex rules to detect a variety of attacks, such as SQL injection or malware propagation.
2. **Intrusion Prevention Mode**: Explore using Snort in **IPS mode**, where it can actively block attacks rather than just logging them.
3. **Integration with Other Tools**: Integrate Snort with SIEM tools like **Splunk** or **ELK Stack** for real-time log aggregation and analysis.
4. **Automation**: Automate Snort rule updates and system monitoring with scripts or configuration management tools like **Ansible**.
