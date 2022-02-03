# Ansible for Database Creation

This ansible project allows for Database Home, Database, and PDB creation and termination through OCI Modules. 

## Getting Started

### Using Ansible with OCI

These playbooks utilize the OCI Ansible modules for database tasks and therefore run on the localhost and use OCI API authentication. The following two set-up steps are required for these playbooks.

1. OCI Authentication: Follow the instructions on this documentation to make sure you are set-up to use ansible with the OCI modules: https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/ansiblegetstarted.htm.

2. Create a file with the VM Cluster OCID if it does not already exist. The file should have one line: vmcluster_ocid=ocid1....

    For ExaCC, this file should be located at /var/opt/oracle/exacc.props

    For ExaCS, there is no set standard but be sure to set the path in the exacs_ocid.yml file in the target_common role. Currently the path is set to be /opt/oracle/exacs.props

## Playbook Execution

### Inventory

Edit the inventory file in this github repo. Add your different Exadata hosts under different groups. Be sure to add in any required ssh args. Check the sample_inventory file for an example or refer to https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#inventory-basics-formats-hosts-and-groups for more information.

Skip this step if you are running these playbooks from an automation system that already tracks inventory, such as Ansible Tower or Rundeck.

### Variables

The roles > defaults > main.yml files for each role has default variables to set according to your environment. Some of the default variables are optional and can be overrided. 

Required variables can be passed in on the command line or through a variable file to be reused for additional operations such as backups or resource termination. Refer to sample_database.yml, sample_database_home.yml, and sample_pdb.yml, and sample_exacc_backup_details.yml in the vars folder for example variable values. 

### Running a job

1. If running a playbook, execute `ansible-playbook -i inventory <ansible_playbook.yml> --extra-vars "<runtime variables>"`. For example: `ansible-playbook -i inventory db_create.yml --extra-vars "@vars/sample_database.yml db_admin_password=$DB_SECRET hostgroup=exacc1"`.

2. Runtime Variables:
    - hostgroup: Host group to run DB operations on, should exist in the inventory
    - additional variables: Sensitive variables (such as passwords) should be only defined at runtime and stored in secrets, not set in a variable file. Check out the Playbooks below for instructions on which plays require additional variables.


## Ansible Codebase

This codebase contains a set of playbooks that can be used individually or combined into a workflow for Database Set-up. Each playbook references ansible roles (an ansible file structure used to group reusable components). Each ansible role folder contains three subdirectories - tasks, defaults, and meta. In the case of the database_scripts role, there is a fourth folder (files) to store scripts used in database set-up.

- Tasks: Contains a main.yml file that will be automatically called if the role is invoked. Also contains reusable standalone tasks.
- Defaults: Default variable values for the tasks in that role. These variables have the least precedence and will be overrided by any variables defined in the included variable file (vars_file) or in the ansible job template. Many of these variables are set as null as they are optional variables for the oci tasks and it is your choice whether to define them. For required variables, check the sample variable files or the oracle.oci ansible documentation. 
- Meta: Sets collection oracle.oci


### Playbooks

**test_setup.yml**
- Tests if the ansible environment has been set-up correctly. Connects to provided host and checks VM Cluster OCID in properties file. Then gets VM Cluster information using OCI module.
- Runtime Variables
    - hostgroup
    - exadata_type
    

**db_home_create.yml**
- Creates a new database home. Identifies the VM Cluster where the new database home will be created, queries the VM Cluster to check for any existing database homes with the same name, and then creates a new database home.
- Variables
    - hostgroup
    - exadata_type ('exacs' or 'exacc')
    - db_home_name
    - db_base_version

**db_home_terminate.yml**
- Terminates a database home. 
- Variables
    - hostgroup
    - db_home_id or db_home_name

**db_create.yml**
- Creates a new database, deletes the automatic PDB created, updates audit trail and default encryption algorithm and bounces database, and then sets automatic backups for ExaCS. Assumes that the database home has already been created.
- Variables
    - hostgroup
    - exadata_type ('exacs' or 'exacc')
    - cdb_name
    - db_home_name
    - db_admin_password - should be saved as secret / encrypted value within rundeck
    - auto_backup_window - ExaCS only, optional - also set in defaults)
    - recovery_window_in_days - ExaCS only, optional - also set in defaults)

**db_terminate.yml**
- Terminates a database.
- Variables
    - hostgroup
    - database_id or cdb_name

**db_setup_script.yml**
- Unarchives newdb zip file and runs DB setup scripts.
- Variables
    - hostgroup

**db_backup.yml**
- Creates a standlone backup for ExaCS. Creates a backup destination (if backup_destination_id is not provided) and adds backup destination to DB for ExaCC.
- Variables
    - database_id [database discovery information](#database_id) to pull database_id from oci
    - exadata_type ('exacs' or 'exacc')  
    - If exacc, additional values based on backup destination type (check sample_exacc.yml)

**pdb_create.yml**
- Creates a new pdb. Assumes that the database has already been created.
- Variables
    - hostgroup
    - pdb_name
    - cdb_name

**pdb_delete.yml**
- Deletes the pdb on an existing database.
- Variables
    - hostgroup
    - pdb_name
    - cdb_name


## Additional Resources

OCI Collection for ansible: https://oci-ansible-collection.readthedocs.io/en/stable/collections/oracle/oci/index.html

