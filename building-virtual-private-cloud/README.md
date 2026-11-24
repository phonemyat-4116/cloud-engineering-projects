
# ☁️ AWS Multi-VPC Networking, Security & Observability

> A hands-on AWS project building and testing interconnected network architectures, implementing network security, and monitoring traffic withing infrastructure. 


## 🎯 Objectives

- Create and configure an **Amazon VPC and subnet**
- Understand **IPv4 and CIDR notation**
- Configure an **Internet Gateway and route table**
- Explore AWS networking through the **AWS Console and CLI**

## 🛠️ AWS Services & Tools

- Amazon VPC
- Subnets, Route Tables, Internet Gateway
- AWS Management Console

## 🏗️ Architecture

![AWS VPC Architecture](docs/architecture.png)


## ⚙️ Implementation

### Step 1: Create a VPC

First, I created an Amazon VPC to establish an isolated virtual network for AWS resources.

- **CIDR Block:** `10.0.0.0/16`

![VPC Configuration](screenshots/vpc-created.png)

---

### Step 2: Create a Subnet

After that, I created a subnet within the VPC to allocate a smaller IP address range to resources.

- **Subnet CIDR Block:** `10.0.0.0/24`
- **Availability Zone:** `apse1-az2 (ap-southeast-1a)`


![Subnet Configuration](screenshots/subnet-created.png)

---

### Step 3: Create an Internet Gateway

Then, I created an Internet Gateway and attached it to the VPC.

- **Purpose:** Provides a gateway for communication between the VPC and the internet.

![Internet Gateway](screenshots/internet-gateway.png)

---

### Step 4: Configure the Route Table

Here, I configured my public route table to control how network traffic is routed from the subnet.

- **Destination:** `my-new-igw`
- **Target:** `0.0.0.0/0`
- **Subnet Association:** `my-public-1`

**What I Learned Here is:** An Internet Gateway alone does not provide internet connectivity. The route table must direct the appropriate traffic to it, and additional network requirements must be satisfied.

![Route Table Configuration](screenshots/public-route-table.png)

---

### Step 5: Create a Security Group

Now, I created a security group to control inbound and outbound traffic for resources within my VPC.

- **Name:** `my-nw-security-gp`
- **Description:** A Security Group for the NextWork VPC.
- **VPC:** `NextWork VPC`

