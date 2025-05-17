
# Custom Ansible Module for Virtual Machine Management

This project demonstrates how to write a custom Ansible module in Python to create and manage virtual machines on a cloud provider. By using a custom module, you gain tighter integration with Ansible’s automation, idempotency, check mode, and structured JSON output—which are not as easily achieved with standalone scripts.

## Why Use an Ansible Module Instead of a Script?

- **Integration with Ansible:**  
  Modules run as part of Ansible plays. They can be easily called from playbooks and integrate with Ansible’s inventory, error handling, and logging infrastructure.
  
- **Idempotency:**  
  Well-written modules implement idempotency, meaning that running the module multiple times produces the same result. This is essential for reliable automation that can be run repeatedly.
  
- **Check Mode Support:**  
  Modules can support Ansible's check mode (dry run), allowing you to see what changes would be made without actually making them.
  
- **Structured Return Data:**  
  Modules return JSON responses that include metadata, such as changed status and output messages. This standardized format helps in debugging and using the results in subsequent tasks.
  
- **Reusability:**  
  Once packaged as a module, your automation logic can be distributed, reused, and maintained within your organization or shared with the community.

## Use Case Overview

In this example, the custom module performs the following operations:
- **Create a new virtual machine (VM)** using provided parameters (name, image, region, etc.).
- **Update or delete the VM** if needed.
- **Return a JSON result** indicating the success or failure, along with details of the action taken.

## Prerequisites

- Ansible 2.9+ installed on your control node.
- Python 3.6+.
- Access credentials and API endpoints for the targeted cloud provider.
- Basic familiarity with writing Ansible playbooks.

## Module File Structure

The custom module is contained in a single file (e.g., `vm_module.py`). Typical file structure for your project might be:

```
custom_ansible_module/
├── README.md
├── vm_module.py
└── playbook.yaml
```

## `vm_module.py` – The Custom Module

Below is a simplified version of the module implementation:

```python
#!/usr/bin/python

from ansible.module_utils.basic import AnsibleModule

def run_module():
    # Define the module's parameters
    module_args = dict(
        name=dict(type='str', required=True),
        image=dict(type='str', required=True),
        region=dict(type='str', required=True),
        state=dict(type='str', default='present', choices=['present', 'absent'])
    )

    # Seed result dictionary
    result = dict(
        changed=False,
        original_message='',
        message='',
        vm_details={}
    )

    # Create Ansible module instance
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    name = module.params['name']
    image = module.params['image']
    region = module.params['region']
    state = module.params['state']

    # For check mode, exit without making changes
    if module.check_mode:
        module.exit_json(**result)
    
    # Simulate processing (this is where you would integrate with APIs)
    if state == 'present':
        # Example logic: create or update virtual machine
        result['changed'] = True  # set changed to True if the VM is created/updated
        result['message'] = f"VM '{name}' is created in region '{region}' with image '{image}'."
        result['vm_details'] = {"name": name, "image": image, "region": region}
    elif state == 'absent':
        # Example logic: delete the virtual machine
        result['changed'] = True  # indicate if deletion occurred
        result['message'] = f"VM '{name}' is deleted."
        result['vm_details'] = {}

    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
```

*Explanation:*
- **Parameter Definition:** The module accepts parameters such as `name`, `image`, `region`, and `state`.
- **Check Mode:** Supports dry-run by returning without making changes.
- **Logic:** In a real-world module, replace the simulation code with API calls to create, update, or delete a VM.
- **Result:** Returns a structured JSON result.

## Example Playbook

Create a playbook `playbook.yaml` to utilize your custom module:

```yaml
- name: Create a virtual machine using a custom module
  hosts: localhost
  connection: local
  tasks:
    - name: Ensure the virtual machine is present
      vm_module:
        name: test-vm
        image: ubuntu-20.04
        region: us-east-1
        state: present
      register: vm_result

    - name: Print the result
      debug:
        var: vm_result
```

## Running the Module

1. **Place the module file (`vm_module.py`) in your module path** or set the `ANSIBLE_LIBRARY` environment variable to point to it.
2. **Execute the playbook:**

   ```bash
   ansible-playbook playbook.yaml
   ```

3. **Observe the output,** which should detail whether the VM was created (or deleted) and display the returned details in JSON format.

## Conclusion

Custom Ansible modules in Python transform reusable scripts into integrated automation components that support idempotency, check mode, and structured output. This makes them preferable to standalone scripts for robust, repeatable, and maintainable automation tasks – such as creating and managing virtual machines in the cloud.

For further details on writing modules, see:
- [Developing Ansible Modules](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules.html)
- [Ansible Module Development Best Practices](https://docs.ansible.com/ansible/latest/dev_guide/developing_modules_best_practices.html)
```


