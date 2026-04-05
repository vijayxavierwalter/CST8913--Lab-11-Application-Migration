# 📘 Tailwind Traders – Application Migration Plan to Azure

## 📌 Overview
Tailwind Traders is migrating a 3-tier on-premises application to Microsoft Azure. The environment includes:

- WEB01, WEB02 – Frontend (VMware)
- APP01 – Application/API server (VMware)
- SQL01 – SQL Server 2017 (Physical)
- LB01 – Internal Load Balancer  
- Additional: backups, firewall rules, strict 1-hour downtime

---

# ✅ Task 1 – Set Up Tooling for Discovery

## Discovery Approach
A hybrid (mix of agentless + agent-based) approach is recommended:
- Agentless → VMware servers (WEB, APP)
- Agent-based → SQL Server (physical machine)

Justification:
- Agentless is fast and non-intrusive for VMware
- Agent-based is required for deeper insights (SQL dependencies, performance)

## Number of Appliances
- 1 Azure Migrate appliance is sufficient

Why:
- All VMware resources are in a single environment
- One appliance can handle discovery + dependency mapping

## Required Credentials

| Purpose | Credentials Needed |
|--------|------------------|
| Software Inventory | Windows admin / domain credentials |
| SQL Discovery | SQL Server login (sysadmin or read access) |
| Dependency Mapping | OS-level credentials + agent installation permissions |

## Discovery Best Practices

1. Run discovery for at least 7–14 days  
2. Avoid peak changes during discovery  
3. Validate collected data regularly  

---

# ✅ Task 2 – Perform Assessment Planning

## Assessment Type
- Production assessment

Reason: Application is business-critical with strict downtime requirements.

## Target Region & Performance History
- Region: Canada Central
- Performance history: 30 days

## Sizing Method
- Performance-based sizing

Why:
- Ensures cost optimization
- Avoids over-provisioning
- Reflects real usage patterns

## Configuration Choices

| Setting | Selection | Justification |
|--------|----------|--------------|
| Comfort Factor | 1.2–1.3 | Adds buffer for peak load |
| Pricing Model | Reserved Instances | Cost savings |
| Licensing | Azure Hybrid Benefit | Reuse licenses |

---

# ✅ Task 3 – Dependency Analysis

## Application Components
- WEB01, WEB02
- APP01
- SQL01
- LB01

## Key Dependencies

1. HTTP/HTTPS (80/443)
2. TCP 8080/443
3. TCP 1433
4. Active Directory
5. DNS
6. Backup endpoints
7. Monitoring tools
8. Firewall rules

## Noise to Filter
- OS updates
- Monitoring chatter
- Test traffic

## Business Requirements

| Category | Details |
|---------|--------|
| business criticality | High |
| Uptime & downtime requirements | 1 hour |
| Data classification | Sensitive |
| Licensing dependencies | SQL + Windows |
| Patching requirements | Monthly |
| Firewall & IP considerations | Strict rules |

---

# ✅ Task 4 – Validate Assessment Results

## VM Sizes

| Tier | VM |
|-----|----|
| WEB | B2ms/B4ms |
| APP | D4s_v5 |
| SQL | E8s_v5 |

## Optimizations
- Replace LB with Azure Load Balancer
- Use Azure SQL MI
- Use Availability Zones

## Dependency Validation

All identified dependencies were validated using Azure Migrate dependency mapping and application owner input.

### ✔️ Confirmed Dependencies
- Web → App communication (HTTP/HTTPS: 80/443)
- App → SQL communication (TCP 1433)
- DNS resolution (internal and external)
- Active Directory authentication (LDAP/Kerberos)
- Load balancer routing (internal traffic flow)
- Backup services connectivity
- Monitoring and logging agents
- Firewall rules allowing inter-tier communication

### ✔️ Validation Outcome
- All critical application paths are accounted for
- No missing communication flows between tiers
- Required ports and protocols will be replicated in Azure NSGs
- External dependencies are confirmed reachable from Azure

## SLA
- 99.9%+
- Downtime achievable

## SQL Options
- Rehost (SQL on VM): Fastest lift-and-shift with no changes, but requires full management.
- Azure SQL Managed Instance: Best balance—high compatibility, minimal changes, and fully managed features.
- Azure SQL Database: Fully managed and scalable, but requires application redesign.

| Option | Notes |
|-------|------|
| Rehost | Fast |
| SQL MI | Best |
| SQL DB | Needs redesign |

---

# ✅ Task 5 – Migration Plan

## Pre-Migration
- Setup Azure
- Backup systems
- Configure replication

## Migration Steps (Per Server Group)

### Web Tier (WEB01, WEB02)      
The web tier will be migrated first as it is stateless and easier to validate.
- Configure replication for WEB01 and WEB02 using Azure Migrate
- Perform a test migration and validate web application accessibility
- Deploy Azure Load Balancer or Application Gateway
- Attach migrated web servers to the Azure load balancing service
- Verify connectivity between web servers and the application tier
- During cutover, stop on-premises web servers and start Azure instances
- Perform final validation of user access and load balancing behavior

### Application Tier (APP01)   
The application tier depends on the web tier and must be migrated after it.
- Enable replication for APP01 using Azure Migrate
- Conduct a test migration and validate application/API functionality
- Update configuration settings if required (e.g., service endpoints)
- Verify communication between web tier and application server
- During cutover, stop the on-premises application server and start the Azure VM
- Validate application services, logs, and integrations

### Database Tier (SQL01)    
The database tier is the most critical and will be migrated last. 
- Perform compatibility assessment for SQL Server 2017
- Take a full backup before migration
- Use Azure Database Migration Service or rehost strategy
- Execute a test migration and validate schema, data, and performance
- Pause application writes during final migration (cutover window)
- Complete final synchronization and bring Azure database online
- Update application connection strings to point to Azure
- Validate data integrity, performance, and transactions

## DNS
- Update to Azure
- Reduce TTL

## Post-Validation
- Test app
- Check DB
- Verify security

## Backout
- Revert DNS
- Restart on-prem

---

# ✅ Task 6 – Migration Waves

The migration is planned in multiple waves to reduce risk, maintain application availability, and meet the strict 1-hour downtime requirement.  
The approach follows a tier-based dependency model, migrating from least critical to most critical components.

## Justification for Grouping

### Dependencies
- Web Tier depends on Application Tier  
- Application Tier depends on Database Tier  
- Sequential migration ensures proper validation  

### Business Criticality
- SQL01 is the most critical (data layer)  
- APP01 handles business logic  
- WEB01/WEB02 are stateless and least critical  

### Downtime Restrictions
- 1-hour downtime window requires careful sequencing  
- Database migration is performed last to minimize impact  

### Testing Requirements
- Web tier can be tested independently  
- Application tier requires web tier  
- Database requires full-stack validation  

## Final Migration Waves

| Wave | Servers | Reason |
|-----|--------|-------|
| Wave 1 | WEB01, WEB02 | Stateless, low risk, easy rollback |
| Wave 2 | APP01 | Depends on web tier, moderate risk |
| Wave 3 | SQL01 | Highest criticality, requires downtime |

## Summary
This wave-based approach reduces risk, ensures proper dependency handling, and meets downtime constraints.
