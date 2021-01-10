# OpenShift 3.11 Install Cluster to Fyre (bare metal -on-prem) 
 Install procedure for OCP311 2-node  cluster to Fyre bare metal 
 
 
Install OpenShift Container Platform, v3.11 (on-premise installation)
-------------------------------------------------------------------

https://docs.openshift.com/container-platform/3.11/getting_started/install_openshift.html

Licensing:
Red Hat Subscription: use your Red Hat Partner Connect account
  Guide -->https://redhat-partner.highspot.com/items/5a8f2362f21676165cb39313?lfrm=shp.0#4
 Section: How do I request an NFR subscription?
  Request appropriate NFR licenses   https://partnercenter.redhat.com/NFR_Redirect
  
  NFR Licenses: 
   NFR Line Items
	NFR LI Name	      NFR Product Name	                                               Product Code Quantity (Pool)
 	NFR-LI-131601	Red Hat OpenShift, Standard Support (10 Cores, NFR, Partner Only)	SER0423	10

   *NFR Pool includes entitlements for OpenShift prerequisites: Ansible, RHEL 7, RH Atomic Host,  RHCOS,etc …

Prerequisites:
https://docs.openshift.com/container-platform/3.11/install/prerequisites.html

 -Valid Red Hat Subscription for OpenShift/Ansible/RHEL 7 (or RH Atomic Host) - NFR production license pool
 -At least two physical or virtual RHEL 7+ machines, with fully qualified domain names (either real world or within a network)
  These machines must be able to ping each other using their FQDN domain names
   And must resolve to the correct IP with nslookup (hosts file override only if nameserver can't resolve). 
   Just ping and hosts file aliases is not sufficient to build cluster.
 -Password-less SSH Access from master to all nodes 

---------

 1. Perform clean RHEL 7 (or RHEL Atomic Host 7) installs on master, and each node with "minimal" installation option (no GUI/Gnome) 
   https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/index

   **Important: very difficult to re-use an existing RHEL install for OCP, its easier just to do a clean install 
    And verify the only repos used are the following: 
    "rhel-7-server-rpms" \
     "rhel-7-server-extras-rpms" \
      "rhel-7-server-ose-3.11-rpms" \
      "rhel-7-server-ansible-2.6-rpms"
   
    If you try the install with additional repos, it will likely fail during cluster create and you will have to start over.
    Or you will spend endless hours in yum, trying to get the correct version of rpms.

    * I only used RHEL 7 (vs RHEL Atomic Host 7) for the greatest flexibility since resources in fyre are very limited.
      i.e. worker node, arlee1 is also stencil image for RH ODM,BPM/WBM stacks using fyre snapshots (can't do that with Atomic host)      

 2. Setup Set up Password-less SSH Access
    https://docs.openshift.com/container-platform/3.11/getting_started/install_openshift.html#set-up-password-less-ssh
    On master:  
     [root@ocp3-node11 ~] ssh-keygen 
     Generating public/private rsa key pair.
     Generating public/private rsa key pair.
     Enter file in which to save the key (/root/.ssh/id_rsa): /root/.ssh/id_rsa
     Enter passphrase (empty for no passphrase): 
     Enter same passphrase again: 
     Your identification has been saved in /root/.ssh/id_rsa
     Your public key has been saved in /root/.ssh/id_rsa.pub.
     The key fingerprint is:
     SHA256:6w+ovTxNZ1XIBA2Ysa/kF3me2c52bt7Qzj76FuRqewQ root@ocp3-node11.fyre.ibm.com
     The key's randomart image is:
     +---[RSA 2048]----+
     |         |
     |     Content Removed/
       Redacted for security purposes 
     
     +----[SHA256]-----+
     [root@ocp3-node11 ~] for host in ocp3-node11.fyre.ibm.com \
                    arlee1.fyre.ibm.com; \
                    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
                    done
       (Follow the prompts and just hit enter when asked for pass phrase.) 

      [root@ocp3-node11 ~]# ssh root@arlee1.fyre.ibm.com
        Last login: Mon Dec 30 21:16:39 PST 2019 on pts/1

      [root@arlee1 ~]# ssh-keygen 
     Generating public/private rsa key pair.
     Enter file in which to save the key (/root/.ssh/id_rsa):/root/.ssh/id_rsa 
     Enter passphrase (empty for no passphrase): 
     Enter same passphrase again: 
     Your identification has been saved in /root/.ssh/id_rsa.
     Your public key has been saved in /root/.ssh/id_rsa.pub.
     The key fingerprint is:
     SHA256:j9xtHaIIC72pYEMoSgaJ3IEAEkzuFiArXCJ0c7FOpOo root@arlee1.fyre.ibm.com
     The key's randomart image is:
       +---[RSA 2048]----+
     |         |
     |     Content Removed/
       Redacted for security purposes 
     
     
     
     +----[SHA256]-----+
     [root@arlee1 ~] for host in ocp3-node11.fyre.ibm.com \
                    arlee1.fyre.ibm.com; \
                    do ssh-copy-id -i ~/.ssh/id_rsa.pub $host; \
                    done
 
     Test out ssh in each direction: If you can ssh to the other node without entering password, it works.
    
       [root@ocp3-node11 ~]#  ssh root@arlee1.fyre.ibm.com
        Last login: Mon Dec 30 21:29:09 PST 2019 on pts/1 from 9.46.65.113

       [root@arlee1 ~]#  ssh root@ocp3-node11.fyre.ibm.com
         Last login: Mon Dec 30 21:42:12 PST 2019 on pts/1 from 9.160.34.126
       [root@ocp3-node11 ~]#
 

   3. Configure openshift inventory file for the cluster on master node  /etc/ansible/hosts
    Refer to RH Doc for specifics:  https://docs.openshift.com/container-platform/3.11/install/index.html

This is the inventory file that built the working 311 cluster-   
#######   /etc/ansible/hosts ###############
[OSEv3:children]
masters
nodes
etcd

[etcd]
ocp3-node11.fyre.ibm.com

[masters]
ocp3-node11.fyre.ibm.com

[nodes]
ocp3-node11.fyre.ibm.com openshift_node_group_name='node-config-master-infra'
arlee1.fyre.ibm.com   openshift_node_group_name='node-config-compute'

[OSEv3:vars]
ansible_ssh_user=root
ansible_become=false
debug_level=2
openshift_deployment_type=openshift-enterprise
openshift_image_tag=v3.11.117
oreg_url=registry.redhat.io/openshift3/ose-${component}:${version}
oreg_auth_user=<Your Red Hat Accnt Username>
oreg_auth_password=<Your Red Hat Accnt Password>
openshift_disable_check=memory_availability,disk_availability,docker_storage,docker_image_availability
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_htpasswd_users={'admin': 'passw0rd'}
openshift_docker_additional_registries=registry.redhat.io
openshift_docker_options="--log-driver json-file --log-opt max-size=1M --log-opt max-file=3"
openshift_master_cluster_method=native
openshift_master_default_subdomain=fyre.ibm.com
openshift_master_cluster_hostname=ocp3-node11.fyre.ibm.com
openshift_master_cluster_public_hostname=ocp3-node11.fyre.ibm.com  
openshift_console_hostname=adminconsole.fyre.ibm.com
openshift_master_api_port=8443
openshift_master_console_port=8443
ansible_service_broker_install=false
openshift_enable_service_catalog=false
template_service_broker_install=false
openshift_logging_install_logging=false
#openshift_master_bootstrap_auto_approve=true

 4. Update  /etc/hosts 
    Master:
    [root@ocp3-node11 ~]# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
9.46.65.113  ocp3-node11.fyre.ibm.com console console.fyre.ibm.com  grafana-openshift-monitoring.fyre.ibm.com prometheus-k8s-openshift-monitoring.fyre.ibm.com alertmanager-main-openshift-monitoring.fyre.ibm.com
9.30.247.243 arlee1.fyre.ibm.com   arlee1

  Worker:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
9.46.65.113  ocp3-node11.fyre.ibm.com console.fyre.ibm.com console
9.30.247.243 arlee1.fyre.ibm.com   arlee1

 **Note: we will be modifying the openshift web console app, to use a different cluster console URL
   once cluster is built

 5. On both master and worker nodes:
   # subscription-manager register
   Pull the latest subscription data from Red Hat Subscription Manager (RHSM):

   # subscription-manager refresh
   List the available subscriptions.

   # subscription-manager list --available
   Find the pool ID that provides OpenShift Container Platform subscription and attach it.

 # subscription-manager attach --pool=<pool_id> 
   this step uses a license from the pool
  pool_id is provided via NFR-LI-131601 pool for OpenShift/Ansible

 # subscription-manager repos --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ose-3.11-rpms" \
    --enable="rhel-7-server-ansible-2.6-rpms"

 6. Install products using yum
# yum -y install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
# yum -y update
# reboot

# yum -y install openshift-ansible
# yum -y install docker


7. Run the Installation Playbooks with Ansible
   
   Initialize env for deploy before running deploy_cluster playbook  (just to be safe)
   use init_ose.sh script for convenience.

 [root@ocp3-node11 bin]# cat /usr/local/bin/init_ose.sh
  #!/bin/bash
  service firewalld stop
  systemctl disable firewalld
  systemctl enable nfs-server 
  systemctl enable rpcbind
  systemctl enable nfs-lock 
  systemctl enable nfs-idmap

  systemctl start nfs-server 
  systemctl start rpcbind
  systemctl start nfs-lock 
  systemctl start nfs-idmap
  systemctl restart NetworkManager
  systemctl restart docker
  systemctl restart atomic-openshift-node.service

   
   Always run prerequisite playbook first, each time to eliminate every issue that is flagged during this phase.
   Note: passing the prerequisites run without errors does not mean the deploy_cluster will be successful. 
    
  cd /usr/share/ansible/openshift-ansible
  ansible-playbook -vv  -i /etc/ansible/hosts  playbooks/prerequisites.yml

  There will likely be many errors first round , where you have to correct the system.
  e.g.  SELinux settings, hostname, etc …
  Run prerequisites.yml playbook until no errors or warnings.

  A successful run of playbooks/prerequisites.yml will have a play recap similar to this:
  PLAY RECAP ************************************************************************************
arlee1.fyre.ibm.com        : ok=580  changed=125  unreachable=0    failed=0    skipped=1005 rescued=0    ignored=0   
localhost                  : ok=11   changed=0    unreachable=0    failed=0    skipped=5    rescued=0    ignored=0   
ocp3-node1.fyre.ibm.com    : ok=112  changed=25   unreachable=0    failed=0    skipped=178  rescued=0    ignored=0   

INSTALLER STATUS ******************************************************************************
Initialization               : Complete (0:01:15)
Health Check                 : Complete (0:00:20)
Node Bootstrap Preparation   : Complete (0:08:06)
etcd Install                 : Complete (0:02:21)
Master Install               : Complete (0:11:06)
Master Additional Install    : Complete (0:02:01)
Node Join                    : Complete (0:04:05)
Hosted Install               : Complete (0:01:47)
Cluster Monitoring Operator  : Complete (0:00:27)
Web Console Install          : Complete (0:01:06)
Console Install              : Complete (0:00:40)
Tuesday 17 December 2019  01:07:31 -0800 (0:00:00.054)       0:37:06.210 ****** 
=============================================================================== 

  Now build the cluster-
  ansible-playbook -vv /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml

  Again, it will fail several times at least.
  Correct as much as you can and re-run.
  
 **You must undeploy the cluster completely before rerunning the deploy playbook again.
  ansible-playbook -v -i /etc/ansible/hosts playbooks/adhoc/uninstall.yml 

  Use deploy_cluster.sh  and redeploy_cluster.sh scripts for convenience 
  [root@ocp3-node11 bin]# cat deploy_cluster.sh 
   #!/bin/bash
   cd /usr/share/ansible/openshift-ansible
   ansible-playbook -vv /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml | tee playbook-deploy_cluster.out
   config_admin_usr.sh

   [root@ocp3-node11 bin]# cat redeploy_cluster.sh 
   #!/bin/bash
   cd /usr/share/ansible/openshift-ansible
   ansible-playbook -v -i /etc/ansible/hosts playbooks/adhoc/uninstall.yml  | tee ~/adhoc-uninstall-yml.out
   ansible-playbook -v -i /etc/ansible/hosts playbooks/deploy_cluster.yml  | tee ~/deploy_cluster.out
  
  [root@ocp3-node11 bin]# cat config_admin_usr.sh 
  #!/bin/bash
   htpasswd -cb /etc/origin/master/htpasswd admin passw0rd
   oc adm policy add-cluster-role-to-user cluster-admin admin
 
  The install process has bugs depending upon the exact version of Ansible and also the 
  openshift image version (openshift_image_tag=v3.11.117 specified in inventory file)
  and generally won't complete without issues, here is what I ended up with.
  The cluster deploy will not complete in 1 pass without errors (with 1 exception)
  It took several passes and a few tricks after running the deploy_cluster.yml.
  It could be environmental, but the nodes and containers need to startup and be in ready state during the run, which doesn't
  always happen in time for a task that relies  on it. 
    Node registration, service CRDs , control_plane , CSR's …  
   If you encounter these failures, google the actual failure and likely you will find some workaround/fix on GitHub in OpenShift or Origin communities.
  Here are some examples that I encountered and found solutions to resolve-
    https://github.com/openshift/openshift-ansible/issues/11333
    https://github.com/openshift/openshift-ansible/issues/10427
    https://access.redhat.com/solutions/3928701 (I needed this for docker)

  Running the deploy_cluster.yml playbook takes a long time, up to 1 hour or more.

 - First pass of deploy cluster-
 ansible-playbook -v -i /etc/ansible/hosts playbooks/deploy_cluster.yml   | tee  deploy_cluster_.out
 the join node failed in the task…
 TASK [openshift_manage_node : Wait for Node Registration] ************************************
 FAILED - RETRYING: Wait for Node Registration (1 retries left).
 fatal: [arlee1.fyre.ibm.com -> ocp3-node11.fyre.ibm.com]: FAILED! => {"attempts": 50, "changed": false, "module_results": {"cmd": "/usr/bin/oc get node   arlee1.fyre.ibm.com -o json -n default", "results": [{}], "returncode": 0, "stderr": "Error from server (NotFound): nodes \"arlee1.fyre.ibm.com\" not  found\n", "stdout": ""}, "state": "list"}

 and in turn ends up failing on the web-console …
 TASK [openshift_web_console : Verify that the console is running] ****************************
 Sunday 05 January 2020  04:17:43 -0800 (0:00:00.147)       0:28:46.888 ******** 
 FAILED - RETRYING: Verify that the console is running (60 retries left).

  --WORKAROUND--
 - I restarted docker on master and worker nodes, them NetworkManager and atomic-openshift-node.service
   and reran join node playbook, and the cluster was then deployed successfully-
    systemctl restart docker
    systemctl restart  NetworkManager.service
    systemctl restart atomic-openshift-node.service
    systemctl status atomic-openshift-node.service
    ansible-playbook -vv -i /etc/ansible/hosts playbooks/openshift-node/join.yml --skip-tags=csr  | tee ~/join-node_skipCSR.out
 
 - I validated the nodes were correct and in ready state in oc
 [root@ocp3-node11 bin]# oc get nodes
  NAME                       STATUS    ROLES          AGE       VERSION
  arlee1.fyre.ibm.com        Ready     compute        5m        v1.11.0+d4cacc0
  ocp3-node11.fyre.ibm.com   Ready     infra,master   5m        v1.11.0+d4cacc0
  
  Then reran playbooks/openshift-web-console/config.yml  and the web-console installed successfully.
   
 A successful run of playbooks/deploy_cluster.yml  will have a play recap similar to this, even if it  
  In this case the run was split up into 3 passes:
   ansible-playbook -v -i /etc/ansible/hosts playbooks/deploy_cluster.yml
   ansible-playbook -v -i /etc/ansible/hosts playbooks/openshift-node/join.yml
   ansible-playbook -v -i /etc/ansible/hostsplaybooks/openshift-web-console/config.yml

 When you see both Web Console Install and Console install completed without failures, 
 the cluster will likely be running in a healthy state. 

  PLAY RECAP *****************************************************************************
arlee1.fyre.ibm.com        : ok=112  changed=20   unreachable=0    failed=0
localhost                  : ok=11   changed=0    unreachable=0    failed=0
ocp3-node11.fyre.ibm.com   : ok=489  changed=97   unreachable=0    failed=0
INSTALLER STATUS *************************************************************
Initialization               : Complete (0:00:25)
Health Check                 : Complete (0:00:14)
Node Bootstrap Preparation   : Complete (0:05:28)
etcd Install                 : Complete (0:00:48)
Master Install               : Complete (0:02:46)
Master Additional Install    : Complete (0:01:01)
Node Join                    : Complete (0:00:26)
Hosted Install               : Complete (0:00:37)
Cluster Monitoring Operator  : Complete (0:00:12)
Web Console Install          : Complete (0:00:22)
Console Install              : Complete (0:00:15)
Monday 13 January 2020  02:32:01 -0800 (0:00:00.043)       0:13:09.879 ********
===============================================================================

  The cluster is now built and started.



8. Customize OpenShift Web Console app, cluster console URL (using Ansible)
# This step is needed for a shared private cloud env like fyre.
  Initial deploy cluster console cluster console URL  Customize OpenShift Web Console so cluster console  URL is unique #####
# example, everyone who has deployed OSE 3.11 cluster in fyre domain (as bare metal servers) will use defaults …  console.fyre.ibm.com
# in that case, cluster console will not work, cannot resolve to my master-infra node IP
# one workaround, is to change the cluster console URL in the openshift-web-console application to a unique URL… adminconsole.fyre.ibm.com
#
# example:
# modify cluster console URL
# Change URL for cluster console - https://access.redhat.com/solutions/4037891
# first uninstall web console
[root@ocp3-node11 openshift-ansible]# ansible-playbook -v -i /etc/ansible/hosts playbooks/openshift-console/config.yml -e openshift_console_install=false

# re-install with new URL
[root@ocp3-node11 openshift-ansible]# ansible-playbook -v -i /etc/ansible/hosts playbooks/openshift-console/config.yml -e openshift_console_install=true  -e openshift_console_hostname=adminconsole.fyre.ibm.com
# May need to change this as well
 oc edit cm webconsole-config -n openshift-web-console
- Update line for adminConsolePublicURL to match the value from openshift_console_hostname.

oc edit cm webconsole-config -n openshift-web-console

 Manually modify the Web Console to point to new Cluster Console hostname 
  - oc edit cm webconsole-config -n openshift-web-console
  - Update line for adminConsolePublicURL to match the value from openshift_console_hostname

 Restart Web Console pods:
 # oc delete pods -n openshift-web-console -l app=openshift-web-console

9. Install DB2 on worker node
   This gives the best performance , i.e.  when db is local to the ODM pods.
   
   LDAP is not being used in this lab .. I have a fyre stack that has an LDAP server,
   can re-install or create separate a project with LDAP integration. Perhaps with ICP4A 19.0.3
   
    
   You are now ready to install ICP4A

-----
  

