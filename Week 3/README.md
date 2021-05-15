# WEEK-3 PRACTICES DOCUMENTATION
# ALL-IN-ONE OPENSTACK DEPLOYMENT
---
## **Author:** *Julian (Phong) Ng.* 
**Date of issue**: *May 16th 2021*

> Welcome back! This is the documentation for my second training project at **Viettelnet**. Enjoy ur time :smile_cat:. Feel free to hit me up if any edition is needed!

---
## **Table of Contents:**

#### [I. Overview](#I.-OVERVIEW)
#### [II. Prerequisite](#II.-Prerequisite)
#### [III. Step-by-step](#III.-Step-by-step)
#### [IV. Debugging](#IV.-Debugging)
#### [V. References](#V.-References)

# **I. OVERVIEW**:
## A. `OPENSTACK`

## B. `KOLLA-ANSIBLE`

# **II. PREREQUISITE**:

# **III. STEP-BY-STEP**:

## A. SET UP ENIVRONMENT:

### 1. Update `apt` & install essentails dependencies:

```
$ sudo apt update

$ sudo apt install python3-dev libffi-dev gcc libssl-dev

```

### 2. Using `virtualenv`:
- Install `virtualenv`:
```
$ sudo apt install python3-venv
```

- Create `virutalenv` & activate that environment:
```
$ python3 -m venv /path/to/venv

$ source /path/to/venv/bin/activate
```

### 3. Install `Ansible` & `Kolla-Ansible` (within `virtualenv`):

- Install `Ansible`:
```
$ pip install 'ansible==2.9'
```

- Install `Kolla-ansible`:

```
$ pip install kolla-ansible
```

### 4. Install `Openstack CLI`:

**Notes:**
> *Optional at this point*

> Due to the fact that `openVSwitch` might get `MACAddress` of network interace, which blocks connection to the Internet. Can install Openstack CLI from this step.

```
$ pip install python-openstackclient python-glanceclient python-neutronclient
```

## B. CONFIGURE `Kolla-Ansible` & `Ansible`:
### 1. Create `/etc/kolla`  directory:

```
$ sudo mkdir -p /etc/kolla
$ sudo chown $USER:$USER /etc/kolla
```

### 2. Copy `passwords.yml` to `/etc/kolla`:
 
```
$ cp -r <path-to-virtualenv>/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
```

### 3. Configure `Ansible`:

```
$ mkdir -p /etc/ansible

$ config="[defaults]\nhost_key_checking=False\npipelining=True\nforks=100"

$ echo -e $config >> /etc/ansible/ansible.cfg
```

## C. PRE-DEPLOY CONFIGURATIONS:

### 1. Configure `all-in-one` (`inventory` file)
**Note**
> Optional. Should defaults for `all-in-one`

### 2. Run `ad-hoc` command `ping` to check configurations:
```
$ ansible -i all-in-one all -m ping
```

> Ping Success:

<img src="./imgs/ping-aio-success.png">


### 3. Create diskspace partition for `Cinder` (*Block Storage*):

```
$ sudo pvcreate /dev/sdb

$ sudo vgcreate cinder-volumes /dev/sdb
```

### 4. Generate Passwords for `Kolla`:
- Stored in `/etc/kolla/passwords.yml`, run commands:

```
$ kolla-genpwd
```

Or

```
$ cd kolla-ansible/tools
$ ./generate_passwords.py
```

### 5. Configure `globals.yml`:

```
$ vi /etc/kolla/globals.yml
```

**Example**: Sample `globals.yml` file

```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"

openstack_release: "train"

network_interface: ens33
neutron_external_interface: ens38

nova_compute_virt_type: "qemu"

enable_haproxy: "no"

enable_cinder: "yes"
enable_cinder_backup: "no"
enable_cinder_backend_lvm: "yes"

```

## D. DEPLOY `OPENSTACK`
- Bootstrap:

> **Debug**: [*Ansible Module Missing*](#'2.-`Cannot-import-name-'AnsibleCollectionLoader'-from-'ansible.utils.collection_loader'-during-Boostrapping`')

```
$ kolla-ansible -i all-in-one bootstrap-servers
```

> Boostrapping Success

<img src="./imgs/success-bootstrap.png">


- Prechecks:
```
$ kolla-ansible -i all-in-one prechecks
```

> Boostrapping Success

<img src="./imgs/success-bootstrap.png">

- Pull Images:
```
$ kolla-ansible -i all-in-one pull
```

>  Pulling Images Success

<img src="./imgs/pull-success.png">

- Deploy:
```
$ kolla-ansible -i all-in-one deploy
```

>  Deploy Success

<img src="./imgs/success-deploy.png">

- Post-deploy:
```
$ kolla-ansible -i all-in-one post-deploy
```

## E. POST-DEPLOYMENT:
- Install Openstack CLI:
```
$ pip install python-openstackclient python-glanceclient python-neutronclient
```

- Run `admin-openrc.sh` to add `ENVIRONMENT VARIABLES`: 
> **Debug**: [*`admin-rc.sh` Not found*](#'3.-`admin-openrc.sh`-missing')
```
$ source /etc/kolla/admin-openrc.sh
```

- Generate token:
```
$ openstack token issue
```

> Token generated

<img src="./imgs/token-issue.png">


## F. ACESSING `HORIZON` DASHBOARD:

> Openstack Login page

<img src="./imgs/success-openstack.png">

> Openstack Dashboard

<img src="./imgs/dashboard-openstack.png">


# **IV. DEBUGGING**:
### 1. `IPv4 not available for ens38`:

- View all NICs & statuses:
```
$ ip r
```

- Remove old IP & Request new IP via `DHCP`:

```
$ sudo dhclient -r <nic-address>

$ sudo dhclient <nic-addr>
```

### 2. `Cannot import name 'AnsibleCollectionLoader' from 'ansible.utils.collection_loader' during Boostrapping`

- Remove existing `ansible` & `ansible-base` packages:

```
$ pip uninstall <package-name> 
```

- Install `Ansible==2.9.0`:
```
$ pip install 'ansible==2.9'
```

<img src="./imgs/kolla-ansible-bootstrap-err.png">

### 3. `admin-openrc.sh` missing:
- Create `/etc/kolla/admin-openrc.sh`

```
$ vi /etc/kolla/admin-openrc.sh

--------------------------------
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://<INTERNAL-IP-ADDRESS>:<PORT-OF-KEYSTONE>/<API-VER>
export OS_IDENTITY_API_VERSION=<API-VER>
```

**Explains**:
<dl>
    <dt>
      INTERNAL-IP-ADDRESS
    </dt>
    <dd>
       Defined internal Network interface.
       <dd><b>Current deployment</b>: ens33 - 192.168.80.137</dd>
    </dd>
	<dt>
		PORT-OF-KEYSTONE
	</dt>
	<dd>
		Port running Keystone Service - Authentication Component (<b>Default</b>: 35357)
	</dd>
    <dt>
		API-VER
	</dt>
	<dd>
		API Version (<b>Default</b>: 3)
	</dd>
</dl>

# **V. REFERENCES**:

