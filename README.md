# AWS EC2 Auto Scaling

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/6e9fbdf7-45c5-46d7-84e8-80b30ba3f19d" />

## 1. Objective

The objective of this practical is to configure **EC2 Auto Scaling** and verify automatic scaling based on CPU utilization.

By completing this practical, you will learn how to:

* Launch and configure an EC2 instance.
* Install and configure a web server.
* Create an Amazon Machine Image (AMI).
* Create an EC2 Launch Template.
* Create an Auto Scaling Group (ASG).
* Configure minimum, desired, and maximum capacity.
* Configure a target tracking scaling policy.
* Generate CPU load on an EC2 instance.
* Monitor CPU utilization using Amazon CloudWatch.
* Verify scale-out and scale-in behavior.
* Understand the relationship between EC2, CloudWatch, Auto Scaling, and Load Balancing.

---

# 2. Architecture

The basic architecture used in this practical is:

```text
                         AWS Cloud
                             |
                             |
                    Auto Scaling Group
                             |
              +--------------+--------------+
              |                             |
              v                             v
         EC2 Instance 1              EC2 Instance 2
              |                             |
              +--------------+--------------+
                             |
                         CloudWatch
                             |
                      CPU Utilization
                             |
                    Auto Scaling Policy
                             |
                  +----------+----------+
                  |                     |
              CPU High               CPU Low
                  |                     |
              Scale Out               Scale In
                  |                     |
            Launch Instance       Terminate Instance
```

For a production environment, an Application Load Balancer can be placed in front of the Auto Scaling Group:

```text
                     Internet Users
                           |
                           v
                  Application Load
                      Balancer
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
           EC2-1         EC2-2         EC2-3
             ^             ^             ^
             |             |             |
             +-------------+-------------+
                           |
                    Auto Scaling Group
                           |
                       CloudWatch
```

---

# 3. Prerequisites

Before starting the practical, make sure you have:

* An AWS account.
* Access to the AWS Management Console.
* Basic knowledge of EC2.
* Basic Linux command-line knowledge.
* An EC2 key pair.
* A default VPC with available subnets.
* Permission to create EC2, AMI, Launch Template, Auto Scaling, and CloudWatch resources.

For this lab, use a small instance type suitable for your account and region, such as `t3.micro`, if available.

---

# 4. Step 1 — Launch an EC2 Instance

Open the AWS Management Console and navigate to:

**EC2 → Instances → Launch instances**

Configure the following:

### Name

```text
AutoScaling-Base-Server
```

### AMI

Select:

```text
Amazon Linux 2023
```

### Instance Type

For the lab:

```text
t3.micro
```

Use a different small instance type if `t3.micro` is unavailable or not appropriate for your account.

### Key Pair

Select an existing key pair.

Example:

```text
autoscaling-key
```

If you do not have one, create a new key pair and download the private key.

### Network

Select:

```text
VPC: Default VPC
Subnet: Default subnet
Auto-assign Public IP: Enable
```

### Security Group

Create a new security group:

```text
AutoScaling-SG
```

Add the following inbound rules:

| Type | Protocol | Port | Source    |
| ---- | -------- | ---: | --------- |
| SSH  | TCP      |   22 | My IP     |
| HTTP | TCP      |   80 | 0.0.0.0/0 |

For SSH, restrict access to your own IP address instead of allowing access from the entire internet.

Click:

**Launch Instance**

---

# 5. Step 2 — Verify the EC2 Instance

Go to:

**EC2 → Instances**

Select:

```text
AutoScaling-Base-Server
```

Verify:

```text
Instance state: Running
Status checks: 2/2 checks passed
```

Record the following information for reference:

```text
Instance ID:
Public IPv4 address:
Private IPv4 address:
Availability Zone:
Security Group:
```

---

# 6. Step 3 — Connect to the EC2 Instance

Select the instance and click:

**Connect**

Choose:

**EC2 Instance Connect**

Click:

**Connect**

A terminal session will open.

---

# 7. Step 4 — Update the Operating System

Run:

```bash
sudo dnf update -y
```

This updates installed packages on the Amazon Linux system.

---

# 8. Step 5 — Install Apache Web Server

Install Apache:

```bash
sudo dnf install httpd -y
```

Start Apache:

```bash
sudo systemctl start httpd
```

Configure Apache to start automatically after reboot:

```bash
sudo systemctl enable httpd
```

