# check_ibm_flashsystem
nagios check for IBM SVC / Storwize / Spectrum Virtualize / FlashSystem storage 

NOTES
-----
This script runs on a nagios server and connects to an IBM SVC / Storwize / Spectrum Virtualization / FlashSystem via SSH to check the following:
- name resolution
- ping reply
- SSH connectivity
- node cluster status
- node health status
- controller health
- SMTP call home configuration
- NTP time sync
- mdisk status (check for degraded mdisks)
- vdisk status (check for offline vdisks)
- pdisk status (check for offline physical disks)
- enclosure drive slots (check for faults)
- FlashCopy consistency group status
- Remote Copy consistency group status
- Metro Mirror / Global Mirror status
- Generate HTML report of vdisks and mirror relationships
- alert on known buggy firmware versions
- alert on obsolete / unsupported firmware versions
- high CPU utilization
- high read / write latency
- contents of error logs
  
  
 
 

SUPPORTED OPERATING SYSTEMS
---------------------------
- Tested on AIX 6.1, 7.1, 7.2
- Tested on CentOS 7, CentOS 8, Ubuntu 20.04
- Will likely work on any UNIX-like operating system with perl and ssh



ASSUMPTIONS
-----------
It is assumed that perl is installed on the machine running this script.  
For RHEL / CentOS     `yum install perl`  
For Debian / Ubuntu   `apt install perl`  
For AIX               (perl should already be in base install)  

It is assumed that this script is being run as a low-privileged user (typically nagios)
  
It is assumed that SSH key pairs have been created between the nagios server and the storage system.





SETUP SSH KEY PAIR AUTHENTICATION
---------------------------------
If the nagios userid on the nagios server does not already have an SSH key pair, please create with:  

    su - nagios
    ssh-keygen -t rsa
 

   
Copy the contents of $HOME/.ssh/id_rsa.pub to a temporary file on your desktop.  This file will be used to upload the SSH public key to the storage system via a web browser.
   
Perform the following steps on the storage system:
a) Point your web browser at the management interface.  Login as superuser (or equivalent).
b) Click User Management, Users, New user
c) Set the username to: nagios
d) Set the Authentication mode to: local
e) Set the User Group to: monitor   (this is a read-only account) (set to administrator group if you want to restart stopped mirrors)
f) Click the Browse button to find the SSH public key for the nagios account.
g) Select the temp file you created earlier.
h) Click the Create button
   
You will need to manually ssh from the nagios server to each storage system to update the known_hosts file on the nagios server.  
    $ ssh admin@10.10.8.191
    RSA key fingerprint is ea:a1:05:58:8d:4e:4e:c4:82:db:cf:87:75:a6:7c:7f.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '10.10.8.191' (RSA) to the list of known hosts.

   


USAGE 
-----
This script is executed remotely on a monitored system by the NRPE or check_by_ssh methods available in nagios.

If you are using the check_by_ssh method, you will need a section in the services.cfg file on the nagios server that looks similar to the following.
This assumes that you already have ssh key pairs configured.
    
       define service{
           use                             generic-24x7-service
           hostgroup_name                  all_ibm_flashsystem
           service_description             IBM FlashSystem
           check_command                   check_by_ssh!"/usr/local/nagios/libexec/check_ibm_flashsystem"
           }

If you are using the check_nrpe method, you will need a section in the services.cfg file on the nagios server that looks similar to the following.
This assumes that you already have ssh key pairs configured.
  
       define service{
           use                             generic-24x7-service
           host_name                       v7000prod,v7000dev,svc01,flashsys02
           service_description             IBM FlashSystem
           check_command                   check_nrpe!check_ibm_flashsystem
           }

If using NRPE, you will also need a section defining the NRPE command in the /usr/local/nagios/nrpe.cfg file that looks like this:  
    command[check_ibm_flashsystem]=/usr/local/nagios/libexec/check_ibm_flashsystem

This script can take several minutes to run, which can potentially cause timeouts in nagios.  We work around this by scheduling a cron job to run every 15 minutes.
Schedule this script to run every 15 minutes from the nagios user crontab, which will update a file at `/tmp/nagios.check_ibm_flashsystem.$hostname.tmp`  

When this script runs as the low-privileged nagios user, the script will read the contents of `/tmp/nagios.check_ibm_flashsystem.$hostname.tmp`  
Create cron entries in the nagios user crontab similar to the following:  

    1,16,31,46 * * * * /usr/local/nagios/libexec/check_ibm_flashsystem v7000prod  1>/dev/null 2>/dev/null
    1,16,31,46 * * * * /usr/local/nagios/libexec/check_ibm_flashsystem v7000dev   1>/dev/null 2>/dev/null
    1,16,31,46 * * * * /usr/local/nagios/libexec/check_ibm_flashsystem svc01      1>/dev/null 2>/dev/null
     
  
