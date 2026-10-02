# Incident 002 — SSH Connectivity Failure Caused by ISP Egress IP Mismatch

## Summary

During an AWS security lab exercise, SSH access to a public EC2 instance began timing out after the instance was stopped and started.

The EC2 instance itself remained healthy and HTTP access continued to work. The issue was traced to the client's ISP using a different public egress IP for the SSH connection than the IP returned by a public IP lookup service.

AWS VPC Flow Logs and CloudWatch Logs Insights were used to identify the actual source IP address seen by AWS.

## Environment

- Cloud Provider: AWS
- Region: Asia Pacific (Sydney)
- Service: Amazon EC2
- Network: Custom VPC
- Instance: Public-subnet web server
- Security Group: `security-lab-web-sg`
- Logging: VPC Flow Logs
- Analysis: CloudWatch Logs Insights
- Administrative Access: SSH and AWS Systems Manager Session Manager

## Initial Symptom

SSH attempts from the administrator's Mac timed out:

```text
ssh: connect to host <EC2_PUBLIC_IP> port 22: Operation timed out
```

A direct TCP connectivity test also timed out:

```text
nc -vz <EC2_PUBLIC_IP> 22
```

At the same time, HTTP access to the same instance worked successfully:

```text
curl http://<EC2_PUBLIC_IP>
<h1>Security Lab Web Server</h1><p>AWS EC2 is working.</p>
```

This showed that the instance, route table, internet gateway, public IP, and web service were functioning.

## Troubleshooting Process

### 1. Verified EC2 Health

The instance was running, passing AWS status checks, and reachable over HTTP.

### 2. Verified SSH Service

Using AWS Systems Manager Session Manager, the EC2 operating system was inspected.

`sshd` was active:

```text
Active: active (running)
```

The service was listening on port 22:

```text
0.0.0.0:22 LISTEN
[::]:22 LISTEN
```

### 3. Verified Host Firewall

Linux `firewalld` allowed both SSH and HTTP:

```text
services: ssh http
```

This ruled out the EC2 operating system and host firewall as the source of the timeout.

### 4. Verified Security Group

The AWS security group restricted SSH to a single `/32` administrator IP.

However, the IP returned by:

```bash
curl https://checkip.amazonaws.com
```

did not consistently match the source IP AWS observed for SSH traffic.

### 5. Used VPC Flow Logs

VPC Flow Logs showed repeated SSH traffic to:

```text
Destination: 10.0.1.21
Destination port: 22
Protocol: TCP
Action: REJECT
```

This proved the connection was being dropped at the AWS network/security layer before reaching `sshd`.

### 6. Used CloudWatch Logs Insights

A Logs Insights query was used to isolate rejected SSH traffic:

```sql
fields @timestamp, @message
| parse @message "* * * * * * * * * * * * * *" as version, account_id, interface_id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, start, end, action, log_status
| filter dstaddr = "10.0.1.21" and dstport = 22
| sort @timestamp desc
| display @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, action, log_status
| limit 50
```

The results showed the actual source IP AWS saw for the SSH traffic.

That IP differed from the public IP returned by the public IP lookup service.

## Root Cause

The administrator's ISP was using different NAT/egress IP addresses for different outbound connections.

As a result:

- The security group allowed one public IP.
- The SSH connection reached AWS from another public IP.
- AWS correctly rejected the SSH packets because the `/32` rule did not match.

This is consistent with ISP NAT or CGNAT-style behavior.

## Remediation

The SSH security-group rule was updated to allow the actual egress IP observed in VPC Flow Logs.

After the rule was updated, the TCP connectivity test succeeded:

```text
Connection to <EC2_PUBLIC_IP> port 22 [tcp/ssh] succeeded!
```

SSH access then worked successfully.

## Security Improvement

Although the SSH issue was resolved, the lab also demonstrated a stronger administrative-access design:

```text
Public Internet
    |
    +---- HTTP/HTTPS ----> EC2
    |
    X---- SSH 22

Administrator
    |
    v
AWS IAM
    |
    v
SSM Session Manager
    |
    v
EC2
```

Using AWS Systems Manager Session Manager removes the need to expose SSH publicly and avoids dependence on changing administrator public IP addresses.

## Security Principles Demonstrated

### Layered Troubleshooting

The issue was isolated by checking one layer at a time:

```text
Application
Operating system
Host firewall
Security group
VPC network telemetry
Client/ISP egress path
```

### Least Exposure

Restricting SSH to a `/32` source reduced exposure compared with allowing `0.0.0.0/0`.

### Network Visibility

VPC Flow Logs provided evidence about:

- Source IP
- Destination IP
- Destination port
- Protocol
- ACCEPT/REJECT decision

### Centralized Analysis

CloudWatch Logs Insights made it possible to filter flow-log data and quickly identify rejected SSH attempts.

## Lessons Learned

- A public IP lookup does not always guarantee that every outbound connection uses the same egress IP.
- Dynamic NAT or CGNAT can cause strict `/32` security-group rules to stop working unexpectedly.
- `Operation timed out` is a network-path symptom and should be investigated differently from `Permission denied`.
- HTTP success on the same EC2 instance can help rule out broader VPC routing problems.
- VPC Flow Logs are extremely useful for proving whether AWS accepted or rejected traffic.
- SSM Session Manager is a strong alternative to public SSH for administrative access.

## Incident Lifecycle

```text
SSH timeout
    ↓
Verify EC2 health
    ↓
Verify sshd
    ↓
Verify host firewall
    ↓
Inspect security group
    ↓
Analyze VPC Flow Logs
    ↓
Identify real egress IP
    ↓
Update rule
    ↓
Validate connectivity
```

## Status

**Resolved**