Verify the service:

```bash
sudo systemctl status httpd
```

Expected result:

```text
Active: active (running)
```

---

# 9. Step 6 — Create a Web Page

Create the web page:

```bash
sudo nano /var/www/html/index.html
```

Add:

```html
<!DOCTYPE html>
<html>
<head>
    <title>EC2 Auto Scaling Lab</title>
</head>
<body>
    <h1>EC2 Auto Scaling Lab</h1>
    <p>This web application is running on Amazon EC2.</p>
    <p>Auto Scaling practical environment.</p>
</body>
</html>
```

Save the file.

You can test the web server locally:

```bash
curl http://localhost
```

You should receive the HTML content.

---

# 10. Step 7 — Test the Web Server

Copy the EC2 instance's **Public IPv4 address**.

Open a browser:

```text
http://<EC2-PUBLIC-IP>
```

Example:

```text
http://3.XX.XX.XX
```

The web page should be displayed.

If it does not load, check:

```text
1. EC2 instance is running
2. Apache service is running
3. Security Group allows TCP port 80
4. Public IPv4 address is correct
```

---

# 11. Step 8 — Create an AMI

The configured EC2 instance will be used as the base image for future instances.

Navigate to:

**EC2 → Instances**

Select:

```text
AutoScaling-Base-Server
```

Select:

**Actions → Image and templates → Create image**

Configure:

### Image Name

```text
AutoScaling-WebServer-AMI
```

### Description

```text
Amazon Linux 2023 instance with Apache web server configured for Auto Scaling practical.
```

Click:

**Create image**

---

# 12. Step 9 — Verify the AMI

Navigate to:

**EC2 → AMIs**

Find:

```text
AutoScaling-WebServer-AMI
```

Wait until the status changes to:

```text
Available
```

Do not continue until the AMI is available.

---

# 13. Step 10 — Create a Launch Template

A Launch Template defines how new EC2 instances should be launched.

Navigate to:

**EC2 → Launch Templates → Create launch template**

### Launch Template Name

```text
AutoScaling-WebServer-Template
```

### AMI

Select:

```text
AutoScaling-WebServer-AMI
```

### Instance Type

```text
t3.micro
```

### Key Pair

Select:

```text
autoscaling-key
```

### Security Group

Select:

```text
AutoScaling-SG
```

### Tags

Add:

```text
Key: Name
Value: AutoScaling-WebServer
```

Click:

**Create launch template**

---

# 14. Step 11 — Verify Launch Template

Navigate to:

**EC2 → Launch Templates**

You should see:

```text
AutoScaling-WebServer-Template
```

Verify that it contains:

```text
AMI
Instance Type
Key Pair
Security Group
Tags
```

The Launch Template will be used by the Auto Scaling Group whenever a new EC2 instance needs to be launched.

---

# 15. Step 12 — Create the Auto Scaling Group

Navigate to:

**EC2 → Auto Scaling Groups**

Click:

**Create Auto Scaling group**

### Auto Scaling Group Name

```text
AutoScaling-WebServer-ASG
```

### Launch Template

Select:

```text
AutoScaling-WebServer-Template
```

Select the latest version.

Click:

**Next**

---

# 16. Step 13 — Configure Network

Select your VPC.

For example:

```text
VPC: Default VPC
```

Select multiple Availability Zone subnets if available.

For example:

```text
Subnet A
Subnet B
```

Using multiple Availability Zones improves availability because instances can be distributed across different Availability Zones.

Click:

**Next**

---

# 17. Step 14 — Configure Load Balancer

For this basic CPU scaling practical, you can select:

```text
No Load Balancer
```

Continue to the next step.

For a production architecture, an **Application Load Balancer** should generally be used to distribute application traffic across the EC2 instances in the Auto Scaling Group.

---

# 18. Step 15 — Configure Group Size

Configure:

```text
Minimum capacity: 1
Desired capacity: 1
Maximum capacity: 3
```

This means:

```text
Minimum = 1
Desired = 1
Maximum = 3
```

### Meaning

**Minimum capacity**

The Auto Scaling Group should normally maintain at least one instance.

**Desired capacity**

The number of instances the Auto Scaling Group should maintain under normal conditions.

**Maximum capacity**

The Auto Scaling Group cannot automatically scale beyond three instances under this configuration.

---

# 19. Step 16 — Configure Scaling Policy

Select:

```text
Target tracking scaling policy
```

For the metric, select:

```text
Average CPU utilization
```

Set the target value:

```text
50%
```

The configuration becomes:

```text
Metric:
Average CPU Utilization

Target:
50%

Minimum:
1

Desired:
1

Maximum:
3
```

The Auto Scaling Group will attempt to maintain the average CPU utilization around the configured target.

---

# 20. Step 17 — Create the Auto Scaling Group

Review the configuration.

You should have:

```text
Auto Scaling Group:
AutoScaling-WebServer-ASG

Launch Template:
AutoScaling-WebServer-Template

Minimum:
1

Desired:
1

Maximum:
3

Scaling Policy:
Target Tracking

Metric:
Average CPU Utilization

Target:
50%
```

Click:

**Create Auto Scaling Group**

---

# 21. Step 18 — Verify Auto Scaling

Navigate to:

**EC2 → Auto Scaling Groups**

Select:

```text
AutoScaling-WebServer-ASG
```

Check the **Instance management** section.

Initially you should have:

```text
Desired capacity: 1
Running instances: 1
```

Go to:

**EC2 → Instances**

You should see the instance created by the Auto Scaling Group.

---

# 22. Step 19 — Check Auto Scaling Activity

Inside the Auto Scaling Group, open:

**Activity**

You should see an event indicating that an instance was launched.

Typical sequence:

```text
Auto Scaling Group created
        |
        v
Desired capacity = 1
        |
        v
Launch EC2 instance
        |
        v
Instance becomes healthy
```

This confirms that the Auto Scaling Group is working.

---

# 23. Step 20 — Generate CPU Load

Connect to the EC2 instance created by the Auto Scaling Group.

Install the CPU stress testing utility:

```bash
sudo dnf install stress-ng -y
```

Start CPU load:

```bash
stress-ng --cpu 2 --timeout 10m
```

This command generates CPU workload for approximately 10 minutes.

Open another terminal session and monitor CPU:

```bash
top
```

You should observe high CPU utilization.

---

# 24. Step 21 — Monitor CPU Using CloudWatch

Navigate to:

**AWS Console → CloudWatch → Metrics → All metrics → EC2**

Select:

```text
Per-Instance Metrics
```

Find:

```text
CPUUtilization
```

Select the instance created by the Auto Scaling Group.

Monitor the CPU utilization.

You may see a pattern similar to:

```text
CPU Utilization

90% |             ********
80% |           **********
70% |         ************
60% |       **************
50% |------ Target --------
40% |
30% |
20% |
10% |
    +------------------------>
             Time
```

---

# 25. Step 22 — Observe Scale-Out

The target tracking policy is configured for:

```text
Target CPU = 50%
```

If CPU utilization remains sufficiently above the target, the Auto Scaling policy may increase the desired capacity.

For example:

```text
Before:

EC2-1
CPU = High

        |
        v

Auto Scaling Policy

        |
        v

Scale Out

        |
        v

EC2-1 + EC2-2
```

The Auto Scaling Group may change from:

```text
Desired capacity = 1
```

to:

```text
Desired capacity = 2
```

Check:

**EC2 → Auto Scaling Groups → Activity**

You should see an event indicating that a new instance was launched.

Then check:

**EC2 → Instances**

You should see another EC2 instance.

---

# 26. Step 23 — Verify Maximum Capacity

Continue generating load if required.

The Auto Scaling Group can scale up to:

```text
Maximum capacity = 3
```

Therefore, the group could reach:

```text
EC2-1
EC2-2
EC2-3
```

It will not automatically scale beyond three instances with the current configuration.

---

# 27. Step 24 — Stop the CPU Load

Stop the stress test:

```text
CTRL + C
```

Or allow the configured timeout to expire.

Check CPU:

```bash
top
```

CPU utilization should return toward normal levels.

---

# 28. Step 25 — Observe Scale-In

When CPU utilization remains below the target, the Auto Scaling policy can reduce the desired capacity.

For example:

```text
High workload:

EC2-1
EC2-2
EC2-3

        |
        v

Workload decreases

        |
        v

Auto Scaling

        |
        v

Scale In

        |
        v

EC2-1
```

The Auto Scaling Group should maintain at least:

```text
Minimum capacity = 1
```

Therefore, it should not automatically scale below one instance with this configuration.

---

# 29. Important Observation

