# Advanced Cloud Computing with Azure: Hands-On Lab Series

## 📚 Technical Introduction

This comprehensive lab series provides hands-on experience with Microsoft Azure, covering fundamental to advanced cloud computing concepts. Through 16 progressive sessions, you'll gain practical skills in deploying, managing, and optimizing cloud infrastructure across compute, storage, databases, security, containers, serverless, DevOps, AI/ML, networking, and cost optimization.

---

**Prerequisites:**
- Basic understanding of cloud computing concepts
- Familiarity with command line interfaces (Bash, Azure CLI, PowerShell)
- Programming knowledge (Python, Node.js, or similar)
- Azure account (Free Tier eligible for most labs)

---

<details>
  <summary>
  Lab Session 01: Azure Compute Foundations – Virtual Machines, Networking, and Application Deployment 
  </summary>

  Gain hands-on experience across multiple Azure compute services by progressively building, securing, and deploying applications using Virtual Machines, Azure Web Apps, and App Service.

  **Labs for this session:**
  - [lab_1_a_vm-on-custom-vnet.md](session01/lab_1_a_vm-on-custom-vnet.md):  
    *Launch and configure a Linux Virtual Machine in a custom Virtual Network, including subnet, network security group, and public IP setup.*
  - [lab_1_b_deploy_secure_multi-vm_python_api.md](session01/lab_1_b_deploy_secure_multi-vm_python_api.md):  
    *Deploy multiple Python Flask APIs on Virtual Machines across different subnets, secure them with Network Security Groups and Application Security Groups, and access each API directly using public IP-based endpoints.*
  - [lab_1_c_host-static-website-storage.md](session01/lab_1_c_host-static-website-storage.md):  
    *Deploy and host a static website using Azure Storage Account static website hosting.*
  - [lab_1_d_deploy-python-app-appservice.md](session01/lab_1_d_deploy-python-app-appservice.md):  
    *Deploy a Python application using Azure App Service for managed scaling and deployment.*

</details>

<details>
  <summary>Lab Session 02: Identity and Access Control with Azure AD and RBAC</summary>

  Master Azure identity and access management by implementing least-privilege security, multi-factor authentication, role-based access control, and managed identities.

  **Labs for this session:**
  - [lab_2_a_azure-ad-users-rbac.md](session02/lab_2_a_azure-ad-users-rbac.md):  
    *Create Azure AD users, groups, and assign built-in RBAC roles following least-privilege principles. Configure custom roles and test permissions using Azure portal and CLI.*
  - [lab_2_b_managed-identities-keyvault.md](session02/lab_2_b_managed-identities-keyvault.md):  
    *Configure managed identities for Azure resources (VMs, App Services) to securely access Key Vault secrets without storing credentials in code. Implement system-assigned and user-assigned identities.*
  - [lab_2_c_conditional-access-mfa.md](session02/lab_2_c_conditional-access-mfa.md):  
    *Implement conditional access policies with multi-factor authentication (MFA) for privileged operations. Configure trusted locations, device compliance, and risk-based access policies.*
  - [lab_2_d_service-principals-rbac.md](session02/lab_2_d_service-principals-rbac.md):  
    *Create service principals for application authentication, configure RBAC permissions, and implement certificate-based authentication for automated workflows.*

</details>

<details>
  <summary>Lab Session 03: Cloud Storage Solutions – Blob Storage, Managed Disks, and Azure Files</summary>

  Master Azure storage services by implementing object storage with Blob Storage, block storage with Managed Disks, and shared file systems with Azure Files. Configure lifecycle policies, encryption, and high availability architectures.

  **Labs for this session:**
  - [lab_3_a_blob-storage-lifecycle.md](session03/lab_3_a_blob-storage-lifecycle.md):  
    *Configure Blob Storage containers with versioning, encryption (Microsoft-managed, customer-managed keys), access policies, and lifecycle management rules for tier transitions (Hot, Cool, Archive).*
  - [lab_3_b_managed-disk-snapshots.md](session03/lab_3_b_managed-disk-snapshots.md):  
    *Create and manage Azure Managed Disks, attach to Virtual Machines, perform snapshots, restore disks, and configure disk encryption with Azure Disk Encryption.*
  - [lab_3_c_static-website-cdn.md](session03/lab_3_c_static-website-cdn.md):  
    *Deploy a static website on Blob Storage with global distribution through Azure CDN. Configure custom domains, HTTPS delivery, caching rules, and content purging.*
  - [lab_3_d_azure-files-shared-storage.md](session03/lab_3_d_azure-files-shared-storage.md):  
    *Create Azure Files file shares with SMB and NFS protocols. Mount shares on Windows and Linux VMs, configure Active Directory authentication, implement snapshots, and test concurrent access.*

</details>

<details>
  <summary>Lab Session 04: Database Services – SQL Database, Cosmos DB, and Azure Cache for Redis</summary>

  Master Azure database services by implementing relational databases with high availability, globally distributed NoSQL databases with Cosmos DB, and in-memory caching with Azure Cache for Redis.

  **Labs for this session:**
  - [lab_4_a_sql-database-private-endpoint.md](session04/lab_4_a_sql-database-private-endpoint.md):  
    *Provision Azure SQL Database in a private subnet using Private Endpoints. Configure firewall rules, connect from Virtual Machine, create tables, and perform SQL operations.*
  - [lab_4_b_cosmos-db-api.md](session04/lab_4_b_cosmos-db-api.md):  
    *Build and query a Cosmos DB database using SQL API and Azure CLI. Work with partition keys, queries, indexing policies, and configure global distribution across regions.*
  - [lab_4_c_sql-database-failover-groups.md](session04/lab_4_c_sql-database-failover-groups.md):  
    *Deploy Azure SQL Database with auto-failover groups for high availability. Configure geo-replication across regions, test automatic failover, and validate data consistency.*
  - [lab_4_d_redis-cache-sessions.md](session04/lab_4_d_redis-cache-sessions.md):  
    *Create Azure Cache for Redis for high-performance in-memory caching. Integrate with Flask application for session management and test Redis operations (SET, GET, EXPIRE).*

</details>

<details>
  <summary>Lab Session 05: Load Balancing and Auto Scaling</summary>

  Implement high availability and elasticity using Azure Load Balancer, Application Gateway, and Virtual Machine Scale Sets with Azure Monitor integration. Learn automatic scaling based on demand and global traffic distribution with Traffic Manager.

  **Labs for this session:**
  - [lab_5_a_load-balancer-deployment.md](session05/lab_5_a_load-balancer-deployment.md):  
    *Deploy web application behind Azure Load Balancer with zone redundancy. Configure backend pools, health probes, load balancing rules, and test automatic failover.*
  - [lab_5_b_vmss-autoscaling-monitor.md](session05/lab_5_b_vmss-autoscaling-monitor.md):  
    *Create Virtual Machine Scale Set with autoscaling rules based on CPU metrics and schedule-based scaling. Monitor scaling activities through Azure Monitor and test automatic capacity adjustment.*
  - [lab_5_c_application-gateway-waf.md](session05/lab_5_c_application-gateway-waf.md):  
    *Deploy Application Gateway with Web Application Firewall (WAF) for HTTP(S) load balancing. Configure path-based routing, SSL termination, WAF rules, and integrate with VMSS for automatic scaling.*
  - [lab_5_d_traffic-manager-multi-region.md](session05/lab_5_d_traffic-manager-multi-region.md):  
    *Implement global high availability with Traffic Manager routing across multiple regions. Deploy identical applications in different regions, configure priority routing for disaster recovery, and test automatic failover.*

</details>

<details>
  <summary>Lab Session 06: Container Orchestration with Azure Container Instances and AKS</summary>

  Master container deployment on Azure through multiple orchestration platforms. Build Docker images, push to Azure Container Registry, and deploy using Azure Container Instances, Web App for Containers, and Azure Kubernetes Service.

  **Labs for this session:**
  - [lab_6_a_acr-aci-deployment.md](session06/lab_6_a_acr-aci-deployment.md):  
    *Build Python Flask API Docker image, push to Azure Container Registry (ACR), and deploy to Azure Container Instances (ACI). Complete container workflow from build to deployment.*
  - [lab_6_b_webapp-containers.md](session06/lab_6_b_webapp-containers.md):  
    *Deploy containerized Python Flask API to Azure Web App for Containers. Configure continuous deployment from ACR, automatic health monitoring, and scaling.*
  - [lab_6_c_aci-container-groups.md](session06/lab_6_c_aci-container-groups.md):  
    *Deploy multi-container applications using ACI Container Groups with shared resources. Configure networking, environment variables, and volume mounts.*
  - [lab_6_d_aks-microservices.md](session06/lab_6_d_aks-microservices.md):  
    *Deploy microservices to Azure Kubernetes Service (AKS). Create cluster with Azure CNI networking, deploy pods and services, configure ingress controller, and implement service-to-service communication.*

</details>

<details>
  <summary>Lab Session 07: Serverless Computing with Azure Functions</summary>

  Build serverless applications using Azure Functions with HTTP triggers, Blob Storage triggers, Queue Storage integration, and Event Grid automation. All labs utilize consumption-based pricing with generous free tier allowances.

  **Labs for this session:**
  - [lab_7_a_function-http-trigger.md](session07/lab_7_a_function-http-trigger.md):  
    *Build serverless REST API with Python Azure Function (GET /joke, GET /jokes, POST /joke) exposed through HTTP triggers. Deploy function app, configure routes, enable CORS, and test endpoints.*
  - [lab_7_b_blob-trigger-function.md](session07/lab_7_b_blob-trigger-function.md):  
    *Process files with Blob Storage trigger and Node.js Azure Function. Upload CSV files to Blob Storage, automatically trigger function to parse and store in Cosmos DB. Event-driven data processing.*
  - [lab_7_c_queue-trigger-function.md](session07/lab_7_c_queue-trigger-function.md):  
    *Create event-driven workflow using Azure Queue Storage and Python Functions. Configure poison queue for failed messages, implement batch processing, and test message handling with retries.*
  - [lab_7_d_eventgrid-function-alerts.md](session07/lab_7_d_eventgrid-function-alerts.md):  
    *Monitor Azure resources with Event Grid and Azure Functions. Configure Event Grid subscriptions for VM state changes, trigger functions for notifications, and integrate with Logic Apps for email alerts.*

</details>

<details>
  <summary>Lab Session 08: Monitoring, Logging, and Security Auditing</summary>

  Monitor Azure resources, audit activity logs, analyze network traffic, and track access patterns using native logging and observability services including Azure Monitor, Log Analytics, and Application Insights.

  **Labs for this session:**
  - [lab_8_a_monitor-dashboard-alerts.md](session08/lab_8_a_monitor-dashboard-alerts.md):  
    *Monitor Virtual Machines with Azure Monitor dashboards showing CPU, network, and disk metrics. Create metric alerts with action groups for email notifications and test alert triggers.*
  - [lab_8_b_activity-log-analytics.md](session08/lab_8_b_activity-log-analytics.md):  
    *Enable Azure Activity Log integration with Log Analytics workspace. Query logs with KQL (Kusto Query Language) for resource operations, failed requests, and security events. Create alerts for administrative actions.*
  - [lab_8_c_nsg-flow-logs.md](session08/lab_8_c_nsg-flow-logs.md):  
    *Capture network traffic with NSG Flow Logs to Storage Account and Traffic Analytics. Analyze allowed and denied connections, identify suspicious activity, and visualize network topology.*
  - [lab_8_d_application-insights.md](session08/lab_8_d_application-insights.md):  
    *Implement application monitoring with Application Insights. Track requests, dependencies, exceptions, and custom events. Create availability tests, analyze performance, and configure smart detection alerts.*

</details>

<details>
  <summary>Lab Session 09: Infrastructure as Code – ARM Templates, Bicep, and Terraform</summary>

  Master Infrastructure as Code (IaC) tools to define, deploy, and manage Azure resources using code. Learn ARM template patterns, Azure Bicep declarative syntax, and Terraform for multi-cloud deployments.

  **Labs for this session:**
  - [lab_9_a_arm-template-vnet.md](session09/lab_9_a_arm-template-vnet.md):  
    *Deploy Virtual Network architecture with ARM template. Create VNet with multiple subnets, NSGs, and Virtual Machines. Use parameters and variables for reusability and test template validation.*
  - [lab_9_b_bicep-modular-deployment.md](session09/lab_9_b_bicep-modular-deployment.md):  
    *Build modular infrastructure with Azure Bicep modules. Create separate modules for networking, compute, and storage. Use module outputs and parameters for composition and deploy with Azure CLI.*
  - [lab_9_c_bicep-serverless-api.md](session09/lab_9_c_bicep-serverless-api.md):  
    *Deploy serverless API using Bicep. Define Function App, Storage Account, and Application Insights using declarative syntax. Compare Bicep advantages over ARM templates with cleaner syntax.*
  - [lab_9_d_terraform-azure-infrastructure.md](session09/lab_9_d_terraform-azure-infrastructure.md):  
    *Learn Terraform with Azure provider for multi-cloud IaC. Write HCL configuration for VNet deployment, configure remote state in Azure Storage, use terraform plan/apply/destroy workflow, and manage Azure resources.*

</details>

<details>
  <summary>Lab Session 10: CI/CD Pipelines – Azure DevOps, GitHub Actions, and Deployment Strategies</summary>

  Build complete CI/CD pipelines using Azure DevOps and GitHub Actions. Learn different deployment approaches: container deployments, infrastructure as code automation, and zero-downtime deployment strategies.

  **Labs for this session:**
  - [lab_10_a_azure-devops-webapp.md](session10/lab_10_a_azure-devops-webapp.md):  
    *Create Azure DevOps pipeline for Python Flask app deployment to App Service. Configure build pipeline, artifact publishing, and release pipeline with staging and production environments.*
  - [lab_10_b_github-actions-acr-aci.md](session10/lab_10_b_github-actions-acr-aci.md):  
    *Build CI/CD with GitHub Actions → ACR → ACI. Create workflow to build Docker image, push to Container Registry, and deploy to Container Instances with automatic updates.*
  - [lab_10_c_azure-pipelines-aks.md](session10/lab_10_c_azure-pipelines-aks.md):  
    *Deploy to AKS with Azure Pipelines. Build Docker images, push to ACR, deploy to Kubernetes using kubectl or Helm charts with rolling updates and health checks.*
  - [lab_10_d_github-actions-terraform.md](session10/lab_10_d_github-actions-terraform.md):  
    *Automate Terraform deployments with GitHub Actions. Configure Azure backend for state management, run terraform plan in pull requests, auto-apply on merge. GitOps workflow for infrastructure.*

</details>

<details>
  <summary>Lab Session 11: AI/ML Services – Cognitive Services, Computer Vision, and Machine Learning</summary>

  Leverage Azure AI/ML services for computer vision, natural language processing, speech recognition, and machine learning model deployment. Build intelligent applications using pre-trained models and Azure Machine Learning.

  **Labs for this session:**
  - [lab_11_a_computer-vision-analysis.md](session11/lab_11_a_computer-vision-analysis.md):  
    *Analyze images with Azure Computer Vision API: detect objects, extract text (OCR), recognize landmarks, and analyze image content. Upload images to Blob Storage and call Cognitive Services APIs.*
  - [lab_11_b_text-analytics-sentiment.md](session11/lab_11_b_text-analytics-sentiment.md):  
    *Perform sentiment analysis with Azure Text Analytics (part of Cognitive Services). Analyze customer reviews for sentiment, extract key phrases, identify entities, and detect languages automatically.*
  - [lab_11_c_translator-speech.md](session11/lab_11_c_translator-speech.md):  
    *Build translation pipeline with Azure Translator service and Speech Services. Translate text between 90+ languages, convert speech to text, and text to speech for multilingual applications.*
  - [lab_11_d_azure-ml-model-deployment.md](session11/lab_11_d_azure-ml-model-deployment.md):  
    *Deploy machine learning model with Azure Machine Learning. Train scikit-learn classifier, register model, deploy to managed online endpoint, test predictions via REST API, and monitor inference.*

</details>

<details>
  <summary>Lab Session 12: Hybrid Cloud Networking – VNet Peering, VPN Gateway, and ExpressRoute</summary>

  Connect Virtual Networks, on-premises networks, and multi-region architectures using Azure networking services. Implement VNet peering, site-to-site VPN, point-to-site VPN, and ExpressRoute concepts for hybrid connectivity.

  **Labs for this session:**
  - [lab_12_a_vnet-peering.md](session12/lab_12_a_vnet-peering.md):  
    *Connect two Virtual Networks in same or different regions with VNet Peering. Configure peering connection, update route tables, test connectivity between VMs in peered VNets, and understand global peering.*
  - [lab_12_b_vnet-gateway-site-to-site.md](session12/lab_12_b_vnet-gateway-site-to-site.md):  
    *Create encrypted site-to-site VPN connection between on-premises network (simulated with VNet) and Azure. Configure Virtual Network Gateway, Local Network Gateway, and IPsec tunnels for hybrid connectivity.*
  - [lab_12_c_point-to-site-vpn.md](session12/lab_12_c_point-to-site-vpn.md):  
    *Configure Point-to-Site VPN for secure remote access to Azure VNet. Set up VPN Gateway with certificate authentication, install VPN client on workstation, and test encrypted connectivity to Azure resources.*
  - [lab_12_d_expressroute-concepts.md](session12/lab_12_d_expressroute-concepts.md):  
    *Understand Azure ExpressRoute concepts for dedicated private connectivity. Learn about ExpressRoute circuits, peering types (private, Microsoft, public), connectivity models, and benefits over VPN for enterprise workloads.*

</details>

<details>
  <summary>Lab Session 13: Security and Compliance – Security Center, Sentinel, Key Vault, and Defender</summary>

  Implement comprehensive security monitoring, compliance auditing, threat detection, and secret management. Use Azure native security services for posture management, SIEM, and encryption key management.

  **Labs for this session:**
  - [lab_13_a_security-center-defender.md](session13/lab_13_a_security-center-defender.md):  
    *Enable Microsoft Defender for Cloud (formerly Security Center) for security posture management. Review secure score, implement recommendations, enable Defender plans for VMs, Storage, and SQL, and export compliance reports.*
  - [lab_13_b_sentinel-threat-detection.md](session13/lab_13_b_sentinel-threat-detection.md):  
    *Deploy Azure Sentinel SIEM for intelligent threat detection. Connect data sources, create analytics rules for suspicious activities, investigate incidents with workbooks, and configure automated response playbooks.*
  - [lab_13_c_key-vault-secrets.md](session13/lab_13_c_key-vault-secrets.md):  
    *Manage secrets, keys, and certificates with Azure Key Vault. Create vault with RBAC policies, store secrets, configure access policies, enable soft delete and purge protection, integrate with applications using managed identities.*
  - [lab_13_d_policy-compliance.md](session13/lab_13_d_policy-compliance.md):  
    *Implement governance with Azure Policy and Blueprints. Create custom policies, assign built-in policies for compliance (encryption, tagging), evaluate compliance state, and remediate non-compliant resources.*

</details>

<details>
  <summary>Lab Session 14: Disaster Recovery and Business Continuity – Azure Backup, Site Recovery, and Multi-Region</summary>

  Implement disaster recovery strategies with Azure Backup, Azure Site Recovery, and geo-redundant architectures. Learn backup policies, VM replication, failover orchestration, and multi-region high availability patterns.

  **Labs for this session:**
  - [lab_14_a_azure-backup.md](session14/lab_14_a_azure-backup.md):  
    *Centralize backup management with Azure Backup. Create Recovery Services vault, configure backup policies for VMs and SQL databases, perform on-demand backups, test restore procedures, and implement retention policies.*
  - [lab_14_b_site-recovery-vm.md](session14/lab_14_b_site-recovery-vm.md):  
    *Configure VM replication with Azure Site Recovery for disaster recovery. Set up replication from primary to secondary region, configure recovery plans, test failover without affecting production, and perform failback.*
  - [lab_14_c_geo-redundant-storage.md](session14/lab_14_c_geo-redundant-storage.md):  
    *Implement geo-redundant storage with GRS and RA-GRS for Blob Storage. Configure replication across regions, test read access from secondary region, initiate account failover, and understand RPO/RTO.*
  - [lab_14_d_multi-region-architecture.md](session14/lab_14_d_multi-region-architecture.md):  
    *Build active-active multi-region architecture with Traffic Manager and Azure Front Door. Deploy applications in multiple regions, configure health checks and routing policies, test automatic failover and traffic distribution.*

</details>

<details>
  <summary>Lab Session 15: Migration and Modernization – Azure Migrate, Database Migration, and App Modernization</summary>

  Migrate and modernize applications using Azure migration tools. Assess on-premises infrastructure with Azure Migrate, migrate databases with Database Migration Service, containerize applications, and refactor to serverless.

  **Labs for this session:**
  - [lab_15_a_azure-migrate-assessment.md](session15/lab_15_a_azure-migrate-assessment.md):  
    *Assess on-premises workloads with Azure Migrate. Create migration project, discover servers and databases, analyze readiness and cost estimates, and create migration plans for lift-and-shift scenarios.*
  - [lab_15_b_database-migration-service.md](session15/lab_15_b_database-migration-service.md):  
    *Migrate database with Azure Database Migration Service. Set up source SQL Server and target Azure SQL Database, create migration project, perform schema and data migration with minimal downtime.*
  - [lab_15_c_app-service-migration.md](session15/lab_15_c_app-service-migration.md):  
    *Migrate web applications to Azure App Service using App Service Migration Assistant. Assess compatibility, migrate .NET or Java applications, configure settings, and validate functionality in Azure.*
  - [lab_15_d_modernize-to-containers.md](session15/lab_15_d_modernize-to-containers.md):  
    *Containerize legacy application with Docker and deploy to AKS. Create Dockerfile for existing application, build image, push to ACR, deploy to AKS with proper networking and monitoring. Application modernization.*

</details>

<details>
  <summary>Lab Session 16: Cost Management and Optimization – Cost Analysis, Advisor, Budgets, and Well-Architected</summary>

  Master Azure cost management and optimization tools. Analyze spending patterns, identify cost-saving opportunities, set budget alerts, review best practices with Azure Advisor, and implement Well-Architected Framework principles.

  **Labs for this session:**
  - [lab_16_a_cost-analysis.md](session16/lab_16_a_cost-analysis.md):  
    *Analyze Azure spending with Cost Management + Billing. Create custom views, break down costs by resource group and service, identify top cost contributors, generate forecasts, and export data for reporting.*
  - [lab_16_b_azure-advisor.md](session16/lab_16_b_azure-advisor.md):  
    *Optimize Azure environment with Azure Advisor recommendations. Review cost, security, reliability, performance, and operational excellence recommendations, prioritize actions, and track implementation progress.*
  - [lab_16_c_budgets-alerts.md](session16/lab_16_c_budgets-alerts.md):  
    *Set cost alerts with Azure Budgets. Create monthly cost budgets with spending limits, configure alert thresholds (50%, 80%, 100%), set up action groups for notifications, and track budget consumption.*
  - [lab_16_d_well-architected-review.md](session16/lab_16_d_well-architected-review.md):  
    *Review workloads with Azure Well-Architected Framework. Assess architecture across five pillars (Cost Optimization, Operational Excellence, Performance Efficiency, Reliability, Security), identify improvements, and create action plans.*

</details>

---

#### 🧑‍🏫 Author: Georges Bou Ghantous
<sub><i>This repository delivers practical Azure training through 16 structured lab sessions covering compute, networking, storage, databases, identity, containers, serverless, monitoring, IaC, CI/CD, AI/ML, hybrid networking, security, disaster recovery, migration, and cost optimization.</i></sub>