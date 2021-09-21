https://docs.microsoft.com/en-us/learn/paths/az-900-describe-cloud-concepts/


Examen -> https://docs.microsoft.com/en-us/learn/certifications/exams/az-900

https://github.com/AzureMentor/Azure-AZ-900-Study-Guide

https://github.com/osmanys/az900-training

https://github.com/davidcervigonluna/AZ-900FAQ

https://ptgmedia.pearsoncmg.com/images/9780135732182/downloads/9780135732182_URLs.pdf

Exam topics


AZ-900 Domain                                                         Area Weight

Describe cloud concepts                                               20-25%

Describe core Azure services                                          15-20%

Describe core solutions and management tools on Azure                 10-15%

Describe general security and network security features               10-15%

Describe identity, governance, privacy, and compliance features       20-25%

Describe Azure cost management and Service Level Agreements           10-15%



https://docs.microsoft.com/en-us/learn/modules/intro-to-azure-fundamentals/what-is-cloud-computing
## What is cloud computing

Cloud computing is the delivery of computing services over the internet by using a pay-as-you-go pricing model. You typically pay only for the cloud services you use, which helps you:
- Lower your operating costs.
- Run your infrastructure more efficiently.
- Scale as your business needs change.
To put it another way, cloud computing is a way to rent compute power and storage from someone else's datacenter.

## What is Azure
Azure is a continually expanding set of cloud services that help your organization meet your current and future business challenges. Azure gives you the freedom to build, manage, and deploy applications on a massive global network using your favorite tools and frameworks.

## What is Azure Portal
he Azure portal is a web-based, unified console that provides an alternative to command-line tools. With the Azure portal, you can manage your Azure subscription by using a graphical user interface. You can:

- Build, manage, and monitor everything from simple web apps to complex cloud deployments.
- Create custom dashboards for an organized view of resources.
- Configure accessibility options for an optimal experience.

## Azure tour
- Compute
- Networking: connect cloud to your infra
- Storage: disk, file, blob, achive
- Mobile: build and deploy native apps, notifications
- Databases: proprietary and open-source engines
- Web: Build, deploy, manage and scale web applications
- IoT: Connect, monitor IoT assets..
- BigData: large volumes of data
- AI: Foresee future behavior
- DevOps: Automating software delivery


## Organizing structure for resources in Azure

Has four levels: management groups, subscriptions, resource groups, and resources.

- *Resources*: Resources are instances of services that you create, like virtual machines, storage, or SQL databases.
- *Resource groups*: Resources are combined into resource groups, which act as a logical container into which Azure resources like web apps, databases, and storage accounts are deployed and managed.
- *Subscriptions*: A subscription groups together user accounts and the resources that have been created by those user accounts. For each subscription, there are limits or quotas on the amount of resources that you can create and use. Organizations can use subscriptions to manage costs and the resources that are created by users, teams, or projects.
- *Management groups*: These groups help you manage access, policy, and compliance for multiple subscriptions. All subscriptions in a management group automatically inherit the conditions applied to the management group.

## Azure regions and availability zones

### Azure regions

Resources are created in regions, which are different geographical locations around the globe that contain Azure datacenters.

A region is a geographical area on the planet that contains at least one but potentially multiple datacenters that are nearby and networked together with a low-latency network. Azure intelligently assigns and controls the resources within each region to ensure workloads are appropriately balanced.

Azure has specialized regions that you might want to use when you build out your applications for compliance or legal purposes. A few examples include:

- US DoD Central, US Gov Virginia, US Gov Iowa and more: These regions are physical and logical network-isolated instances of Azure for U.S. government agencies and partners. These datacenters are operated by screened U.S. personnel and include additional compliance certifications.
    China East, China North, and more: These regions are available through a unique partnership between Microsoft and 21Vianet, whereby Microsoft doesn't directly maintain the datacenters.
- Regions are what you use to identify the location for your resources. There are two other terms you should also be aware of: geographies and availability zones.

### Azure availability zones
Availability zones are physically separate datacenters within an Azure region. Each availability zone is made up of one or more datacenters equipped with independent power, cooling, and networking. An availability zone is set up to be an isolation boundary. If one zone goes down, the other continues working. Availability zones are connected through high-speed, private fiber-optic networks.
Not every region has support for availability zones. 

### Azure region pairs
Each Azure region is always paired with another region within the same geography (such as US, Europe, or Asia) at least 300 miles away. This approach allows for the replication of resources (such as VM storage) across a geography that helps reduce the likelihood of interruptions because of events such as natural disasters, civil unrest, power outages, or physical network outages that affect both regions at once. 

## Azure resource and resource groups

- *Resource*: A manageable item that's available through Azure. Virtual machines (VMs), storage accounts, web apps, databases, and virtual networks are examples of resources.
- *Resource group*: A container that holds related resources for an Azure solution. The resource group includes resources that you want to manage as a group. You decide which resources belong in a resource group based on what makes the most sense for your organization.

All resources must be in a resource group, and a resource can only be a member of a single resource group. Many resources can be moved between resource groups with some services having specific limitations or requirements to move. Resource groups can't be nested. Before any resource can be provisioned, you need a resource group for it to be placed in.

If you delete a resource group, all resources contained within it are also deleted. 
Resource groups are also a scope for applying role-based access control (RBAC) permissions. By applying RBAC permissions to a resource group, you can ease administration and limit access to allow only what's needed.

Azure Resource Manager is the deployment and management service for Azure. It provides a management layer that enables you to create, update, and delete resources in your Azure account. You use management features like access control, locks, and tags to secure and organize your resources after deployment.

# Azure management groups

Azure management groups provide a level of scope above subscriptions. You organize subscriptions into containers called management groups and apply your governance conditions to the management groups. All subscriptions within a management group automatically inherit the conditions applied to the management group. 

- 10,000 management groups can be supported in a single directory.
- A management group tree can support up to six levels of depth. This limit doesn't include the root level or the subscription level.
- Each management group and subscription can support only one parent.
- Each management group can have many children.
- NUEVO_CREACION_VDCORGAll subscriptions and management groups are within a single hierarchy in each directory.



