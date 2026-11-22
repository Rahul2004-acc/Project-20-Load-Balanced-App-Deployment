# Project 20: AWS Load Balanced Application Deployment

## **Objective**

Deploy a highly available web application using AWS Application Load Balancer (ALB) to distribute incoming HTTP traffic across multiple EC2 instances, ensuring fault tolerance and scalability.

---

## **Architecture Overview**

This project implements a classic multi-tier load-balanced architecture on AWS:

```
                    ┌─────────────────────┐
                    │   Internet Traffic  │
                    │      (HTTP:80)      │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Application Load   │
                    │   Balancer (ALB)    │
                    │  (Internet-facing)  │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
         ┌──────▼─────┐   ┌────▼──────┐
         │   EC2-1    │   │   EC2-2    │
         │ ap-south-1a│   │ ap-south-1b│
         │  (Apache)  │   │  (Apache)  │
         └────────────┘   └────────────┘
```

### **Key Components**

- **Application Load Balancer (ALB)**
  - Internet-facing ALB listening on HTTP port 80
  - Distributes incoming traffic across registered targets
  - Performs health checks every 30 seconds

- **EC2 Instances**
  - Instance 1: `ap-south-1a` (Availability Zone A)
  - Instance 2: `ap-south-1b` (Availability Zone B)
  - Both running Amazon Linux 2023
  - Apache HTTP Server installed and configured on both instances

- **Target Group**
  - Both EC2 instances registered as targets
  - Health check protocol: HTTP
  - Health check port: 80
  - Health check path: `/`
  - Healthy threshold: 2 consecutive successful checks
  - Unhealthy threshold: 2 consecutive failed checks

- **Security Groups**
  - **ALB Security Group**: Allows inbound HTTP traffic (port 80) from the internet
  - **EC2 Security Group**: Allows inbound HTTP traffic (port 80) from ALB's security group

---

## **Technologies Used**

| Technology | Purpose |
|-----------|---------|
| **AWS EC2** | Virtual compute instances |
| **AWS Application Load Balancer** | Load balancing and traffic distribution |
| **AWS Target Groups** | Grouping backend instances for health checks and routing |
| **AWS Security Groups** | Network access control |
| **Amazon Linux 2023** | Operating system for EC2 instances |
| **Apache HTTP Server** | Web server application |
| **AWS CLI** | Command-line interface for AWS operations |
| **Linux Shell Scripting** | Automation and configuration scripts |

---

## **Implementation Steps**

### **1. Network Setup**
- Create a VPC with public subnets in multiple Availability Zones
- Configure Internet Gateway for internet connectivity
- Create security groups for ALB and EC2 instances

### **2. EC2 Instance Deployment**
- Launch two EC2 instances (t2.micro) in separate Availability Zones
- Assign elastic IPs for persistent connectivity
- Apply appropriate security groups

### **3. Apache Web Server Configuration**
- Update system packages: `sudo yum update -y`
- Install Apache: `sudo yum install -y httpd`
- Create unique index.html files on each instance
- Enable and start Apache service:
  ```bash
  sudo systemctl enable httpd
  sudo systemctl start httpd
  ```

### **4. Load Balancer Setup**
- Create an Application Load Balancer (internet-facing)
- Configure listeners on port 80
- Create target group with health check configuration
- Register both EC2 instances as targets

### **5. Health Check Configuration**
- Protocol: HTTP
- Port: 80
- Path: `/` (or `/index.html`)
- Interval: 30 seconds
- Timeout: 5 seconds
- Healthy threshold: 2
- Unhealthy threshold: 2

### **6. Security Group Configuration**
- **ALB Security Group Inbound Rules:**
  - Protocol: TCP, Port: 80, Source: 0.0.0.0/0 (Allow HTTP from anywhere)

- **EC2 Security Group Inbound Rules:**
  - Protocol: TCP, Port: 80, Source: ALB Security Group (Allow HTTP only from ALB)

---

## **Final Outcome**

✅ **Successfully Completed**

### **Verification Results**

- **EC2 Instances**
  - Both instances are running and healthy
  - Instance 1 (ap-south-1a): Status = Running
  - Instance 2 (ap-south-1b): Status = Running

- **Target Group Health**
  - Target 1: **Healthy** ✓
  - Target 2: **Healthy** ✓
  - Health check response: HTTP 200 OK

- **Load Balancer**
  - ALB Status: Active
  - Listener (HTTP:80): Active
  - Target Group: 2/2 targets healthy

- **Traffic Distribution**
  - Successfully load-balanced HTTP requests across both instances
  - Each instance responds with its unique identifier
  - Round-robin distribution verified

### **Testing Results**