Do not expect scaling to happen immediately after running the CPU stress command.

Auto Scaling uses CloudWatch metrics and scaling policies. The policy evaluates the workload and applies scaling behavior according to the configured policy and cooldown/warm-up behavior.

Therefore, allow sufficient time for:

```text
CPU Load
   ↓
CloudWatch Metric
   ↓
Scaling Policy Evaluation
   ↓
Auto Scaling Decision
   ↓
EC2 Launch/Termination
```

---

# 30. Practical Verification Table

| Test                         | Expected Result                      |
| ---------------------------- | ------------------------------------ |
| Launch base EC2              | Instance reaches Running state       |
| Install Apache               | Apache service becomes active        |
| Open public IP               | Web page loads                       |
| Create AMI                   | AMI becomes Available                |
| Create Launch Template       | Template created successfully        |
| Create ASG                   | ASG created                          |
| Desired capacity = 1         | One EC2 instance runs                |
| CPU increases                | CloudWatch reports high utilization  |
| CPU remains high             | ASG may scale out                    |
| Additional instance required | New EC2 instance launches            |
| CPU decreases                | CloudWatch reports lower utilization |
| Sustained low CPU            | ASG may scale in                     |
| Minimum capacity = 1         | At least one instance remains        |

---

# 31. Important AWS Components

## EC2

Amazon EC2 provides virtual servers that run applications.

In this practical:

```text
EC2 = Compute Server
```

---

## AMI

An Amazon Machine Image contains the configuration required to launch an EC2 instance.

In this practical:

```text
AMI
 |
 +-- Amazon Linux
 +-- Apache
 +-- Web Page
```

---

## Launch Template

A Launch Template defines how new EC2 instances should be configured.

Example:

```text
Launch Template
 |
 +-- AMI
 +-- Instance Type
 +-- Key Pair
 +-- Security Group
 +-- Tags
```

---

## Auto Scaling Group

The Auto Scaling Group manages the number of EC2 instances.

Example:

```text
Minimum = 1
Desired = 1
Maximum = 3
```

---

## CloudWatch

CloudWatch monitors AWS resources and collects metrics.

In this practical:

```text
EC2
 |
 CPU Utilization
 |
 v
CloudWatch
```

---

## Scaling Policy

The scaling policy determines when the Auto Scaling Group should increase or decrease capacity.

Example:

```text
Target CPU = 50%
```

---

# 32. Scale-Out vs Scale-In

| Feature | Scale-Out                | Scale-In                 |
| ------- | ------------------------ | ------------------------ |
| Purpose | Increase capacity        | Decrease capacity        |
| Trigger | Increased workload       | Reduced workload         |
| Action  | Launch EC2 instances     | Terminate EC2 instances  |
| Example | 1 → 2 instances          | 3 → 2 instances          |
| Benefit | Handles increased demand | Reduces unnecessary cost |

---

# 33. Important Interview Concept

A common misconception is:

> Auto Scaling distributes traffic between EC2 instances.

The technically correct explanation is:

> **EC2 Auto Scaling automatically adjusts the number of EC2 instances according to workload and scaling policies. A Load Balancer distributes incoming application traffic across the available EC2 instances.**

Therefore:

```text
Auto Scaling
      |
      | Manages number of instances
      v
EC2 Instances

Load Balancer
      |
      | Distributes application traffic
      v
EC2 Instances
```

---

# 34. Production Architecture

A more realistic AWS architecture would be:

```text
                    Internet
                       |
                       v
               Application Load
                  Balancer
                       |
              Target Group
                       |
          +------------+------------+
          |            |            |
          v            v            v
        EC2-1        EC2-2        EC2-3
          |            |            |
          +------------+------------+
                       |
                Auto Scaling Group
                       |
                 Scaling Policy
                       |
                   CloudWatch
```

The Auto Scaling Group automatically adds or removes instances while the Application Load Balancer distributes requests among healthy instances.

---

# 35. Troubleshooting

## Problem 1: Web page is not opening

Check:

```bash
sudo systemctl status httpd
```

If Apache is stopped:

```bash
sudo systemctl start httpd
```

Then verify Security Group:

```text
Inbound Rule
TCP
Port 80
Source: 0.0.0.0/0
```

---

## Problem 2: AMI is not available

Wait until the AMI status becomes:

```text
Available
```

Do not use an AMI while it is still being created.

---

## Problem 3: Auto Scaling is not launching another instance

