# Automated-Web-Tier-High-Availability-with-ALB-and-ASG
This is a scalable application architecture which scales up upon an increase in traffic as the CPU utilisation turns out to be >= 50% and scales in as traffic decreases.

## Architecture Overview

```plain
								  [ Internet Traffic ]
										   │
										   ▼
							  [ Application Load Balancer ]
										   │
							 ┌─────────────┴─────────────┐
							 ▼                           ▼
					[ EC2 Instance 1 ]          [ EC2 Instance 2 (Scaled) ]
					(Always Running / Min: 1)       (Spins up if CPU >= 50%)
							 │                           │
							 └─────────────┬─────────────┘
										   ▼
								 [ Amazon CloudWatch ] 
							(Monitors CPU & Triggers ASG)
```

## Step 1: Create a Custom VPC Security Group

Before creating the load balancer or instances, we need a Security Group that allows web traffic to flow smoothly.

1. Open the **AWS Management Console** and navigate to **VPC > Security groups**.
2. Click **Create security group**.
3. Configure the following details:
    - **Security group name**: ```web-tier-sg```
	- **Description**: ```Allow HTTP and SSH access```
	- **VPC**: Select your default VPC.
4. Under Inbound rules, add the following three rules:
    - Rule 1: Type: ```HTTP``` | Source: ```Anywhere-IPv4 (0.0.0.0/0)```
	- Rule 2: Type: ```HTTPS``` | Source: ```Anywhere-IPv4 (0.0.0.0/0)```
	- Rule 3: Type: ```SSH``` | Source: ```My IP``` (Crucial for secure access later)
5. Click Create security group.

## Step 2: Create the Target Group & Application Load Balancer

The Load Balancer acts as the front door, distributing traffic to the instances managed by the ASG.

## Part A: Create a Target Group
1. Navigate to **EC2 > Target groups** (under Load Balancing in the left menu).
2. Click **Create target group**.
3. Choose **Instances** as the target type.
4. **Target group name**: web-target-group
5. Protocol: ```HTTP``` | Port: ```80```
6. Click **Next** (Do not manually register any instances here; the ASG will do this automatically later).
7. Click **Create target group**.

## Part B: Create the Load Balancer
1. Navigate to **EC2 > Load balancers** and click Create load balancer.
2. Select **Application Load Balancer (ALB)** and click **Create**.
3. **Load balancer name**: ```web-alb```
4. **Scheme**: ```Internet-facing``` | IP address type: ```IPv4```
5. **Network mapping**: Select your VPC and check at least two Subnets/Availability Zones (e.g., sa-east-1a and sa-east-1b).
6. **Security groups**: Remove the default group and select your newly created ```web-tier-sg```.
7. **Listeners and routing**: Under Protocol HTTP Port 80, set the default action to **Forward to target groups** and select **web-target-group**.
8. Click **Create load balancer**. *Note the DNS Name (e.g., ```://amazonaws.com```) once it shifts to an "Active" state*.

## Step 3: Create a Launch Template

A Launch Template tells the Auto Scaling Group exactly what kind of machine to spin up when scaling out.

1. Navigate to **EC2 > Launch templates** and click **Create launch template**.
2. **Launch template name**: ```web-launch-template```
3. **Application and OS Images (AMI)**: Select **Amazon Linux 2023 AMI** (Free tier eligible).
4. **Instance type**: Select ```t3.micro``` or ```t2.micro```.
5. **Key pair**: Select your existing SSH key pair.
6. **Network settings**: Select an existing security group and choose ```web-tier-sg```.
7. Scroll down to the very bottom and expand Advanced details.
8. Scroll to the ```User data``` text field at the absolute bottom and paste the following bash script. This script automatically installs an Apache web server, sets up a visual status homepage, and installs the utility we need to stress test the CPU later:
```bash
#!/bin/bash
# Update OS and install Apache Web Server + Stress engine
dnf update -y
dnf install -y httpd stress

# Start and enable Apache web server
systemctl start httpd
systemctl enable httpd

# Create a dynamic homepage showing system stats and instance details
INSTANCE_ID=$(curl -s http://169.254.169)
AVAILABILITY_ZONE=$(curl -s http://169.254.169)

cat <<EOF > /var/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>ASG Scaling Test Tier</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 100px; background-color: #f4f6f9; }
        .card { display: inline-block; padding: 30px; background: white; border-radius: 10px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
        h1 { color: #FF9900; }
        span { font-weight: bold; color: #232F3E; }
    </style>
</head>
<body>
    <div class="card">
        <h1>Hello from Your Automated Web Tier!</h1>
        <p>Instance ID: <span>$INSTANCE_ID</span></p>
        <p>Availability Zone: <span>$AVAILABILITY_ZONE</span></p>
    </div>
</body>
</html>
EOF
```
9. Click **Create Launch Template**


