# gcp-wih-ansible

We’ll use the Ansible playbook to create a VM disk, a VPC network, an IPv4 address, and finally our new instance of Red Hat Enterprise Linux 8.
I’m showing the Ansible playbook in Figure 7 as a screenshot to preserve the indentation. You can use the playbook and use it as an example (gcp-resources.yaml).

```shell
- name: Create a GCP instance
  hosts: localhost
  vars:
     gcp_project: ansible-tower-sreejith
     gcp_cred_kind: serviceaccount
  tasks:
    - name: create a disk mapped from RHEL8 image
      gcp_compute_disk:
        name: disk-instance
        size_gb: 50
        source_image: projects/rhel-cloud/global/images/rhel-8-v20190905
        zone: us-central1-a
        project: "{{ gcp_project }}"
        auth_kind: "{{ gcp_cred_kind }}"
        state: present
      register: disk

    - name: create a  VPC network
      gcp_compute_network:
        name: network-instance
        project: "{{ gcp_project }}"
        auth_kind: "{{ gcp_cred_kind }}"
        state: present
      register: network

    - name: create an IPv4 public IP Address
      gcp_compute_address:
        name: address-instance
        region: us-central1
        project: "{{ gcp_project }}"
        auth_kind: "{{ gcp_cred_kind }}"
        state: present
      register: address

    - name: create the RHEL8 instance
      gcp_compute_instance:
        name: rhel8
        machine_type: n1-standard-1
        disks:
        - auto_delete: 'true'
          boot: 'true'
          source: "{{ disk }}"
        network_interfaces:
        - network: "{{ network }}"
          access_configs:
          - name: External NAT
            nat_ip: "{{ address }}"
            type: ONE_TO_ONE_NAT
        zone: us-central1-a
        project: "{{ gcp_project }}"
        auth_kind: "{{ gcp_cred_kind }}"
        state: present
    - name: Show RHEL8 Instance Details
      debug:
        msg: "The RHEL8 instance is accessible at {{ address.address }}"

 ```

## Let’s quickly look at each task in the playbook:
### Task 1: 
Create the VM disk: We can use a source image to create a VM disk and make the disk bootable. In this case, we’ll use the certified Red Hat Enterprise Linux 8 image available from Google Cloud Platform. The gcp_compute_disk module uses the rhel-8-v20190905 image to add a persistent disk for our Red Hat Enterprise Linux 8 instance.
### Task 2: 
Create the VPC network: Next, the gcp_compute_network module creates a VPC network. The Red Hat Enterprise Linux instance will have an interface associated with that network.
### Task 3: 
Create the IPv4 address: The gcp_compute_address module allocates an external IPv4 address to be associated with the Red Hat Enterprise Linux instance.
### Task 4: 
Create the Red Hat Enterprise Linux instance: The gcp_compute_instance module uses the resources from the previous tasks to create an instance of Red Hat Enterprise Linux 8.
### Task 5: 
Check the IPv4 address: The debug module shows the IPv4 address associated with the Red Hat Enterprise Linux 8 instance.
Note that we will execute the playbook within Ansible Tower. We’ll use the auth_kind attribute to reference the Google Cloud Platform credentials required for each task. We’ll then use the gcp_cred_kind variable to map the tasks to our Google Cloud Platform serviceaccount.
During execution, Ansible Tower will use the GCP_AUTH_KIND environment variable to reference and pass the Google Cloud Platform service account’s email, project, and private-key contents to the playbook.
## Execute the playbook
Next, we want to create a new job template to execute the playbook in Ansible Tower. Before we can do that, we need to create an inventory and a project referencing the playbook. 

## Run the job template
Now, we can run the job template. Check the Google Cloud Platform console to see the new resources being created as each individual task is executed. From the job template output shown in Figure 9, you can see that the new resources were created.

![img](https://github.com/alioualarbi/gcp-wih-ansible/blob/main/task%20output.png)

You can also use the Google Cloud Platform console to verify the resources. In Figure 10, you can see the VM disk, disk-instance, was created with a size of 50 GB.
![img](https://github.com/alioualarbi/gcp-wih-ansible/blob/main/disk.png)
the below screenshotshows that the VPC network, network-instance, was created on subnet 10.240.0.0/16.
![img](https://github.com/alioualarbi/gcp-wih-ansible/blob/main/network.png)

# Create a dynamic inventory for GCP instances
## Connect your GCP service account to Ansible Tower

Ansible Tower lets you periodically sync with the Google Cloud API to find realtime instance counts and details for resources hosted on Google Cloud Platform.
To create a new inventory, choose Google Compute Engine as the source, then you can select the Google Cloud Platform credential (Service account).

![img](https://github.com/alioualarbi/gcp-wih-ansible/blob/main/ansible%20tower%20dashboard.png)

Next, use the Inventory Sync button to synchronize the newly created instance rhel8 in the US Central (A) region.
Once synced, check the new hosts in the inventory
![img](https://github.com/alioualarbi/gcp-wih-ansible/blob/main/ansible%20tower%20sync.png)
