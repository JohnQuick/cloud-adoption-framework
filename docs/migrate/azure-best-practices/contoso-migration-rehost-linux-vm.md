---
title: Rehost an on-premises Linux application to Azure VMs
description: Learn how Contoso rehosts an on-premises Linux app by migrating to Azure VMs.
author: deltadan
ms.author: abuck
ms.date: 07/01/2020
ms.topic: conceptual
ms.service: cloud-adoption-framework
ms.subservice: migrate
services: azure-migrate
---

<!-- cSpell:ignore givenscj OSTICKETWEB OSTICKETMYSQL OSTICKETWEB OSTICKETMYSQL contosohost vcenter contosodc osTicket binlog systemctl NSGs distros -->

# Rehost an on-premises Linux application to Azure VMs

This article shows how the fictional company Contoso rehosts a two-tier [LAMP-based](https://wikipedia.org/wiki/LAMP_(software_bundle)) application, using Azure IaaS VMs.

osTicket, the service desk application used in this example is provided as open source. If you'd like to use it for your own testing purposes, you can download it from [GitHub](https://github.com/osTicket/osTicket).

## Business drivers

The IT leadership team has worked closely with business partners to understand what they want to achieve with this migration:

- **Address business growth.** Contoso is growing, and as a result there's pressure on the on-premises systems and infrastructure.
- **Limit risk.** The service desk application is critical for the Contoso business. Contoso wants to move it to Azure with zero risk.
- **Extend.** Contoso don't want to change the application right now. It simply wants to ensure that the application is stable.

## Migration goals

The Contoso cloud team has pinned down goals for this migration, to determine the best migration method:

- After migration, the application in Azure should have the same performance capabilities as it does today in their on-premises VMware environment. The application will remain as critical in the cloud as it is on-premises.
- Contoso doesn't want to invest in this application. It is important to the business, but in its current form Contoso simply wants to move it safely to the cloud.
- Contoso doesn't want to change the ops model for this application. It wants to interact with the application in the cloud in the same way that they do now.
- Contoso doesn't want to change application functionality. Only the application location will change.
- Having completed a couple of Windows application migrations, Contoso wants to learn how to use a Linux-based infrastructure in Azure.

## Solution design

After pinning down goals and requirements, Contoso designs and review a deployment solution, and identifies the migration process, including the Azure services that Contoso will use for the migration.

### Current application

- The OSTicket application is tiered across two VMs (`OSTICKETWEB` and `OSTICKETMYSQL`).
- The VMs are located on VMware ESXi host `contosohost1.contoso.com` (version 6.5).
- The VMware environment is managed by vCenter Server 6.5 (`vcenter.contoso.com`), running on a VM.
- Contoso has an on-premises datacenter (`contoso-datacenter`) with an on-premises domain controller (`contosodc1`).

### Proposed architecture

- Since the application is a production workload, the VMs in Azure will reside in the production resource group `ContosoRG`.
- The VMs will be migrated to the primary region (East US 2) and placed in the production network (`VNET-PROD-EUS2`):
  - The web VM will reside in the front-end subnet (`PROD-FE-EUS2`).
  - The database VM will reside in the database subnet (`PROD-DB-EUS2`).
- The on-premises VMs in the Contoso datacenter will be decommissioned after the migration is done.

![Scenario architecture](./media/contoso-migration-rehost-linux-vm/architecture.png)

### Solution review

Contoso evaluates the proposed design by putting together a pros and cons list.

| Consideration | Details |
| --- | --- |
| **Pros** | Both the application VMs will be moved to Azure without changes, making the migration simple. <br><br> Since Contoso is using a lift and shift approach for both application VMs, no special configuration or migration tools are needed for the application database. <br><br> Contoso will retain full control of the application VMs in Azure. <br><br> The application VMs are running Ubuntu 16.04-TLS, an endorsed Linux distribution. [Learn more](https://docs.microsoft.com/azure/virtual-machines/linux/endorsed-distros). |
| **Cons** | The web and data tier of the application remain single points of failover. <br><br> Contoso will need to continue supporting the application as Azure VMs rather than moving to a managed service such as Azure App Service and Azure Database for MySQL. <br><br> Contoso realizes that, by keeping things simple with a lift and shift VM migration, they're not taking full advantage of the features provided by [Azure Database for MySQL](https://docs.microsoft.com/azure/mysql/overview) (built-in high availability, predictable performance, simple scaling, automatic backups, and built-in security). |

### Migration process

Contoso will migrate as follows:

- As a first step, Contoso prepares and sets up Azure components for Azure Migrate: Server Migration, and prepares the on-premises VMware infrastructure.
- They already have the [Azure infrastructure](./contoso-migration-infrastructure.md) in place, so Contoso just needs to configure the replication of the VMs through the Azure Migrate: Server Migration tool.
- With everything prepared, Contoso can start replicating the VMs.
- After replication is enabled and working, Contoso will migrate the VM by failing it over to Azure.

![Migration process](./media/contoso-migration-rehost-linux-vm/migration-process-az-migrate.png)

### Azure services

| Service | Description | Cost |
| --- | --- | --- |
| [Azure Migrate: Server Migration](https://docs.microsoft.com/azure/migrate/contoso-migration-rehost-linux-vm) | The service orchestrates and manages migration of your on-premises applications and workloads, and AWS/GCP VM instances. | During replication to Azure, Azure Storage charges are incurred. Azure VMs are created, and incur charges, when migration occurs. [Learn more](https://azure.microsoft.com/pricing/details/azure-migrate) about charges and pricing. |

## Prerequisites

Here's what Contoso needs for this scenario.

Requirements | Details |
| --- | --- |
| **Azure subscription** | Contoso created subscriptions in an early article in this series. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free). <br><br> If you create a free account, you're the administrator of your subscription and can perform all actions. <br><br> If you use an existing subscription and you're not the administrator, you need to work with the admin to assign you Owner or Contributor permissions. <br><br> If you need more granular permissions, review [this article](https://docs.microsoft.com/azure/site-recovery/site-recovery-role-based-linked-access-control). |
| **Azure infrastructure** | [Learn how](./contoso-migration-infrastructure.md) Contoso set up an Azure infrastructure. <br><br> Learn more about specific [prerequisites](./contoso-migration-devtest-to-iaas.md#prerequisites) for Azure Migrate: Server Migration. |
| **On-premises servers** | The on-premises vCenter Server should be running version 5.5, 6.0, or 6.5. <br><br> An ESXi host running version 5.5, 6.0 or 6.5. <br><br> One or more VMware VMs running on the ESXi host. |
| **On-premises VMs** | [Review Linux Distros](https://docs.microsoft.com/azure/virtual-machines/linux/endorsed-distros) that are endorsed to run on Azure. |

## Scenario steps

Here's how Contoso will complete the migration:

> [!div class="checklist"]
>
> - **Step 1: Prepare Azure for Azure Migrate: Server Migration.** They add the Azure Migrate: Server Migration tool to their Azure Migrate project.
> - **Step 2: Prepare on-premises VMware for Azure Migrate: Server Migration.** They prepare accounts for VM discovery, and prepare to connect to Azure VMs after migration.
> - **Step 3: Replicate VMs.** They set up replication, and start replicating VMs to Azure Storage.
> - **Step 4: Migrate the VMs with Azure Migrate: Server Migration.** They run a test migration to make sure everything's working, and then run a to migration to move the VMs to Azure.

## Step 1: Prepare Azure for the Azure Migrate: Server Migration tool

Here are the Azure components Contoso needs to migrate the VMs to Azure:

- A VNet in which Azure VMs will be located when they're created during migration.
- The Azure Migrate: Server Migration tool provisioned.

They set these components up as follows:

1. **Set up a network:** Contoso already set up a network that can be for Azure Migrate: Server Migration when they [deployed the Azure infrastructure](./contoso-migration-infrastructure.md)

    - The SmartHotel360 application is a production application, and the VMs will be migrated to the Azure production network (`VNET-PROD-EUS2`) in the primary region (`East US 2`).
    - Both VMs will be placed in the `ContosoRG` resource group, which is used for production resources.
    - The application front-end VM (`OSTICKETWEB`) will migrate to the front-end subnet (`PROD-FE-EUS2`), in the production network.
    - The application database VM (`OSTICKETMYSQL`) will migrate to the database subnet (`PROD-DB-EUS2`), in the production network.

2. **Provision the Azure Migrate: Server Migration tool:** With the network and storage account in place, Contoso now creates a Recovery Services vault (`ContosoMigrationVault`), and places it in the `ContosoFailoverRG` resource group in their primary region (`East US 2`).

    ![Azure Migrate Server Migration tool](./media/contoso-migration-rehost-linux-vm/server-migration-tool.png)

**Need more help?**

[Learn about](https://docs.microsoft.com/azure/migrate) setting up Azure Migrate: Server Migration tool.

## Step 2: Prepare on-premises VMware for Azure Migrate: Server Migration

After migration to Azure, Contoso wants to be able to connect to the replicated VMs in Azure. To do this, there's a couple of things that the Contoso admins need to do:

- To access Azure VMs over the internet, they enable SSH on the on-premises Linux VM before migration. For Ubuntu this can be completed using the following command: `sudo apt-get ssh install -y`.
- Install the [Azure Linux agent](https://docs.microsoft.com/azure/virtual-machines/extensions/agent-linux)
- After they run the migration, they can check **Boot diagnostics** to view a screenshot of the VM.
- If this doesn't work, they'll need to check that the VM is running, and review these [troubleshooting tips](https://social.technet.microsoft.com/wiki/contents/articles/31666.troubleshooting-remote-desktop-connection-after-failover-using-asr.aspx).

**Need more help?**

- Learn about [preparing VMs for migration](https://docs.microsoft.com/azure/migrate/prepare-for-migration).

## Step 3: Replicate the on-premises VMs

Before Contoso admins can run a migration to Azure, they need to set up and enable replication.

With discovery completed, you can begin replication of VMware VMs to Azure.

1. In the Azure Migrate project > **Servers**, **Azure Migrate: Server Migration**, select **Replicate**.

    ![Replicate VMs](./media/contoso-migration-rehost-linux-vm/select-replicate.png)

2. In **Replicate** > **Source settings** > **Are your machines virtualized?**, select **Yes, with VMware vSphere**.

3. In **On-premises appliance**, select the name of the Azure Migrate appliance that you set up > **OK**.

    ![Source settings](./media/contoso-migration-rehost-linux-vm/source-settings.png)

4. In **Virtual machines**, select the machines you want to replicate.
    - If you've run an assessment for the VMs, you can apply VM sizing and disk type (premium/standard) recommendations from the assessment results. To do this, in **Import migration settings from an Azure Migrate assessment?**, select the **Yes** option.
    - If you didn't run an assessment, or you don't want to use the assessment settings, select the **No** option.
    - If you selected to use the assessment, select the VM group, and assessment name.

    ![Select assessment](./media/contoso-migration-rehost-linux-vm/select-assessment.png)

5. In **Virtual machines**, search for VMs as needed, and check each VM you want to migrate. Then select **Next: Target settings**.

6. In **Target settings**, select the subscription, and target region to which you'll migrate, and specify the resource group in which the Azure VMs will reside after migration. In **Virtual Network**, select the Azure VNet/subnet to which the Azure VMs will be joined after migration.

7. In **Azure Hybrid Benefit**, select the following:

    - Select **No** if you don't want to apply Azure Hybrid Benefit. Then select **Next**.
    - Select **Yes** if you have Windows Server machines that are covered with active Software Assurance or Windows Server subscriptions, and you want to apply the benefit to the machines you're migrating. Then select **Next**.

8. In **Compute**, review the VM name, size, OS disk type, and availability set. VMs must conform with [Azure requirements](https://docs.microsoft.com/azure/migrate/migrate-support-matrix-vmware#vmware-requirements).

    - **VM size:** If you're using assessment recommendations, the VM size dropdown will contain the recommended size. Otherwise Azure Migrate picks a size based on the closest match in the Azure subscription. Alternatively, pick a manual size in **Azure VM size.**
    - **OS disk:** Specify the OS (boot) disk for the VM. The OS disk is the disk that has the operating system bootloader and installer.
    - **Availability set:** If the VM should be in an Azure availability set after migration, specify the set. The set must be in the target resource group you specify for the migration.

9. In **Disks**, specify whether the VM disks should be replicated to Azure, and select the disk type (standard SSD/HDD or premium-managed disks) in Azure. Then select **Next**.
    - You can exclude disks from replication.
    - If you exclude disks, won't be present on the Azure VM after migration.

10. In **Review and start replication**, review the settings, then select **Replicate** to start the initial replication for the servers.

> [!NOTE]
> You can update replication settings any time before replication starts, in **Manage** > **Replicating machines**. Settings can't be changed after replication starts.

## Step 4: Migrate the VMs

Contoso admins run a quick test migration, and then a migration to move the VMs.

### Run a test migration

1. In **Migration goals** > **Servers** > **Azure Migrate: Server Migration**, select **Test migrated servers**.

     ![Test migrated servers](./media/contoso-migration-rehost-linux-vm/test-migrated-servers.png)

2. Select and hold (or right-click) the VM to test, then select **Test migrate**.

    ![Test migration](./media/contoso-migration-rehost-linux-vm/test-migrate.png)

3. In **Test Migration**, select the Azure VNet in which the Azure VM will be located after the migration. We recommend you use a nonproduction VNet.
4. The **Test migration** job starts. Monitor the job in the portal notifications.
5. After the migration finishes, view the migrated Azure VM in **Virtual Machines** in the Azure portal. The machine name has a suffix **-Test**.
6. After the test is done, select and hold (or right-click) the Azure VM in **Replicating machines**, then select **Clean up test migration**.

    ![Clean up migration](./media/contoso-migration-rehost-linux-vm/clean-up.png)

### Migrate the VMs

Now Contoso admins run a full migration to complete the move.

1. In the Azure Migrate project > **Servers** > **Azure Migrate: Server Migration**, select **Replicating servers**.

    ![Replicating servers](./media/contoso-migration-rehost-linux-vm/replicating-servers.png)

2. In **Replicating machines**, select and hold (or right-click) the VM > **Migrate**.
3. In **Migrate** > **Shut down virtual machines and perform a planned migration with no data loss**, select **Yes** > **OK**.
    - By default Azure Migrate shuts down the on-premises VM, and runs an on-demand replication to synchronize any VM changes that occurred since the last replication occurred. This ensures no data loss.
    - If you don't want to shut down the VM, select **No**.
4. A migration job starts for the VM. Track the job in Azure notifications.
5. After the job finishes, you can view and manage the VM from the **Virtual Machines** page.

### Connect the VM to the database

As the final step in the migration process, Contoso admins update the connection string of the application to point to the application database running on the `OSTICKETMYSQL` VM.

1. They make an SSH connection to the `OSTICKETWEB` VM using PuTTY or another SSH client. The VM is private so they connect using the private IP address.

    ![Connect to database](./media/contoso-migration-rehost-linux-vm/db-connect.png)

    ![Connect to database](./media/contoso-migration-rehost-linux-vm/db-connect2.png)

2. They need to make sure that the `OSTICKETWEB` VM can communicate with the `OSTICKETMYSQL` VM. Currently the configuration is hardcoded with the on-premises IP address `172.16.0.43`.

    **Before the update:**

    ![Update IP](./media/contoso-migration-rehost-linux-vm/update-ip1.png)

    **After the update:**

    ![Update IP](./media/contoso-migration-rehost-linux-vm/update-ip2.png)

3. They restart the service with **systemctl restart apache2**.

    ![Restart](./media/contoso-migration-rehost-linux-vm/restart.png)

4. Finally, they update the DNS records for `OSTICKETWEB` and `OSTICKETMYSQL`, on one of the Contoso domain controllers.

    ![Update DNS](./media/contoso-migration-rehost-linux-vm-mysql/update-dns.png)

    ![Update DNS](./media/contoso-migration-rehost-linux-vm-mysql/update-dns.png)

**Need more help?**

- [Learn about](https://docs.microsoft.com/azure/migrate/tutorial-migrate-vmware#run-a-test-migration) running a test migration.
- [Learn about](https://docs.microsoft.com/azure/migrate/tutorial-migrate-vmware#migrate-vms) migrating VMs to Azure.

## Clean up after migration

With migration complete, the osTicket application tiers are now running on Azure VMs.

Now, Contoso needs to clean up as follows:

- Remove the on-premises VMs from the vCenter inventory.
- Remove the on-premises VMs from local backup jobs.
- Update their internal documentation to show the new location, and IP addresses for `OSTICKETWEB` and `OSTICKETMYSQL`.
- Review any resources that interact with the VMs, and update any relevant settings or documentation to reflect the new configuration.
- Contoso used the Azure Migrate service with management VM to assess the VMs for migration. Admins should remove the migration VM and web VMs from VMware ESXi server.

## Review the deployment

With the application now running, Contoso needs to fully operationalize and secure their new infrastructure.

### Security

The Contoso security team reviews the OSTICKETWEB and OSTICKETMYSQL VMs to determine any security issues.

- The team reviews the network security groups (NSGs) for the VMs to control access. NSGs are used to ensure that only traffic allowed to the application can pass.
- The team also considers securing the data on the VM disks using Azure Disk Encryption and Azure Key Vault.

For more information, see [Security best practices for IaaS workloads in Azure](https://docs.microsoft.com/azure/security/fundamentals/iaas).

<!-- docsTest:ignore "Quickstart: Set" -->

### BCDR

For business continuity and disaster recovery, Contoso takes the following actions:

- **Keep data safe.** Contoso backs up the data on the VMs using [Azure VM backup](https://docs.microsoft.com/azure/backup/backup-azure-vms-introduction).
- **Keep applications up and running.** Contoso replicates the application VMs in Azure to a secondary region using Site Recovery. For more information, see [Quickstart: Set up disaster recovery to a secondary Azure region for an Azure VM](https://docs.microsoft.com/azure/site-recovery/azure-to-azure-quickstart).

### Licensing and cost optimization

- After deploying resources, Contoso assigns Azure tags as defined during the [Azure infrastructure deployment](./contoso-migration-infrastructure.md#set-up-tagging).
- Contoso has no licensing issues with the Ubuntu servers.
- Contoso will use [Azure Cost Management and Billing](https://docs.microsoft.com/azure/cost-management-billing/cost-management-billing-overview) to ensure they stay within budgets established by their IT leadership.
