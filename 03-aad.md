# Prep for Azure Active Directory Integration

In the prior step, you [generated the user-facing TLS certificate](./02-ca-certificates.md); now we'll prepare Azure AD for Kubernetes role-based access control (RBAC). This will ensure you have an Azure AD security group(s) and user(s) assigned for group-based Kubernetes control plane access.

## Expected results

Following the steps below you will result in an Azure AD configuration that will be used for Kubernetes control plane (Cluster API) authorization.

| Object                         | Purpose                                                 |
|--------------------------------|---------------------------------------------------------|
| A Cluster Admin Security Group | Will be mapped to `cluster-admin` Kubernetes role.      |
| A Cluster Admin User           | Represents at least one break-glass cluster admin user. |
| Cluster Admin Group Membership | Association between the Cluster Admin User(s) and the Cluster Admin Security Group. |
| _Additional Security Groups_   | _Optional._ A security group (and its memberships) for the other built-in and custom Kubernetes roles you plan on using. |

## Steps

> :book: The Contoso Bicycle Azure AD team requires all admin access to AKS clusters be security-group based. This applies to the new Secure AKS cluster that is being built for Application ID a0008 under the BU001 business unit. Kubernetes RBAC will be AAD-backed and access granted based on a user's AAD group membership.

1. Query and save your Azure subscription's tenant id. This is the tenant with your Azure subscription to host all resourced to be deployed. If your company has created an Azure AD group for this lab, please run the following query and go to Step 6. You can manually set the variables with the value offered by your company.

   ```bash
   TENANTID_AZURERBAC=$(az account show --query tenantId -o tsv)
   ```

1. Playing the role as the Contoso Bicycle Azure AD team, login into the tenant where Kubernetes Cluster API authorization will be associated with. If you have created a new Azure AD tenant to provide Kubernetes RBAC API, please replace the tenant ID with the informaton you write down when preparing for Prequisites and use the "aksworkshop" user you have configured to log in.
   ```bash
   az login -t <Replace-With-ClusterApi-AzureAD-TenantId> --allow-no-subscriptions
   TENANTID_K8SRBAC=$(az account show --query tenantId -o tsv)
   echo $TENANTID_AZURERBAC
   echo $TENANTID_K8SRBAC
   ```
   If you have created a new Azure AD tenant, TENANTID_AZURERBAC and TENANTID_K8SRBAC are showing different vaules. 
   
   If your company have created an Azure AD group in the existing tenant, TENANTID_AZURERBAC and TENANTID_K8SRBAC are both set to the value of your company Azure AD tenant ID.
   
1. Create/identify the Azure AD security group that is going to map to the [Kubernetes Cluster Admin](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles) role `cluster-admin`.

   If you already have a security group that is appropriate for your cluster's admin service accounts, use that group and skip this step. If using your own group or your Azure AD administrator created one for you to use; you will need to update the group name throughout the reference implementation.

   ```bash
   
   # Create Azure AD Group for Kubernetes RBAC (if you have created an Azure AD group, pls skip this)
   export AADOBJECTNAME_GROUP_CLUSTERADMIN=cluster-admins-bu0001a000800
   export AADOBJECTID_GROUP_CLUSTERADMIN=$(az ad group create --display-name $AADOBJECTNAME_GROUP_CLUSTERADMIN --mail-nickname $AADOBJECTNAME_GROUP_CLUSTERADMIN --description "Principals in this group are cluster admins in the bu0001a000800 cluster." --query objectId -o tsv)
   echo $AADOBJECTNAME_GROUP_CLUSTERADMIN
   echo $AADOBJECTID_GROUP_CLUSTERADMIN
   
   # Query Azure AD Group for Kubernetes RBAC
   export AADOBJECTNAME_GROUP_CLUSTERADMIN=cluster-admins-bu0001a000800
   export AADOBJECTID_GROUP_CLUSTERADMIN=$(az ad group show --group $AADOBJECTNAME_GROUP_CLUSTERADMIN --query objectId --out tsv)
   echo $AADOBJECTNAME_GROUP_CLUSTERADMIN
   echo $AADOBJECTID_GROUP_CLUSTERADMIN
   ```