**Inbound Rule:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` |

**Outbound Rules:** Default rule allowing all outbound traffic.


![Security Group Configuration](screenshots/security-group.png)

**Note Here:** I created only security group here. Yet I haven't used it to launch EC2, which I will do so later. 

---

### Step 6: Create a Network ACL (NACL)

Created a custom Network ACL to control inbound and outbound traffic at the subnet level.

- **Name:** `my-public-nw-acl`
- **VPC:** `NextWork VPC`

**Inbound Rules:**

| Rule Number | Type | Source | Action |
|---|---|---|---|
| 100 | All Traffic | `0.0.0.0/0` | Allow |

![Network ACL Inbound](screenshots/public-nacl-inbound.png)

**Outbound Rules:**

| Rule Number | Type | Destination | Action |
|---|---|---|---|
| 100 | All Traffic | `0.0.0.0/0` | Allow |

![Network ACL Inbound](screenshots/public-nacl-outbound.png)

**Subnet Association:**
- Associated with: `my-public-1`


![Network ACL Configuration](screenshots/subnet-association.png)

---


### Step 7: Create a Private Subnet

Now, it's time to create a second subnet within the existing VPC to separate private resources from publicly accessible resources.

- **Name:** `my-private-1`
- **VPC:** `NextWork VPC`
- **IPv4 CIDR Block:** `10.0.1.0/24`
- **Availability Zone:** Second Availability Zone


![Private Subnet Configuration](screenshots/private-subnet.png)

**Similarily as above:** I have to associated my above private subnet to my newly created private route table "**private-nw-route-table**". 

![Route Table Subnet Association](screenshots/subnet-association-private.png)

Again, when I create a private subnet, there is also Network ACL created as default and already associated with that subnet, but I will create custom Network ACL **my-private-nw-acl**.

![Route Table Subnet Association](screenshots/private-nacl.png)

**Note:** The custom network ACLs start with denying all inbound and outbound traffic, but I will leave it for now, and will customize later. 

---


### Step 8: Launch instances in each subnet

After setting up the VPC as above, now it is ready to launch a **public server** and a **private server** in each subnet. 

![Public Server](screenshots/public-server.png)

![Private Server](screenshots/private-server.png)

✅ I created a separate **security group for the private server**, allowing inbound traffic only from the public subnet because internal communication is required, but outbound traffic to the private server is not.

![Private Security Group](screenshots/private-security-group.png)

---


### Step 9: Connect to Public Server (Testing Starts)

After launching both EC2 instances, I use **EC2 Instance Connect** to establish an SSH connection to my private server directly through **AWS Management Console**.

**Initial Connection Issues**

My first connection attempt failed because my public server's security only allow HTTP port as inboud traffic, not SSH. 

![SSH Connection Issue](screenshots/ssh-connection-issue.png)


**As a solution** for this, I updated the inbound rules of my public security group to allow SSH traffic.

![SSH Connection Solved](screenshots/public-sg-update-ssh.png)

*Note: Since this is a testing, SSH access is configured from Anywhere, but in the real-production (which will be in another project), SSH access should be restricted to trusted IP.*

**What I learned:**

- SSH provides encrypted remote acccess to server
- To use SSH, security group must allow this SSH traffic

---

### Step 10: Test Internal Connectivity Between EC2 Instances

In this section, I tested communication between my public server and private server with **ping** command.

```bash
ping 10.0.1.170
```

**Issue here:**
The ping command did not recieve any replies because the custom private Network ACL is still denying all inbound and outbound traffic.

![Ping Fail](screenshots/ping-fail-to-private-server.png)

![Private NACL Deny All Trafic](screenshots/private-nacl-deny.png)


**As a solution,** I updated the private Nacl to allow ICMP traffic from the public subnet.

*Outbound Rules:*

| Rule Number | Type | Source | Action |
|---|---|---|---|
| 100 | All ICMP - IPv4 | `10.0.0.0/24` | Allow |

*Outbound Rules:*

| Rule Number | Type | Destination | Action |
|---|---|---|---|
| 100 | All ICMP - IPv4 | `10.0.0.0/24` | Allow |


Along with this, I also updated the private server's security group to allow **ICMP traffic** from my public server's security group.

**Testing Result**
After updating the configuration, I tested the ping command to private server, and successfully received response from server.

![Connection to Private Server Success](screenshots/success-result.png)


**What I learned**:
- *ping* command uses ICMP to test the network connectivity.
- Both security groups and NACLs must permit the required traffic for successful communication.

---

### Step 11: Test Intenet Connectivity

After successfully tested the internal communication, I moved on to testing whether my public server could access external websites through the Internet Gateway.

I executed the following command.

```bash
curl example.com
```

The command successfully reterived the HTML response, which comfirmed my public server could communicate with internet. 

![Curl Success](screenshots/curl-success.png)

**Learning Note**
Both command test the connectivity, but in details,

- *ping*: Check one computer can connect another
- *curl*: Used to transfer data to and from a server

---





## 🔧 Challenges & Solutions

- **Challenge:** *Describe an issue you encountered.*
- **Solution:** *Explain how you resolved it.*

## 🚀 Future Improvements

- Deploy an EC2 instance and test connectivity
- Build public and private subnets
- Explore VPC Flow Logs and network security
- Automate infrastructure using Terraform

## 📚 References

- [NextWork — Build a Virtual Private Cloud](https://nextwork.ai/projects/7f907ac7-2473-5c15-a8be-edb36581cdba?track=high)
- [Amazon VPC Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
