# Incident 001 — Overly Permissive SSH Security Group

## Summary

During a controlled AWS security lab exercise, the EC2 security group was temporarily misconfigured to allow SSH access on TCP port 22 from `0.0.0.0/0`.

This configuration would allow any IPv4 address on the internet to attempt an SSH connection to the instance.

The EC2 instance was stopped before introducing the misconfiguration, so the vulnerable SSH service was not exposed while the rule was active.

## Environment

- Cloud Provider: AWS
- Region: Asia Pacific (Sydney)
- Service: Amazon EC2
- Network: Custom VPC
- Instance: Public-subnet web server
- Security Control: EC2 Security Group
- Detection Source: AWS CloudTrail

## Misconfiguration

The SSH rule was changed from a trusted single-host `/32` source to:

```text
Protocol: TCP
Port: 22
Source: 0.0.0.0/0
```

This created an overly permissive inbound rule.

## Risk

Allowing SSH from `0.0.0.0/0` increases attack surface because any internet host can attempt to reach the SSH service.

Possible risks include:

- Password or key brute-force attempts
- Automated internet scanning
- Exploitation attempts against the SSH service
- Increased exposure if credentials or SSH configuration are weak

## Detection

AWS CloudTrail recorded the configuration change.

The relevant event was:

```text
Event name: ModifySecurityGroupRules
Event source: ec2.amazonaws.com
Read-only: false
Protocol: TCP
From port: 22
To port: 22
CIDR: 0.0.0.0/0
```

The raw CloudTrail event confirmed that the security group rule had been modified to allow SSH from all IPv4 addresses.

## Investigation

The investigation focused on:

1. Identifying the affected security group
2. Reviewing the CloudTrail event time
3. Confirming the source identity that made the change
4. Checking the source IP address
5. Reviewing the request parameters
6. Confirming that TCP port 22 was opened to `0.0.0.0/0`

Sensitive account identifiers, access-key values, and personal IP addresses are intentionally omitted from this public report.

## Remediation

The SSH rule was changed back to a trusted single-host `/32` source.

Final SSH rule:

```text
Protocol: TCP
Port: 22
Source: Trusted administrator IP /32
```

The EC2 instance was started only after the insecure rule had been removed.

## Validation

After remediation:

- SSH was no longer exposed to the entire internet
- The security group allowed SSH only from the trusted administrator IP
- HTTP remained publicly available on TCP port 80 for the web server
- CloudTrail retained evidence of both the insecure change and the remediation

## Security Principles Demonstrated

### Least Privilege

Network access should be limited to only the sources and ports required.

### Defense in Depth

The EC2 instance was protected using multiple layers:

- EC2 Security Group
- Linux `firewalld`
- SSH key-based authentication
- Password authentication disabled
- Direct root SSH login disabled

### Auditability

CloudTrail provided evidence showing who changed the security group, when it happened, and what parameters were modified.

## Lessons Learned

- Public SSH access should not use `0.0.0.0/0` unless there is a specific controlled requirement.
- CloudTrail is valuable for investigating cloud configuration changes.
- Security-group changes should be reviewed and monitored because a single rule can significantly change the attack surface.
- Sensitive values should be redacted before publishing screenshots or incident reports publicly.

## Incident Lifecycle

```text
Misconfiguration
      ↓
CloudTrail Event
      ↓
Investigation
      ↓
Risk Identified
      ↓
Remediation
      ↓
Validation
```

## Status

**Resolved**