## Step 4: Create the Auto Scaling Group (ASG) with 50% CPU Dynamic Policy

Now we tie the Launch Template and Load Balancer together with a dynamic rule that triggers scaling at ≥ 50% CPU usage.

1. Navigate to **EC2 > Auto Scaling Groups** and click **Create Auto Scaling group**.
2. Name: ```web-asg``` | **Launch template**: Choose web-launch-template and click Next.
3. **Network**: Choose your VPC and select the **same subnets** you assigned to your Load Balancer in Step 2. Click **Next**.
4. **Configure advanced options**:
    - Under **Load balancing**, check **Attach to an existing load balancer**.
    - Choose **Select from your Amazon EC2 Auto Scaling target groups** and pick ```web-target-group```.
	- Under **Health checks**, check **Elastic Load Balancing (ELB)** health checks. Click **Next**.
5. Configure group size and scaling policies:
    - **Desired capacity**: ```1```
	- **Minimum capacity**: ```1```
	- **Maximum capacity**: ```3``` (This keeps 1 machine baseline, but allows scaling up to 3 during traffic spikes)
6. **Automatic scaling policies**:
    - Select **Target tracking scaling policy**.
	- **Metric type**: ```Average CPU utilisation```
	- **Target value**: ```50``` (This dictates that as CPU utilisation reaches ≥ 50%, it triggers the ASG to scale out)
	- **Instance warmup**: ```60``` seconds.
7. Click **Next**, skip notifications/tags, and **click Create Auto Scaling group**.
Observe: Within 1-2 minutes, the ASG will launch your very first base instance automatically. You can confirm this by going to **EC2 > Instances**.

## Step 5: Test and Spike Traffic to Observe Automatic Scaling

Now we will simulate a high-traffic production scenario by forcing the single baseline server's CPU to jump to 100%.

1. Verify the Load Balancer Front Door
Copy the **DNS Name** of your ALB (from Step 2B) and paste it into a web browser. You should see a white screen with an AWS card reading: "Hello from Your Automated Web Tier!" along with the specific Instance ID.
2. **Connect and Inject the CPU Spike**
   - Go to your EC2 Instances dashboard and locate the running instance created by the ASG.
   - Connect to it via SSH using your terminal:
```bash
ssh -i your-key.pem ec2-user@your-instance-public-ip
```
   - Run the ```stress``` application we installed via user data to spin up intensive mathematical tasks across 4 CPU cores:
```bash
sudo stress --cpu 4 --timeout 600
```
*(This utility commands the OS to redline the CPU at 100% capacity for 10 minutes)*.

## Step 6: Monitor the Results (What to document for GitHub)

Leave the terminal running and return to your AWS console dashboard to capture screenshots for your GitHub portfolio repository.

## Phase A: Scale-Out Activity (The Expansion)
1. Navigate to **CloudWatch > Alarms**. You will see a target tracking alarm automatically generated by the ASG (e.g., ```TargetTracking-web-asg-AlarmHigh```).
2. Within 2-3 minutes of running the stress script, the alarm will transition to In Alarm as average CPU crosses the 50% threshold.
3. Go back to **EC2 > Auto Scaling Groups**, click on ```web-asg```, and open the Activity tab. You will see a chronological log line reading: *“Launching a new EC2 instance... status: InProgress”*.
4. Check your **EC2 Instances page**. You will observe Instance 2 and Instance 3 spinning up automatically to distribute the heavy load.
5. Refresh your ALB browser tab repeatedly. You will notice the displayed **Instance ID** changes dynamically as the load balancer splits traffic across the new servers.

## Phase B: Scale-In Activity (The Cool-Down)
1. Go back to your SSH terminal. If the 10-minute timeout hasn't finished, stop the stress tool manually by typing ```Ctrl + C```.
2. The instance's CPU utilization will immediately drop back down to 0-1%.
3. Within several minutes, CloudWatch will trigger the low-threshold alarm (```AlarmLow```).
4. Look at the Activity history of the ASG. You will see it log a reduction request, initiating termination steps for the extra instances.
5. Return to the **EC2 Instances menu**. The additional instances will display a status of **Shutting-down / Terminated**, leaving **exactly one machine running** as your baseline tier.