Check:

```text
1. CPU utilization
2. CloudWatch metric
3. Scaling policy
4. Maximum capacity
5. Auto Scaling Activity
6. Instance health
```

Also remember that a short CPU spike may not immediately result in a scaling action.

---

## Problem 4: Instance immediately becomes unhealthy

Check:

```text
Security Group
Subnet
AMI
Instance type
Launch Template
IAM permissions
System status checks
```

---

# 36. Resource Cleanup

This is an important step because AWS resources can incur charges.

After completing the practical:

### Step 1

Go to:

**EC2 → Auto Scaling Groups**

Set:

```text
Desired capacity = 0
```

Then delete the Auto Scaling Group.

### Step 2

Delete unused EC2 instances.

### Step 3

Delete the Launch Template if it is no longer required.

### Step 4

Delete the AMI.

### Step 5

Check for associated EBS snapshots and delete unnecessary snapshots.

### Step 6

If you created a Load Balancer, Target Group, NAT Gateway, or Elastic IP for the lab, remove them if they are no longer required.

---

# 37. Interview Questions

### Q1. What is EC2 Auto Scaling?

**Answer:**

EC2 Auto Scaling is an AWS service that automatically adjusts the number of EC2 instances based on application demand, configured scaling policies, and capacity limits.

---

### Q2. What are Minimum, Desired and Maximum capacity?

**Answer:**

* **Minimum:** Lowest number of instances the Auto Scaling Group should maintain.
* **Desired:** Target number of instances under normal conditions.
* **Maximum:** Highest number of instances the Auto Scaling Group can automatically launch.

---

### Q3. What is an AMI?

**Answer:**

An AMI, or Amazon Machine Image, is a template containing the software and configuration required to launch EC2 instances.

---

### Q4. Why did we create an AMI?

**Answer:**

We created an AMI so that Auto Scaling could launch additional EC2 instances with the same operating system, Apache configuration, and application setup as the original server.

---

### Q5. What is a Launch Template?

**Answer:**

A Launch Template defines the configuration used to launch EC2 instances, including the AMI, instance type, key pair, security groups, and tags.

---

### Q6. What triggers Auto Scaling?

**Answer:**

Auto Scaling can be triggered by configured scaling policies based on CloudWatch metrics such as CPU utilization, network traffic, or application-specific metrics.

---

### Q7. What is scale-out?

**Answer:**

Scale-out means increasing the number of EC2 instances when application demand increases.

Example:

```text
1 EC2 → 2 EC2 → 3 EC2
```

---

### Q8. What is scale-in?

**Answer:**

Scale-in means reducing the number of EC2 instances when demand decreases.

Example:

```text
3 EC2 → 2 EC2 → 1 EC2
```

---

### Q9. What is the role of CloudWatch?

**Answer:**

CloudWatch collects and monitors metrics from AWS resources. In this practical, CPU utilization is monitored and used by the scaling policy to make scaling decisions.

---

### Q10. Does Auto Scaling distribute traffic?

**Answer:**

No. Auto Scaling manages the number of EC2 instances. A Load Balancer, such as an Application Load Balancer, distributes incoming traffic across healthy instances.

---

# 38. Final Practical Flow

The complete practical can be remembered as:

```text
EC2 Instance
     |
     v
Install Application
     |
     v
Create AMI
     |
     v
Create Launch Template
     |
     v
Create Auto Scaling Group
     |
     v
Configure
Min / Desired / Max
     |
     v
Configure Scaling Policy
     |
     v
Monitor CPU using CloudWatch
     |
     v
Generate CPU Load
     |
     v
Scale Out
     |
     v
Stop CPU Load
     |
     v
Scale In
     |
     v
Verify Results
     |
     v
Clean Up Resources
```

## Final Result

At the end of this practical, you should be able to demonstrate:

```text
Normal Workload
       |
       v
   1 EC2 Instance
       |
       |
High CPU / Workload
       |
       v
Auto Scaling Policy
       |
       v
   2 EC2 Instances
       |
       |
Higher Workload
       |
       v
   3 EC2 Instances
       |
       |
Workload Decreases
       |
       v
Scale In
       |
       v
   1 EC2 Instance
```

This gives you a complete hands-on understanding of **EC2 → AMI → Launch Template → Auto Scaling Group → CloudWatch → Scaling Policy → Scale-Out → Scale-In**.
