
# Ansible dynamic assignment and community roles

Let's reconstruct this Ansible project based on the provided details. Here's a detailed breakdown of each step and how to implement the dynamic assignment and community roles for MySQL and Load Balancers.

---

### 1. **Understanding Static vs Dynamic Assignments**

- **Static Assignments (`import`)**: 
  These tasks are processed at the time the playbook is parsed. Ansible will load all playbooks that are `imported` upfront.
  
- **Dynamic Assignments (`include`)**:
  These tasks are processed during the execution of the playbook, allowing for flexibility (e.g., choosing roles based on specific environment variables). The `include` module enables dynamic loading.

### 2. **Setting Up the Project Structure**

#### Step 1: **Repository Creation**

Create a GitHub repository called `ansible-config-mgt`.

```
mkdir ansible-config-mgt
cd ansible-config-mgt
git init
git checkout -b dynamic-assignments
```
![image](https://github.com/user-attachments/assets/4c14c1e6-207c-4ac6-b468-524cb35a8f01)


#### Step 2: **Creating Dynamic Assignments Folder**

You will create a folder named `dynamic-assignments` to hold the dynamically assigned environment variables.

```
mkdir dynamic-assignments
touch dynamic-assignments/env-vars.yml
```

#### Step 3: **Creating Environment Variables Files**

Inside `dynamic-assignments/env-vars.yml`, we will dynamically include different environment-specific files using `include_vars`. The file will loop through and load the relevant variable file based on the environment (e.g., dev, stage, prod, uat).

```
---
- name: Collate variables from environment-specific file, if it exists
  hosts: all
  tasks:
    - name: Loop through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - dev.yml
            - stage.yml
            - prod.yml
            - uat.yml
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always
```

Then, create the `env-vars` folder and environment-specific variable files.

```
mkdir env-vars
touch env-vars/dev.yml env-vars/stage.yml env-vars/prod.yml env-vars/uat.yml
```
![image](https://github.com/user-attachments/assets/f0011562-6cdc-44ad-82d2-43ebca46fcb0)

For example, `env-vars/dev.yml` could look like this:

```
---
enable_nginx_lb: true
load_balancer_is_required: true
```

Similarly, for `env-vars/prod.yml`:

```
---
enable_nginx_lb: false
enable_apache_lb: true
load_balancer_is_required: true
```

### 3. **Updating the `site.yml` Playbook**

In the `site.yml`, you will use both static and dynamic assignments. The static ones handle general tasks that are the same for all environments, and the dynamic assignments handle environment-specific variables.

```
touch site.yml
```

```
---
- name: Include dynamic variables
  hosts: all
  tasks:
    import_playbook: ../static-assignments/common.yml
    include: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- name: Webserver assignment
  hosts: webservers
  tasks:
    import_playbook: ../static-assignments/webservers.yml

- name: Loadbalancers assignment
  hosts: lb
  tasks:
    import_playbook: ../static-assignments/loadbalancers.yml
    when: load_balancer_is_required
```

This will ensure that dynamic environment variables are loaded based on the specific environment you are running Ansible against.

### 4. **MySQL Role from Ansible Galaxy**

To avoid reinventing the wheel, we'll use a community role for managing MySQL. This can be installed using the following command:

```
ansible-galaxy install geerlingguy.mysql
mv geerlingguy.mysql/ roles/mysql
```
![image](https://github.com/user-attachments/assets/8a97526c-ec87-4702-b6de-788e1d3940d0)


Modify the `roles/mysql/defaults/main.yml` to include the MySQL configurations required for your project. For instance:

```
---
mysql_root_password: "root_password"
mysql_databases:
  - name: tooling_db
mysql_users:
  - name: tooling_user
    password: "password"
    priv: "tooling_db.*:ALL"
```
![image](https://github.com/user-attachments/assets/1a4ec127-111e-47fb-9018-ecfaf93c761e)


### 5. **Setting Up Load Balancer Roles**

You will create roles for managing Nginx and Apache as load balancers. For flexibility, you can choose which load balancer to use based on environment variables.

#### Step 1: **Creating the Nginx Role**

Create the role directory structure for Nginx.

```
mkdir -p roles/nginx/tasks
touch roles/nginx/tasks/main.yml
```

Populate `main.yml` for Nginx setup:

```
---
- name: Install Nginx
  yum:
    name: nginx
    state: present
```

#### Step 2: **Creating the Apache Role**

Similarly, create the role directory for Apache:

```
mkdir -p roles/apache/tasks
touch roles/apache/tasks/main.yml
```

Populate `main.yml` for Apache setup:

```
---
- name: Install Apache
  yum:
    name: httpd
    state: present
```

### 6. **Updating `loadbalancers.yml`**

You will now update the `static-assignments/loadbalancers.yml` file to include roles based on conditions for the load balancers.

```
touch static-assignments/loadbalancers.yml
```

Populate the file:

```
---
- hosts: lb
  roles:
    - { role: nginx, when: enable_nginx_lb and load_balancer_is_required }
    - { role: apache, when: enable_apache_lb and load_balancer_is_required }
```

### 7. **Commit and Push to GitHub**

Finally, commit the changes and push them to the GitHub repository:

```
git add .
git commit -m "Commit new dynamic assignment and roles for MySQL and Load Balancers"
git push --set-upstream origin dynamic-assignments
```

### 8. **Testing the Playbook**

To test the setup, update your Ansible inventory file with different environments and run the playbook:

```
ansible-playbook -i inventory/uat site.yml
```

This will use the environment-specific variables (`uat.yml`) to determine which roles to run and set up the necessary configurations, like the load balancers and MySQL.

---

### **Conclusion**

This project setup allows for flexible dynamic assignments through `include_vars` and uses community roles for easier management of tasks like MySQL installation. The ability to switch between Nginx and Apache based on environment-specific variables also introduces flexibility in the infrastructure management process.