1. Create a "break-glass" cluster administrator user for your AKS cluster.

   > :book: The organization knows the value of having a break-glass admin user for their critical infrastructure. The app team requests a cluster admin user and Azure AD Admin team proceeds with the creation of the user in Azure AD.

   ```bash
   export TENANTDOMAIN_K8SRBAC=$(az ad signed-in-user show --query 'userPrincipalName' -o tsv | cut -d '@' -f 2 | sed 's/\"//')
   echo $TENANTDOMAIN_K8SRBAC
   
   # Create Azure AD User for Kubernetes RBAC (if you have created an Azure AD User, pls skip this)
   export AADOBJECTNAME_USER_CLUSTERADMIN=bu0001a000800-admin
   export AADOBJECTID_USER_CLUSTERADMIN=$(az ad user create --display-name=${AADOBJECTNAME_USER_CLUSTERADMIN} --user-principal-name ${AADOBJECTNAME_USER_CLUSTERADMIN}@${TENANTDOMAIN_K8SRBAC} --force-change-password-next-login --password ChangeMebu0001a0008AdminChangeMe --query objectId -o tsv)
   echo $AADOBJECTNAME_USER_CLUSTERADMIN
   echo $AADOBJECTID_USER_CLUSTERADMIN
   
    # Query Azure AD User for Kubernetes RBAC
   export AADOBJECTNAME_USER_CLUSTERADMIN=bu0001a000800-admin
   export AADOBJECTID_USER_CLUSTERADMIN=$(az ad user show --id ${AADOBJECTNAME_USER_CLUSTERADMIN}@${TENANTDOMAIN_K8SRBAC} --query objectId -o tsv)
   echo $AADOBJECTNAME_USER_CLUSTERADMIN
   echo $AADOBJECTID_USER_CLUSTERADMIN
   
   ```
   


1. Add the cluster admin user(s) to the cluster admin security group.

   > :book: The recently created break-glass admin user is added to the Kubernetes Cluster Admin group from Azure AD. After this step the Azure AD Admin team will have finished the app team's request.

   ```bash
   az ad group member add -g $AADOBJECTID_GROUP_CLUSTERADMIN --member-id $AADOBJECTID_USER_CLUSTERADMIN
   ```

   This object ID will be used later while creating the cluster. This way, once the cluster gets deployed the new group will get the proper Cluster Role bindings in Kubernetes.
   


1. Now please update the values for those variables in the [`variables.txt` file](./variables.txt):
   Since you have git cloned the repo to your cloud shell, you will update this file there. You can input "Code variables.txt" to edit the file.After you    get all variables set correctly, pls copy all commandlines from the variables.txt and run in the cloud shell:
   
   ```bash
   echo "Exporting environment variables"
   export AADOBJECTNAME_GROUP_CLUSTERADMIN='please replace with your values'
   export AADOBJECTID_GROUP_CLUSTERADMIN='please replace with your values'
   export TENANTID_K8SRBAC='please replace with your values'
   export TENANTID_AZURERBAC='please replace with your values'
   export TENANTDOMAIN_K8SRBAC='please replace with your values'
   export AADOBJECTNAME_USER_CLUSTERADMIN='please replace with your values'
   export AADOBJECTID_USER_CLUSTERADMIN='please replace with your values'
   

   echo $AADOBJECTNAME_GROUP_CLUSTERADMIN
   echo $AADOBJECTID_GROUP_CLUSTERADMIN
   echo $TENANTID_K8SRBAC
   echo $TENANTID_AZURERBAC
   echo $TENANTDOMAIN_K8SRBAC
   echo $AADOBJECTNAME_USER_CLUSTERADMIN
   echo $AADOBJECTID_USER_CLUSTERADMI
   ```
1. Set up groups to map into other Kubernetes Roles. _Optional, fork required._

   > :book: The team knows there will be more than just cluster admins that need group-managed access to the cluster. Out of the box, Kubernetes has other roles like _admin_, _edit_, and _view_ which can also be mapped to Azure AD Groups for use both at namespace and at the cluster level.

   In the [`cluster-rbac.yaml` file](./cluster-manifests/cluster-rbac.yaml) and the various namespaced [`rbac.yaml files`](./cluster-manifests/cluster-baseline-settings/rbac.yaml), you can uncomment what you wish and replace the `<replace-with-an-aad-group-object-id...>` placeholders with corresponding new or existing AD groups that map to their purpose for this cluster or namespace. You do not need to perform this action for this walk through; they are only here for your reference.

   :bulb: Alternatively/Additionally, you can make some of these group associations to [Azure RBAC roles](https://docs.microsoft.com/azure/aks/manage-azure-rbac). At the time of this writing, this feature is still in _preview_. This reference implementation has not been validated with that feature.

### Next step

:arrow_forward: [Deploy the hub-spoke network topology](./04-networking.md)