```bash
# Test ALB DNS endpoint
curl -s http://<ALB-DNS-Name> | grep "Server"

# Output alternates between:
# Server: Instance 1 - ap-south-1a
# Server: Instance 2 - ap-south-1b
```

---

## **Key Learning Outcomes**

### **Concepts Mastered**

1. **Load Balancing**
   - Distributing incoming traffic across multiple backend servers
   - Benefits of load balancing for scalability and performance

2. **High Availability**
   - Deploying resources across multiple Availability Zones
   - Automatic failover capabilities
   - Zero-downtime traffic distribution

3. **Fault Tolerance**
   - Redundancy through multiple EC2 instances
   - Health checks and automatic recovery
   - Single point of failure mitigation

4. **Health Checks**
   - Automatic detection of unhealthy targets
   - Configuration of check intervals and thresholds
   - Impact on traffic distribution

5. **AWS Networking**
   - Security group configuration and best practices
   - Least privilege access control
   - Target group routing mechanics

6. **Traffic Distribution**
   - Round-robin algorithm
   - Request routing based on target health
   - Connection draining for graceful shutdowns

---

## **Commands Reference**

### **Check ALB Status**
```bash
aws elbv2 describe-load-balancers --region ap-south-1
```

### **Check Target Health**
```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region ap-south-1
```

### **View EC2 Instance Details**
```bash
aws ec2 describe-instances --region ap-south-1
```

### **Test Load Balancer Endpoint**
```bash
# Single request
curl -s http://<ALB-DNS-Name>

# Multiple requests to observe round-robin
for i in {1..5}; do curl -s http://<ALB-DNS-Name> | grep -i "server"; done
```

### **SSH into EC2 Instance**
```bash
ssh -i <KEY_PAIR.pem> ec2-user@<INSTANCE_PUBLIC_IP>
```

---

## **Troubleshooting Guide**

### **Issue: Health Checks Failing**

**Symptom:** Target status shows "unhealthy"

**Root Cause:** Security group rules not allowing traffic from ALB

**Solution:**
1. Identify ALB's security group ID
2. Update EC2 security group inbound rule:
   ```bash
   aws ec2 authorize-security-group-ingress \
     --group-id <EC2_SG_ID> \
     --protocol tcp \
     --port 80 \
     --source-group <ALB_SG_ID> \
     --region ap-south-1
   ```

### **Issue: One Instance Unhealthy**

**Troubleshooting Steps:**
1. Check Apache service status:
   ```bash
   sudo systemctl status httpd
   ```
2. Verify security group rules
3. Check instance's health check path exists:
   ```bash
   curl -s http://localhost/ | head -20
   ```

### **Issue: Uneven Load Distribution**

**Consideration:** ALB uses flow hash algorithm and may not show perfect 50-50 split on small test sets. Test with larger sample sizes.

---

## **Cost Optimization Tips**

- Use **t2.micro** instances (eligible for AWS Free Tier)
- Configure **ALB to terminate idle connections** after 60 seconds
- Use **Scheduled scaling** if traffic is predictable
- Monitor with **CloudWatch** to identify optimization opportunities

---

## **Security Best Practices Implemented**

✅ **Principle of Least Privilege**
- EC2 instances only receive traffic from ALB security group
- ALB only accepts traffic on necessary ports

✅ **Multi-Availability Zone Deployment**
- No single point of failure
- Automatic failover protection

✅ **Health Checks**
- Automatic removal of unhealthy instances from load balancing
- Quick detection of application failures

✅ **Security Group Segmentation**
- Separate security groups for ALB and EC2
- Explicit ingress rules

---

## **Next Steps for Enhancement**

1. **HTTPS/TLS Configuration**
   - Add SSL/TLS certificate to ALB
   - Redirect HTTP to HTTPS

2. **Auto Scaling**
   - Create Auto Scaling Group
   - Set scaling policies based on CPU/memory metrics

3. **Advanced Monitoring**
   - Set up CloudWatch alarms
   - Configure SNS notifications
   - Create custom dashboards

4. **Database Integration**
   - Add RDS database backend
   - Configure connection pooling

5. **CI/CD Pipeline**
   - Automate instance provisioning
   - Implement blue-green deployments

6. **Global Load Balancing**
   - Use Route 53 for DNS failover
   - Distribute across multiple regions

---

## **Resources**

- [AWS Application Load Balancer Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/)

---

## **Summary**

This project demonstrates the core principles of cloud infrastructure design, emphasizing high availability, fault tolerance, and efficient traffic distribution. By deploying an Application Load Balancer with multiple EC2 instances across Availability Zones, we've created a resilient, scalable web application architecture that can handle increased traffic and recover from instance failures seamlessly.

**Status:** ✅ **Complete and Verified**

---

*Last Updated: November 22, 2026*
