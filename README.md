
# **Automated Cloud Security Audit Using a Custom Ansible Module**

## **Use Case: Ensuring Cloud Security Compliance for Virtual Machines**

Security teams often need **periodic audits** to verify if all virtual machines (VMs) meet organizational security standards, such as:
- **Firewall rules**
- **Allowed ports**
- **Encryption settings**
- **Machine tags for tracking**
- **OS hardening policies**

A traditional approach requires **manual verification** or a mix of scripts, leading to inconsistent audits and **potential security risks**.

### **Why Use an Ansible Module Instead of a Script?**
- **Automation & Reusability:** The module integrates **directly into Ansible playbooks**, allowing repeatable security enforcement.
- **Idempotency:** Ensures compliance checks run **only when necessary** without redundant actions.
- **Structured Output:** Results are returned **in Ansible JSON format**, making reporting easier.
- **Check Mode Support:** Runs audits in **dry-run mode** before enforcing compliance.

---

## **Step 1: Custom Ansible Module (Security Compliance Checker)**

Create a file named **`vm_security_audit.py`** inside an Ansible **library directory**:

```python
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient

def check_vm_compliance(vm):
    """ Validate VM security settings """
    issues = []
    
    # Example: Check if VM has a firewall rule allowing all traffic (which is unsafe)
    if vm.network_profile is None or "allowAll" in vm.tags.get("firewall_rules", ""):
        issues.append("VM allows unrestricted traffic.")

    # Example: Ensure OS disk is encrypted
    if not vm.storage_profile.os_disk.encryption_settings:
        issues.append("OS disk is not encrypted.")

    # Example: Ensure VM is tagged properly
    if "owner" not in vm.tags:
        issues.append("Missing 'owner' tag for tracking responsibility.")

    return issues

def run_module():
    module_args = dict(
        resource_group=dict(type='str', required=True),
        subscription_id=dict(type='str', required=True)
    )

    module = AnsibleModule(argument_spec=module_args, supports_check_mode=True)

    resource_group = module.params["resource_group"]
    subscription_id = module.params["subscription_id"]

    credential = DefaultAzureCredential()
    compute_client = ComputeManagementClient(credential, subscription_id)
    
    vms = compute_client.virtual_machines.list(resource_group)
    non_compliant_vms = {}

    for vm in vms:
        issues = check_vm_compliance(vm)
        if issues:
            non_compliant_vms[vm.name] = issues

    result = dict(
        changed=False,
        non_compliant_vms=non_compliant_vms
    )

    module.exit_json(**result)

if __name__ == "__main__":
    run_module()
```

### **Key Features:**
✅ **Retrieves all VMs from Azure using Managed Identity (OIDC)**  
✅ **Checks firewall rules, disk encryption, and mandatory VM tagging**  
✅ **Returns structured compliance data**  
✅ **Runs in audit mode (check mode) without making changes**  

---

## **Step 2: Using the Custom Module in an Ansible Playbook**

Create an **Ansible playbook (`audit_vm_security.yaml`)** to invoke this module:

```yaml
- name: Cloud Security Audit
  hosts: localhost
  tasks:
    - name: Check VM Security Compliance
      vm_security_audit:
        resource_group: "Production-VMs"
        subscription_id: "12345678-aaaa-bbbb-cccc-ddddeeefff00"
      register: audit_results

    - name: Display Non-Compliant VMs
      debug:
        msg: "{{ audit_results.non_compliant_vms }}"
```

Run the playbook:
```bash
ANSIBLE_LIBRARY=./library ansible-playbook audit_vm_security.yaml
```

Expected output:

```bash
TASK [Display Non-Compliant VMs] ******************************************
ok: [localhost] => {
    "msg": {
        "vm-1": ["VM allows unrestricted traffic.", "OS disk is not encrypted."],
        "vm-3": ["Missing 'owner' tag for tracking responsibility."]
    }
}
```

---

## **Step 3: Enforcing Compliance**
To **enforce security fixes**, modify the module to:
- Apply **secure firewall rules**.
- Enable **disk encryption**.
- Add **mandatory tags**.

Then, update the playbook to **execute fixes instead of reporting**.

---

## **Step 4: Running Periodic Audits in AKS**
To **automate audits**, deploy this playbook in **Azure Kubernetes Service (AKS)** as a **CronJob**.

### **Kubernetes CronJob (`aks_security_audit_cronjob.yaml`)**
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: vm-security-audit
spec:
  schedule: "0 2 * * *"  # Runs daily at 2 AM UTC
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: vm-security-audit
        spec:
          containers:
          - name: vm-security-audit
            image: myregistry.azurecr.io/vm-security-audit:latest
            env:
            - name: RESOURCE_GROUP
              value: "Production-VMs"
            - name: SUBSCRIPTION_ID
              value: "12345678-aaaa-bbbb-cccc-ddddeeefff00"
          restartPolicy: OnFailure
```

Deploy to AKS:
```bash
kubectl apply -f aks_security_audit_cronjob.yaml
```

---

## **Step 5: Logging Audit Results to Azure Application Insights**

To enable **centralized monitoring**, modify the module to **send logs** to **Azure Application Insights**:

```python
from opencensus.ext.azure.log_exporter import AzureLogHandler

logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(connection_string=os.environ.get("APPINSIGHTS_CONNECTION_STRING")))
logger.setLevel(logging.INFO)

logger.info(f"Compliance audit completed. Results: {non_compliant_vms}")
```

---

## **Conclusion**

This **custom Ansible module** automates **cloud security audits**, ensuring that **virtual machines comply with security standards**. The solution **eliminates manual checking**, integrates with **Azure**, enables **continuous security enforcement**, and **executes scheduled audits in AKS**.

### **Key Benefits**
✅ **Ensures all VMs adhere to security policies**  
✅ **Uses Managed Identity (OIDC) for secure API authentication**  
✅ **Automates audits with Ansible playbooks & AKS CronJobs**  
✅ **Logs results to Azure Application Insights for monitoring**  

### **Further Reading**
- [Ansible Module Development](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules.html)
- [Azure Compute Python SDK](https://docs.microsoft.com/en-us/azure/developer/python/sdk/)
- [Managed Identity & OIDC Authentication](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)
- [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Azure Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)

---
