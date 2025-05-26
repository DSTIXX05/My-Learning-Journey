# AWS Solutions Architect Lab: Application Load Balancer with RDS

**Lab Objective**:  
Deploy a scalable web application with an RDS database backend, using an Application Load Balancer (ALB) for traffic distribution.

## ✅ Tasks Completed

### Task 1: Create an Amazon RDS Database

- **Engine**: MySQL/Aurora (Multi-AZ enabled)
- **Configuration**:
  - DB instance class: `db.t3.micro`
  - Storage: 20GB GP2
  - Connectivity: Placed in private subnets with a security group allowing EC2 instances
- **Security**:  
  ![](![alt text](./Images/lab3/image.png))  
  _Configured security groups to allow EC2 instances to connect to RDS on port 3306_

---

### Task 2: Configure Application Load Balancer (ALB)

**Architecture**:  
![ALB Flow](![alt text](./Images/lab3/image-1.png))  
_ALB routes traffic to EC2 instances in private subnets_

1. **Target Group Creation**:

   - Protocol: HTTP (Port 80)
   - Health checks: `/index.php` endpoint
   - Registered EC2 instances as targets  
     ![](![alt text](./Images/lab3/image-2.png))

2. **ALB Setup**:
   - Scheme: Internet-facing
   - Listeners: HTTP (Port 80) → Forward to target group
   - Security group: Allow HTTP traffic from anywhere

---

### Task 3: Test Connectivity

1. Accessed ALB DNS (`http://my-alb-1234567890.us-east-1.elb.amazonaws.com`)
2. Verified web app connectivity to RDS:  
   ![](![alt text](./Images/lab3/image-3.png))  
   _Successful query to RDS from the application_

---

### Optional Task: Cross-Region Read Replica

- **Source DB**: `us-east-1`
- **Read Replica**: `us-west-2`  
  ![](![alt text](./Images/lab3/image-4.png))  
  _Enabled disaster recovery and read scaling across regions_

---

## 🔍 Key Learnings

1. **ALB vs RDS Placement**:
   - ALB in _public subnets_, RDS/EC2 in _private subnets_ for security.
2. **Target Groups**:
   - Must match instance health check paths.
3. **Cross-Region Replicas**:
   - Useful for DR and latency reduction.

---

## 🛠️ AWS Services Used

| Service | Purpose                  |
| ------- | ------------------------ |
| **RDS** | Managed MySQL database   |
| **ALB** | Distributes HTTP traffic |
| **EC2** | Hosts web application    |
| **VPC** | Network isolation        |

[View Full Lab Screenshots](./Images/lab3/) | [AWS Documentation](https://docs.aws.amazon.com/)
