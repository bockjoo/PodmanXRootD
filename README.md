# PodmanXRootD

Bockjoo Kim, August 28, 2023


## [1] Introduction
This doocumentation describes a procedure for a podman based XRootD deployment at RC for the Florida CMS Tier2.

## [2] Preparing the Machine for the XRootD podman Needs Privileged Actions: Puppetization
### [2-1] Hardware and OS
<pre>
cmsio machines can be used as the host machines.
RHEL8 or RHEL9 host needs to be prepared.
For this documentation, two VMs are used to evaluate the podman solution.
</pre>
### [2-2] Package Installation
<pre>
<b>A. Explannation</b>
podman, podman-compose, buildah, and fuse-overlayfs are the primary packages needed.
The others are either automatic subsidiary packages or convenience packages.
<b>B. In Puppet, something like is needed: </b>
yum -y install podman btrfs-progs-devel conmon containernetworking-plugins  \
       containers-common crun device-mapper-devel git glib2-devel glibc-devel \
       glibc-static go golang-github-cpuguy83-md2man gpgme-devel iptables \
       libassuan-devel libgpg-error-devel libseccomp-devel libselinux-devel \
       make pkgconfig
yum install -y slirp4netns
yum install -y fuse-overlayfs
yum install -y netavark
yum install podman-compose buildah
</pre>
### [2-3] User Namespace
<pre>

<b>A. A process of configuring user namespaces </b>
At this stage, podman ps would result in:
-bash-4.2$ podman ps
cannot clone: Invalid argument
user namespaces are not enabled in /proc/sys/user/max_user_namespaces
Error: could not get runtime: cannot re-exec process
-bash-4.2$ cat /proc/sys/user/max_user_namespaces
0
[~]# echo 10000 > /proc/sys/user/max_user_namespaces

-bash-4.2$ podman ps
ERRO[0000] cannot find mappings for user bockjoo: No subuid ranges found for user "bockjoo" in /etc/subuid 
ERRO[0000] cannot find mappings for user bockjoo: No subuid ranges found for user "bockjoo" in /etc/subuid 
CONTAINER ID  IMAGE  COMMAND  CREATED  STATUS  PORTS  NAMES

Add the podman user to /etc/subuid
For example, 
echo bockjoo:100000:65536 >> /etc/subuid
Add the podgman group to /etc/subgid
For example,
echo bockjoo:100000:65536 >> /etc/subgid

[ ~]# echo bockjoo:100000:65536 >> /etc/subgid
[ ~]# echo bockjoo:100000:65536 >> /etc/subgid

-bash-4.2$ podman ps
-bash-4.2$ echo $?
0

One can use the usermod command to update subuid and subgid:
usermod --add-subuids 100000-65536 --add-subgids 100000-65536 bockjoo

<b>B. In Puppet, do something like: </b>
echo 15000 > /proc/sys/user/max_user_namespaces
usermod --add-subuids 100000-65536 --add-subgids 100000-65536 bockjoo

</pre>
### [2-4] Configure cgroup V2 support 
<pre>
<b>A. A description</b>
It allows the user to limit the amount of resources a rootless container can use.
<!--If the Linux distribution that you are running Podman on is enabled with cgroup V2 (Centos8/EL8/RHEL8),
you might need to change the default OCI Runtime.-->

<b>B. In Puppet, one can do something like</b> :
grep -q systemd.unified_cgroup_hierarchy=1 /proc/cmdline || \
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
systemctl reboot
</pre>
### [2-5] Container Storage Configuration
<pre>
<b>A long explanation</b>
If the main podman user home directory is mounted on an NFS, it is not capable of overlayfs.
There is a multiple way of doing this, i.e., preparing the container storage to store images, etc.
One is configuring graphroot parameter in ~/.config/containers/storage.conf
Another is symlinking ~/.local/share/containers to a disk is local to the machine.
Here's the second method assuming the main podman user is bockjoo:
mkdir /opt/cms
chown -R bockjoo:avery /opt/cms
if [ -d ~/.local/share/containers ] ; then
   if [ -L ~/.local/share/containers ] ;  then
      ls -al ~/.local/share/containers
      echo Warning ~/.local/share/containers is already a symlink
   else
      cp -pR ~/.local/share/containers /opt/cms/.local/share
      rm -rf ~/.local/share/containers
      cd ~/.local/share
      ln -s /opt/cms/.local/share/containers 
   fi
else
   mkdir -p /opt/cms/.local/share/containers
   chown -R bockjoo:avery /opt/cms
   su - bockjoo -c "mkdir -p ~/.local/share ; cd ~/.local/share ; ln -s /opt/cms/.local/share/containers ;"   
fi

If the machine is just prepared, one will just need to execute:

mkdir -p /opt/cms/.local/share/containers
chown -R bockjoo:avery /opt/cms
su - bockjoo -c "mkdir -p ~/.local/share ; cd ~/.local/share ; ln -s /opt/cms/.local/share/containers ;"   
</pre>
<pre>
<b>B. In Puppet, one can do something like</b> :
mkdir /opt/cms ; chown bockjoo:avery /opt/cms
</pre>
### [2-6] Information on Users who read or write through XRootD
<pre>
<b>A. Who has access to the /cmsuf through XRootD</b>
These XRootD users are defined in /etc/xrootd/voms-mapfile and /etc/grid-security/grid-mapfile.
Correspondg users in the username space need to be created to access the Lustre filesystem
<b>B. For Puppet, if needed, you might want to refer to</b>
/cmsuf/t2/operations/podman/xrootd/etc/grid-security/grid-mapfile
/cmsuf/t2/operations/podman/xrootd/etc/xrootd/voms-mapfile

</pre>
### [2-7] Preparing the podman accounts in the user namespace
<pre>
<b> A. Explanation </b>
It appears the users in the podman user namespace need to exist for the users in the
podman XRootD container to read and write files in Lustre, /cmsuf
<b> B. In Puppet, one can do something like:</b>
See /cmsuf/t2/operations/ftl_create_podman_users.sh

For the whole account choreographing, see /cmsuf/t2/operations/ftl_create_podman_users_on_podman_machines.sh
</pre>
### [2-8] The XRootD backend storage directory ownership and permission
<pre>
<b>A. Explanation </b>
/cmsuf/data/store has various subdirectories owned by the users accessing the XRootD.
We will use /cmsuf/podman/data/store instead for the podman XRootD as they need the podman user ownership.
So, /cmsuf/podman needs to be owned by the user who is running podman container (bockjoo:avery)
<b>B. One time operation </b>
mkdir -p /cmsuf/podman
chown bockjoo:avery /cmsuf/podman
</pre>
### [2-9] Additional Ports for the transition from the regular XRootD to the podman XRootD:
<pre>
<b>A. Explanation</b>
During the transition, both the regular XRootD to the podman XRootD need to coexist.
<b>B. Puppet </b>
ports 1095 and 3121 need to be open
</pre>
### [2-10] Persistent User to prevent the issue [6-2]
<pre>
<b>A. Explanation</b>
After logging out from machine, Podman containers are stopped for some users. To prevent that, enable lingering for users running containers.
<b>B. Puppet </b>
loginctl enable-linger bockjoo
</pre>

## [3] T2 Actions: Unprivileged
### [3-1] Checking the podman requirements
<pre>
[3-1-1] OS > rhel7 ?
uname -a | grep "el8\|el9"

[3-1-2] Do podman packages exist?
rpm_qa=$(rpm -qa)
for p in podman podman-compose buildah fuse-overlayfs ; do
    printf "$rpm_qa\n" | grep $p
done

[3-1-3] It should support cgroup2fs
stat -fc %T /sys/fs/cgroup/

[3-1-4] There should be non-zero username space
cat /proc/sys/user/max_user_namespaces

[3-1-5] podman user is added to /etc/subuid and /etc/subgid for the user namespace
grep bockjoo /etc/subuid
grep bockjoo /etc/subgid

[3-1-6] Does /opt/cms exist and is it owned by bockjoo:avery?
stat -c %U:%G /opt/cms

[3-1-7] Do podman users/groups exist?

for u in $(cat /cmsuf/t2/operations/ftl_create_podman_users.sh | grep useradd | awk '{print $NF}') ; do
   getent passwd $u
done

for g in $(cat /cmsuf/t2/operations/ftl_create_podman_users.sh | grep groupadd | cut -d\# -f1 | awk '{print $NF}') ; do
   getent group $g
done

[3-1-8] Is /cmsuf/podman owned by bockjoo:avery?
stat -c %U:%G /cmsuf/podman

</pre>
### [3-2] Building the XRootD Images
<pre>
cd /cmsuf/t2/operations/podman/xrootd/server
# Cleanup image: 
podman image rm $(podman images | grep xrootd_server | awk '{print $3}')
# Building
buildah bud -f Dockerfile.Systemd -t xrootd_server
</pre>
### [3-3] Starting the XRootD Server in the Podman Container
#### [3-1] One time setup in each podman machine
<pre>
# XRootD configurations
if [ ! -d /opt/cms/etc/xrootd ] ; then
   mkdir -p /opt/cms/etc/xrootd
   xrootd_configs="xrootd-clustered.cfg Authfile ban-robots.txt macaroon-secret robots.txt"
   xrootd_configs="$xrootd_configs scitokens.cfg scitokens_mapfile_wlcg.json scitokens-map.json voms-mapfile" 
   for f in $xrootd_configs ; do
       /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/xrootd/$f /opt/cms/etc/xrootd/
   done
   if [ $(/bin/hostname -s | grep -q 2 ; echo $?) -eq 0 ] ; then
      /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/xrootd/xrootd-clustered.cfg.redirector /opt/cms/etc/xrootd/xrootd-clustered.cfg
      if [ $(grep ^all.manager /opt/cms/etc/xrootd/xrootd-clustered.cfg | awk '{print $NF}' | cut -d. -f1) == $(/bin/hostname -s) ] ; then
         echo OK redirector
      else
         echo ERROR the redirector needs to be reconfigured.
         vi /opt/cms/etc/xrootd/xrootd-clustered.cfg
      fi
   fi
fi

# GSI
if [ ! -d /opt/cms/etc/grid-security ] ; then
   mkdir -p /opt/cms/etc/grid-security/xrd
   mapfiles="ban-mapfile grid-mapfile voms-mapfile voms-mapfile2"
   for f in $mapfiles ; do
       /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/grid-security/$f /opt/cms/etc/grid-security/
   done
   certs="hostcert_$(/bin/hostname -s).pem hostkey_$(/bin/hostname -s).pem"
   for f in $certs ; do
       /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/grid-security/$f /opt/cms/etc/grid-security/
   done
fi

# For the FQHN
if [ ! -d /opt/cms/etc/sysconfig ] ; then
   mkdir -p /opt/cms/etc/sysconfig
   echo HOSTNAME=$(/bin/hostname -s).rc.ufl.edu > /opt/cms/etc/sysconfig/xrootd
fi

# For the FQHN
if [ ! -d /opt/cms/etc/systemd/system/systemd-hostnamed.service.d ] ; then
   mkdir -p /opt/cms/etc/systemd/system/systemd-hostnamed.service.d ] ; then
   
   for f in cmsd@.service xrootd-privileged@.service ; do
       /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/systemd/system/$f /opt/cms/etc/systemd/system
   done
   
   /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/systemd/system/systemd-hostnamed.service.d/override.conf /opt/cms/etc/systemd/system/systemd-hostnamed.service.d
fi

# For the FQHN
if [ ! -f /opt/cms/etc/hosts ] ; then
   /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/hosts /opt/cms/etc/hosts
fi

# For the FQHN
if [ ! -f /opt/cms/etc/hostname ] ; then
   echo $(/bin/hostname -s).rc.ufl.edu > /opt/cms/etc/hostname
fi

# selinux
if [ ! -f /opt/cms/etc/selinux/config ] ; then
   /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/selinux/config /opt/cms/etc/selinux/config
   
fi

# File mapping /cmsuf/podman/data/store -> /store ( storage_podman.xml and storage.xml )
# To switch the mapping for /cmsuf/data/store -> /store, use storage_regular.xml
if [ ! -d /opt/cms/etc/SITECONF/local/PhEDEx ] ; then
   mkdir -p /opt/cms/etc/SITECONF/local/PhEDEx
   storage_maps="storage_opt.xml storage_regular.xml storage_podman.xml storage.xml"
   for f in $storage_maps ; do
      /bin/cp /cmsuf/t2/operations/podman/xrootd/etc/SITECONF/local/PhEDEx/$f /opt/cms/etc/SITECONF/local/PhEDEx/
   done
fi

container_id=$(podman run -d --rm --name xrootd_server \
               --cgroup-manager=cgroupfs --tmpfs /tmp \
               --tmpfs /run \
               -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
               -v /opt/cms/etc/xrootd/:/etc/xrootd/:ro \
               -v /opt/cms/etc/grid-security/grid-mapfile:/etc/grid-security/grid-mapfile:ro \
               -v /opt/cms/etc/grid-security/ban-mapfile:/etc/grid-security/ban-mapfile:ro \
               -v /opt/cms/etc/grid-security/voms-mapfile:/etc/grid-security/voms-mapfile:ro \
               -v /opt/cms/etc/grid-security/hostcert.pem:/etc/grid-security/hostcert.pem:ro \
               -v /opt/cms/etc/grid-security/hostkey.pem:/etc/grid-security/hostkey.pem:ro \
               -v /opt/cms/etc/grid-security/xrd/:/etc/grid-security/xrd:rw \
               -v /opt/cms/etc/sysconfig/xrootd:/etc/sysconfig/xrootd:ro \
               -v /opt/cms/etc/systemd/system/xrootd-privileged@.service:/etc/systemd/system/xrootd-privileged@.service:ro \
               -v /opt/cms/etc/systemd/system/cmsd@.service:/etc/systemd/system/cmsd@.service:ro \
               -v /opt/cms/etc/hosts:/etc/hosts:rw \
               -v /opt/cms/etc/hostname:/etc/hostname:rw \
               -v /opt/cms/etc/systemd/system/systemd-hostnamed.service.d/:/etc/systemd/system/systemd-hostnamed.service.d/:ro \
               -v /opt/cms/etc/selinux/config:/etc/selinux/config:rw \
               -v /opt/cms/podman/:/opt/cms/podman/:rw \
               -v /opt/cms/store/:/opt/cms/store/:rw \
               -v /opt/cms/etc/SITECONF/:/etc/SITECONF/:ro \
               -v /cmsuf/:/cmsuf/:rw \
               --systemd=true --network=host --cgroup-manager=systemd localhost/xrootd_server:latest)

# xrd cert/key pair preparation
podman exec -it ${container_id} cp /etc/grid-security/hostcert.pem /etc/grid-security/xrd/xrdcert.pem
podman exec -it ${container_id} cp /etc/grid-security/hostkey.pem /etc/grid-security/xrd/xrdkey.pem
podman exec -it ${container_id} chown xrootd:xrootd /etc/grid-security/xrd/xrdcert.pem
podman exec -it ${container_id} chown xrootd:xrootd /etc/grid-security/xrd/xrdkey.pem

# xrootd/cmsd servers need to be restarted after xrd cert/key pair preparation
podman exec -it ${container_id} systemctl restart xrootd-privileged@clustered.service
podman exec -it ${container_id} systemctl restart cmsd@clustered.service
# /cmsuf/t2/operations/ftl_create_podman_xrootd_storage_directory.sh: This creates the following directories
# with the proper mode and the proper ownership inside the container
# <FONT color='red'><b>The script will fail if the accounts in the Section [2-7] do not exist</b></FONT>
</pre>
## [4] Tests with Possible Use Cases
### Inpections with the container
<pre>
podman exec -it ${container_id} ps auxwww # To check the xrootd/cmsd processes are running
podman exec -it ${container_id} /bin/bash # To inpect the container interactively
</pre>
### [4-1] check with xrdmapc and xrdfs behavior, a.k.a. AAA in CMS
<pre>
xrdmapc --list all cmspodman1.rc.ufl.edu:1094 
xrdmapc --list all cmspodman2.rc.ufl.edu:1094 

xrdfs cmspodman1.rc.ufl.edu:1094 query config version
xrdfs cmspodman2.rc.ufl.edu:1094 query config version

xrdfs cmspodman2.rc.ufl.edu:1094 ls -l /store
xrdfs cmspodman2.rc.ufl.edu:1094 locate /store/mc/SAM/GenericTTbar/AODSIM/CMSSW_9_2_6_91X_mcRun1_realistic_v2-v1/00000/A64CCCF2-5C76-E711-B359-0CC47A78A3F8.root
</pre>

### [4-2] Test for the /store/user read/write (user bockjoo)
#### xrdcp upload
<pre>

# For some reason, there was the permission denied error initially ( Morning of Aug 21, but it became successful Afternoon of Aug 21 )

[bockjoo@cms ~]$ xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list
[2023-08-21 15:03:36.307318 -0400][Info   ][AsyncSock         ] [cmspodman2.rc.ufl.edu:1094.0] TLS hand-shake done.
[2023-08-21 15:03:38.734174 -0400][Info   ][AsyncSock         ] [cmspodman1.rc.ufl.edu:1094.0] TLS hand-shake done.
[26.51kB/26.51kB][100%][==================================================][26.5[26.51kB/26.51kB][100%][==================================================][26.51kB/s]  
[bockjoo@cms ~]$ ls /cmsuf/podman/data/store/user/bockjoo/
sitedb.list  test_21AUG2023.txt
[bockjoo@cms ~]$ xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/subdir/sitedb.list
[2023-08-21 15:06:20.251453 -0400][Info   ][AsyncSock         ] [cmspodman2.rc.ufl.edu:1094.0] TLS hand-shake done.
[2023-08-21 15:06:22.567718 -0400][Info   ][AsyncSock         ] [cmspodman1.rc.ufl.edu:1094.0] TLS hand-shake done.
[26.51kB/26.51kB][100%][==================================================][26.5[26.51kB/26.51kB][100%][==================================================][26.51kB/s]  
[bockjoo@cmspodman1 ~]$ ls -al /cmsuf/podman/data/store/user/bockjoo/subdir/sitedb.list 
-rw-r--r-- 1 podman podman 27150 Aug 21 15:06 /cmsuf/podman/data/store/user/bockjoo/subdir/sitedb.list
</pre>


#### xrdcp download
<pre>
[bockjoo@cms ~]$ ls -al  /cmsuf/podman/data/store/user/bockjoo/subdir1/sitedb.list 
-rw-r--r-- 1 563326 563751 27150 Aug 21 19:19 /cmsuf/podman/data/store/user/bockjoo/subdir1/sitedb.list
[bockjoo@cms ~]$ xrdcp -d 1 -f root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list ./
[2023-08-21 19:20:53.262015 -0400][Info   ][AsyncSock         ] [cmspodman2.rc.ufl.edu:1094.0] TLS hand-shake done.
[2023-08-21 19:20:55.666187 -0400][Info   ][AsyncSock         ] [cmspodman1.rc.ufl.edu:1094.0] TLS hand-shake done.
[26.51kB/26.51kB][100%][==================================================][6.628kB/s]  
[bockjoo@cms ~]$ ls -al sitedb.list 
-rwxr-xr-x 1 bockjoo avery 27150 Aug 21 19:20 sitedb.list
</pre>

#### gfal-copy upload and download
<pre>
[bockjoo@cms ~]$ /usr/bin/python3 $(which gfal-copy) -f file://`pwd`/sitedb.list davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/gfal_copy/sitedb.list
Copying file:///home/bockjoo/sitedb.list   [DONE]  after 0s                                                  
[bockjoo@cms ~]$ ls -al /cmsuf/podman/data/store/user/bockjoo/gfal_copy/sitedb.list 
-rw-r--r-- 1 563326 563751 27150 Aug 21 19:49 /cmsuf/podman/data/store/user/bockjoo/gfal_copy/sitedb.list

[bockjoo@cms ~]$ /usr/bin/python3 $(which gfal-copy) -f davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/gfal_copy/sitedb.list ./
Copying davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/gfal_copy/sitedb.list   [DONE]  after 0s       
[bockjoo@cms ~]$ ls -al sitedb.list 
-rwxr-xr-x 1 bockjoo avery 27150 Aug 21 20:02 sitedb.list

</pre>

#### token xrdcp upload/download : https://twiki.cern.ch/twiki/bin/view/CMSPublic/XRootDAndTokens
##### Token Handling
<pre>
 eval `oidc-agent`
 oidc-add -l
 oidc-add bockjoo_xrd
 export BEARER_TOKEN=$(oidc-token --scope=offline_access --scope=storage.read:/ --time=3600 bockjoo_xrd)
 httokendecode
 </pre>
##### XRootD Token Configuration
<pre>
# add the following lines to appropriate places in /opt/cms/etc/xrootd/xrootd-clustered.cfg
xrd.tls /etc/grid-security/xrd/xrdcert.pem /etc/grid-security/xrd/xrdkey.pem
xrd.tlsca certdir /etc/grid-security/certificates
xrootd.tls capable all
sec.protocol /usr/lib64 ztn
</pre>
##### xrdmapc
<pre>
[bockjoo@cms ~]$ export XrdSecPROTOCOL="ztn,unix" ; xrdmapc --list all cmspodman1.rc.ufl.edu:1094
0**** cmspodman1.rc.ufl.edu:1094
      Srv cmspodman1.ufhpc:1094
[bockjoo@cms ~]$ export XrdSecPROTOCOL="ztn,unix" ; xrdmapc --list all cmspodman2.rc.ufl.edu:1094
0**** cmspodman2.rc.ufl.edu:1094
      Srv cmspodman1.rc.ufl.edu:1094

</pre>
##### xrdcp download
mdYHMS=$(date +%b%d%Y+%H+%M+%S)
xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS
( 
   eval `oidc-agent`
   oidc-add bockjoo_xrd
   export BEARER_TOKEN=$(oidc-token --scope=offline_access --scope=storage.read:/ --time=3600 bockjoo_xrd)
   export XrdSecPROTOCOL="ztn,unix"
   xrdcp -d 1 -f root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS?authz=Bearer%20$BEARER_TOKEN ./
)
ls -al sitedb.list_$mdYHMS 
/usr/bin/python3 $(which gfal-ls) davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS
/usr/bin/python3 $(which gfal-rm) davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS
rm -f sitedb.list_$mdYHMS 


##### gfal-copy download (upload does not work as I can not get the storage.write:/ scope)
<pre>
mdYHMS=$(date +%b%d%Y+%H+%M+%S)
xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS
( 
   eval `oidc-agent`
   oidc-add bockjoo_xrd
   export BEARER_TOKEN=$(oidc-token --scope=offline_access --scope=storage.read:/ --time=3600 bockjoo_xrd)
   export XrdSecPROTOCOL="ztn,unix"
   /usr/bin/python3 $(which gfal-copy) -f davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS ./
ls -al sitedb.list_$mdYHMS
)

/usr/bin/python3 $(which gfal-ls) davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS
/usr/bin/python3 $(which gfal-rm) davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list_$mdYHMS
rm -f sitedb.list_$mdYHMS 
</pre>
### [4-3] Role Based Tests (cmsprod)
#### Change the grid-mapfile for the role
<pre>
sed -i 's|"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim" bockjoo|#"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim" bockjoo|' /opt/cms/etc/grid-security/grid-mapfile
grep Bockjoo /opt/cms/etc/grid-security/grid-mapfile
</pre>

#### Restart the Container, Thus XRootD
<pre>
podman stop $container_id
container_id=$(podman run -d --rm --name xrootd_server \
               --cgroup-manager=cgroupfs --tmpfs /tmp \
               --tmpfs /run \
               -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
               -v /opt/cms/etc/xrootd/:/etc/xrootd/:ro \
               -v /opt/cms/etc/grid-security/grid-mapfile:/etc/grid-security/grid-mapfile:ro \
               -v /opt/cms/etc/grid-security/ban-mapfile:/etc/grid-security/ban-mapfile:ro \
               -v /opt/cms/etc/grid-security/voms-mapfile:/etc/grid-security/voms-mapfile:ro \
               -v /opt/cms/etc/grid-security/hostcert.pem:/etc/grid-security/hostcert.pem:ro \
               -v /opt/cms/etc/grid-security/hostkey.pem:/etc/grid-security/hostkey.pem:ro \
               -v /opt/cms/etc/grid-security/xrd/:/etc/grid-security/xrd:rw \
               -v /opt/cms/etc/sysconfig/xrootd:/etc/sysconfig/xrootd:ro \
               -v /opt/cms/etc/systemd/system/xrootd-privileged@.service:/etc/systemd/system/xrootd-privileged@.service:ro \
               -v /opt/cms/etc/systemd/system/cmsd@.service:/etc/systemd/system/cmsd@.service:ro \
               -v /opt/cms/etc/hosts:/etc/hosts:rw \
               -v /opt/cms/etc/hostname:/etc/hostname:rw \
               -v /opt/cms/etc/systemd/system/systemd-hostnamed.service.d/:/etc/systemd/system/systemd-hostnamed.service.d/:ro \
               -v /opt/cms/etc/selinux/config:/etc/selinux/config:rw \
               -v /opt/cms/podman/:/opt/cms/podman/:rw \
               -v /opt/cms/store/:/opt/cms/store/:rw \
               -v /opt/cms/etc/SITECONF/:/etc/SITECONF/:ro \
               -v /cmsuf/:/cmsuf/:rw \
               --systemd=true --network=host --cgroup-manager=systemd localhost/xrootd_server:latest)
#### Check the proxy has phedex or production role for cmsprod
<pre>
source /cvmfs/oasis.opensciencegrid.org/osg-software/osg-wn-client/current/el$(source /cvmfs/cms.cern.ch/cmsset_default.sh ; cmsos| cut -d_ -f1 | sed 's#[a-z]\|[A-Z]##g')-x86_64/setup.sh
export X509_USER_PROXY=$X509_USER_PROXY_NONCMS
export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
voms-proxy-info -fqan
</pre>
#### xrdmapc and xrdfs with the role
<pre>
(
   export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
   xrdfs cmspodman1.rc.ufl.edu:1094 query config version
   xrdfs cmspodman2.rc.ufl.edu:1094 query config version
   xrdfs cmspodman1.rc.ufl.edu:1094 query config version
   xrdfs cmspodman2.rc.ufl.edu:1094 query config version
   xrdfs cmspodman2.rc.ufl.edu:1094 ls /store
)
# on the podman machine with the xrootd server
podman exec -it $container_id tail -50 /var/log/xrootd/clustered/xrootd.log | grep bockjoo | grep "login as"
       
(    
       export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy ; 
       xrdfs cmspodman1.rc.ufl.edu:1094 locate  /store/mc/SAM/GenericTTbar/AODSIM/CMSSW_9_2_6_91X_mcRun1_realistic_v2-v1/00000/A64CCCF2-5C76-E711-B359-0CC47A78A3F8.root ; 
)
</pre>

#### xrdcp upload and remove the file
<pre>
/home/bockjoo/.cmssoft/phedex_proxy is mapped to cmsprod account in the container
</pre>
<pre>
# on the client
(
   mdYHMS=$(date +%b%d%Y+%H+%M+%S)
   export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
   xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_$mdYHMS
   ls -al /cmsuf/podman/data/store/user/rucio/bockjoo/sitedb.list_$mdYHMS
   /usr/bin/python3 $(which gfal-ls) davs://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_$mdYHMS
)

# on the podman machine
mdYHMS=Aug242023+21+18+01
podman exec -it $container_id ls -al /cmsuf/podman/data/store/user/rucio/bockjoo/sitedb.list_$mdYHMS

# on the client
(
   mdYHMS=Aug242023+21+18+01
   export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
   /usr/bin/python3 $(which gfal-rm) davs://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_$mdYHMS

)



[bockjoo@cms ~]$ # on the client
[bockjoo@cms ~]$ (
>    mdYHMS=$(date +%b%d%Y+%H+%M+%S)
>    export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
>    xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_$mdYHMS
>    ls -al /cmsuf/podman/data/store/user/rucio/bockjoo/sitedb.list_$mdYHMS
>    /usr/bin/python3 $(which gfal-ls) davs://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_$mdYHMS
> )
[2023-08-24 21:18:37.408742 -0400][Info   ][AsyncSock         ] [cmspodman2.rc.ufl.edu:1094.0] TLS hand-shake done.
[2023-08-24 21:18:39.835221 -0400][Info   ][AsyncSock         ] [cmspodman1.rc.ufl.edu:1094.0] TLS hand-shake done.
[26.51kB/26.51kB][100%][==================================================][26.51kB/s]  
-rw-r--r-- 1 559474 563752 27150 Aug 24 21:18 /cmsuf/podman/data/store/user/rucio/bockjoo/sitedb.list_Aug242023+21+18+37
davs://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_Aug242023+21+18+37




[bockjoo@cmspodman1 ~]$ # on the podman machine
[bockjoo@cmspodman1 ~]$ mdYHMS=Aug242023+21+18+01
[bockjoo@cmspodman1 ~]$ podman exec -it $container_id ls -al /cmsuf/podman/data/store/user/rucio/bockjoo/sitedb.list_$mdYHMS
-rw-r--r-- 1 cmsprod cmsdata 27150 Aug 25 01:18 /cmsuf/podman/data/store/user/rucio/bockjoo/sitedb.list_Aug242023+21+18+01



[bockjoo@cms ~]$ # on the client
[bockjoo@cms ~]$ (
>    mdYHMS=Aug242023+21+18+01
>    export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
>    /usr/bin/python3 $(which gfal-rm) davs://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_$mdYHMS
> 
> )
davs://cmspodman2.rc.ufl.edu:1094//store/user/rucio/bockjoo/sitedb.list_Aug242023+21+18+01      DELETED


</pre>
### [4-4] Analysis Scenario
#### Change the grid-mapfile for the role cms0001 on cmspodman1 and cmspodman2
<pre>
grep -q ^"#\"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim\"" /opt/cms/etc/grid-security/grid-mapfile
if [ $? -eq 0 ] ; then
   echo INFO Bockjoo Kim DN is commented out in the grid-mapfile
else
   sed -i 's|"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim" bockjoo|#"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim" bockjoo|' /opt/cms/etc/grid-security/grid-mapfile
fi
grep Bockjoo /opt/cms/etc/grid-security/grid-mapfile
</pre>
#### Restart the Container, Thus XRootD
<pre>
podman stop $container_id
container_id=$(podman run -d --rm --name xrootd_server \
               --cgroup-manager=cgroupfs --tmpfs /tmp \
               --tmpfs /run \
               -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
               -v /opt/cms/etc/xrootd/:/etc/xrootd/:ro \
               -v /opt/cms/etc/grid-security/grid-mapfile:/etc/grid-security/grid-mapfile:ro \
               -v /opt/cms/etc/grid-security/ban-mapfile:/etc/grid-security/ban-mapfile:ro \
               -v /opt/cms/etc/grid-security/voms-mapfile:/etc/grid-security/voms-mapfile:ro \
               -v /opt/cms/etc/grid-security/hostcert.pem:/etc/grid-security/hostcert.pem:ro \
               -v /opt/cms/etc/grid-security/hostkey.pem:/etc/grid-security/hostkey.pem:ro \
               -v /opt/cms/etc/grid-security/xrd/:/etc/grid-security/xrd:rw \
               -v /opt/cms/etc/sysconfig/xrootd:/etc/sysconfig/xrootd:ro \
               -v /opt/cms/etc/systemd/system/xrootd-privileged@.service:/etc/systemd/system/xrootd-privileged@.service:ro \
               -v /opt/cms/etc/systemd/system/cmsd@.service:/etc/systemd/system/cmsd@.service:ro \
               -v /opt/cms/etc/hosts:/etc/hosts:rw \
               -v /opt/cms/etc/hostname:/etc/hostname:rw \
               -v /opt/cms/etc/systemd/system/systemd-hostnamed.service.d/:/etc/systemd/system/systemd-hostnamed.service.d/:ro \
               -v /opt/cms/etc/selinux/config:/etc/selinux/config:rw \
               -v /opt/cms/podman/:/opt/cms/podman/:rw \
               -v /opt/cms/store/:/opt/cms/store/:rw \
               -v /opt/cms/etc/SITECONF/:/etc/SITECONF/:ro \
               -v /cmsuf/:/cmsuf/:rw \
               --systemd=true --network=host --cgroup-manager=systemd localhost/xrootd_server:latest)

</pre>
#### Check the role cms0001
<pre>
# on the client
xrdfs cmspodman2.rc.ufl.edu:1094 ls /store/user/bockjoo (

# on the podman machine with the xrootd server
podman exec -it $container_id tail -50 /var/log/xrootd/clustered/xrootd.log | grep bockjoo | grep "login as"

[bockjoo@cms ~]$ xrdfs cmspodman2.rc.ufl.edu:1094 ls /store/user/bockjoo
/store/user/bockjoo/gfal_copy
/store/user/bockjoo/sitedb.list_Aug242023+15+55+24
/store/user/bockjoo/sitedb.list_Aug242023+15+57+16
/store/user/bockjoo/sitedb.list_Aug242023+17+29+42
/store/user/bockjoo/sitedb.list_Aug242023+17+35+15
/store/user/bockjoo/sitedb.list_Aug242023+18+37+11
/store/user/bockjoo/sitedb.list_Aug242023+18+56+52
/store/user/bockjoo/sitedb.list_Aug242023+19+12+28
/store/user/bockjoo/sitedb.list_Aug242023+19+14+08
/store/user/bockjoo/subdir
/store/user/bockjoo/subdir1

[bockjoo@cmspodman1 ~]$ podman exec -it $container_id tail -50 /var/log/xrootd/clustered/xrootd.log | grep bockjoo | grep "login as"
230825 02:46:44 062 XrootdXeq: bockjoo.4066878:39@cms-data pvt IPv4 TLSv1.3 login as cms0001

</pre>
#### xrdcp upload and remove the file
<pre>
# client
mdYHMS=$(date +%b%d%Y+%H+%M+%S)
xrdcp -d 1 -f file://`pwd`/sitedb.list root://cmspodman2.rc.ufl.edu:1094//store/temp/user/cms0001.$mdYHMS/sitedb.list_$mdYHMS
ls -al /cmsuf/podman/data/store/temp/user/cms0001.$mdYHMS/sitedb.list_$mdYHMS

# podman
mdYHMS=Aug242023+22+51+10
podman exec -it $container_id  ls -al /cmsuf/podman/data/store/temp/user/cms0001.${mdYHMS}/sitedb.list_${mdYHMS}


/usr/bin/python3 $(which gfal-ls) davs://cmspodman2.rc.ufl.edu:1094//store/temp/user/cms0001.$mdYHMS/sitedb.list_$mdYHMS
/usr/bin/python3 $(which gfal-rm) davs://cmspodman2.rc.ufl.edu:1094//store/temp/user/cms0001.$mdYHMS/sitedb.list_$mdYHMS

</pre>

### [4-5] Production Scenario
#### [4-5-1] SAM WebDAV Tests
##### Change the grid-mapfile for the role
<pre>
sed -i 's|#"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim" bockjoo|"/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=bockjoo/CN=556538/CN=Bockjoo Kim" bockjoo|' /opt/cms/etc/grid-security/grid-mapfile
grep Bockjoo /opt/cms/etc/grid-security/grid-mapfile
</pre>

##### Restart the Container, Thus XRootD
<pre>
podman stop $container_id
container_id=$(podman run -d --rm --name xrootd_server \
               --cgroup-manager=cgroupfs --tmpfs /tmp \
               --tmpfs /run \
               -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
               -v /opt/cms/etc/xrootd/:/etc/xrootd/:ro \
               -v /opt/cms/etc/grid-security/grid-mapfile:/etc/grid-security/grid-mapfile:ro \
               -v /opt/cms/etc/grid-security/ban-mapfile:/etc/grid-security/ban-mapfile:ro \
               -v /opt/cms/etc/grid-security/voms-mapfile:/etc/grid-security/voms-mapfile:ro \
               -v /opt/cms/etc/grid-security/hostcert.pem:/etc/grid-security/hostcert.pem:ro \
               -v /opt/cms/etc/grid-security/hostkey.pem:/etc/grid-security/hostkey.pem:ro \
               -v /opt/cms/etc/grid-security/xrd/:/etc/grid-security/xrd:rw \
               -v /opt/cms/etc/sysconfig/xrootd:/etc/sysconfig/xrootd:ro \
               -v /opt/cms/etc/systemd/system/xrootd-privileged@.service:/etc/systemd/system/xrootd-privileged@.service:ro \
               -v /opt/cms/etc/systemd/system/cmsd@.service:/etc/systemd/system/cmsd@.service:ro \
               -v /opt/cms/etc/hosts:/etc/hosts:rw \
               -v /opt/cms/etc/hostname:/etc/hostname:rw \
               -v /opt/cms/etc/systemd/system/systemd-hostnamed.service.d/:/etc/systemd/system/systemd-hostnamed.service.d/:ro \
               -v /opt/cms/etc/selinux/config:/etc/selinux/config:rw \
               -v /opt/cms/podman/:/opt/cms/podman/:rw \
               -v /opt/cms/store/:/opt/cms/store/:rw \
               -v /opt/cms/etc/SITECONF/:/etc/SITECONF/:ro \
               -v /cmsuf/:/cmsuf/:rw \
               --systemd=true --network=host --cgroup-manager=systemd localhost/xrootd_server:latest)

</pre>
##### Check bockjoo login as bockjoo
<pre>
# on client
#(
#   export X509_USER_PROXY=/home/bockjoo/.cmssoft/phedex_proxy
   xrdfs cmspodman2.rc.ufl.edu:1094 ls /store/user/bockjoo
#)

# on the podman machine with the xrootd server
podman exec -it $container_id tail -50 /var/log/xrootd/clustered/xrootd.log | grep bockjoo | grep "login as"
</pre>

<pre>
export X509_CERT_DIR=/cvmfs/cms.cern.ch/grid/etc/grid-security/certificates
export X509_USER_PROXY=/home/bockjoo/.cmsuser.proxy
export X509_USER_PROXY_NONCMS=/home/bockjoo/.griduser.proxy
export SAME_SENSOR_HOME=$HOME/cmssam/SiteTests/testjob
export PYTHONPATH=$PYTHONPATH:$SAME_SENSOR_HOME/../SRMv2/tests/nap
# The reference test
(
   cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
   host=cmsio2.rc.ufl.edu   
   $SAME_SENSOR_HOME/../SE/se_webdav.py -H ${host} -E ${host}:1094 -X $X509_USER_PROXY -N $X509_USER_PROXY_NONCMS -T RD3PCP /store/mc/SAM/  -T WRDEL3PCP /store/user/bockjoo -C /dev/null  > /opt/cms/services/T2/ops/webdav/runSAMWebDAV.$(echo $host | cut -d. -f1).out 2>&1 
)

# The WebDAV SAM test on the podman
(
   cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
   host=cmspodman2.rc.ufl.edu   
   $SAME_SENSOR_HOME/../SE/se_webdav_cmspodman2.py -H ${host} -E ${host}:1094 -X $X509_USER_PROXY -N $X509_USER_PROXY_NONCMS -T RD3PCP /store/mc/SAM/  -T WRDEL3PCP /store/user/bockjoo -C /dev/null  > /opt/cms/services/T2/ops/webdav/runSAMWebDAV.$(echo $host | cut -d. -f1).out 2>&1 
)
</pre>
#### [4-5-2] SAM XRootD Tests
##### Preparation for the xrootd python (maybe unnecessary)
<pre>
source /cvmfs/cms.cern.ch/el8_amd64_gcc12/external/cmake/3.25.2-ba1804546854f5a6aa1f118c2f8f4439/etc/profile.d/init.sh
cd ~/bin
ln -sf $(which cmake) cmake3
pip install --upgrade xrootd
</pre>     
##### Setting up latest SAM test package
<pre>
cd /opt/cms/services/T2/ops/
git clone https://gitlab.cern.ch/etf/cmssam
cd cmssam/
cd SiteTests/SRMv2/tests/
git clone  https://gitlab.cern.ch:8443/etf/nap
</pre>
##### Running the latest WebDAV SAM tests
<pre>
export X509_CERT_DIR=/cvmfs/cms.cern.ch/grid/etc/grid-security/certificates
export X509_USER_PROXY=/home/bockjoo/.cmsuser.proxy
export X509_USER_PROXY_NONCMS=/home/bockjoo/.griduser.proxy
export SAME_SENSOR_HOME=/opt/cms/services/T2/ops/cmssam/SiteTests/testjob

# Reference Test with a cmsio machine
(    
    cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
    host=cmsio3.rc.ufl.edu
    $SAME_SENSOR_HOME/../SE/se_webdav_uf.py -d --print-all -H ${host} -E ${host}:1094 -X $X509_USER_PROXY -N $X509_USER_PROXY_NONCMS -T RD3PCP /store/mc/SAM/  -T WRDEL3PCP /store/user/bockjoo// -C /dev/null  > /opt/cms/services/T2/ops/cmssam/se_webdav.$(echo $host | cut -d. -f1).out ; 
)

# Podman WebDAV test with a podman 
(    
    cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
    host=cmspodman2.rc.ufl.edu
    $SAME_SENSOR_HOME/../SE/se_webdav_uf_podman.py -d --print-all -H ${host} -E ${host}:1094 -X $X509_USER_PROXY -N $X509_USER_PROXY_NONCMS -T RD3PCP /store/mc/SAM/  -T WRDEL3PCP /store/user/bockjoo// -C /dev/null  > /opt/cms/services/T2/ops/cmssam/se_webdav.$(echo $host | cut -d. -f1).out ; 
)
</pre>
##### Running the latest WebDAV SAM tests with token
###### CMS Token Twiki 
<pre>
https://twiki.cern.ch/twiki/bin/view/CMSPublic/XRootDAndTokens
https://twiki.cern.ch/twiki/bin/viewauth/CMS/IAMTokens
</pre>
###### Setting up the Token
<pre>
eval `oidc-agent`
oidc-add bockjoo_xrd
#export BEARER_TOKEN=$(oidc-token --scope=offline_access --scope=storage.read:/store --scope=storage.modify:/store/temp/user --time=3600 bockjoo_xrd)
export BEARER_TOKEN=$(oidc-token --scope=offline_access --scope=storage.read:/store --time=3600 bockjoo_xrd)
</pre>

###### SAM WebDAV Test This will not work until I have the token with the write-scope
<pre>

# Reference
(
     cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
     host=cmsio3.rc.ufl.edu
     $SAME_SENSOR_HOME/../SE/se_webdav_uf.py -d --print-all -H ${host} -E ${host}:1094 -X $X509_USER_PROXY -N $X509_USER_PROXY_NONCMS -T RD3PCP /store/mc/SAM/  -T WRDEL3PCP /store/temp/user/bockjoo_sam// -I $BEARER_TOKEN -C /dev/null > /opt/cms/services/T2/ops/cmssam/se_webdav.$(echo $host | cut -d. -f1).out
)

# Podman WebDAV test with a podman 
(    
    cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
    host=cmspodman2.rc.ufl.edu
    $SAME_SENSOR_HOME/../SE/se_webdav_uf_podman.py -d --print-all -H ${host} -E ${host}:1094 -X $X509_USER_PROXY -N $X509_USER_PROXY_NONCMS -T RD3PCP /store/mc/SAM/  -T WRDEL3PCP /store/user/bockjoo// -I $BEARER_TOKEN -C /dev/null  > /opt/cms/services/T2/ops/cmssam/se_webdav.$(echo $host | cut -d. -f1).out ; 
)

# This runs fine.
</pre>
###### Running the latest XRootD SAM tests, fails, but I think it's due to 5.5.5 sever with 5.6.1 client
<pre>
export X509_CERT_DIR=/cvmfs/cms.cern.ch/grid/etc/grid-security/certificates
export X509_USER_PROXY=/home/bockjoo/.cmsuser.proxy
export X509_USER_PROXY_NONCMS=/home/bockjoo/.griduser.proxy
export SAME_SENSOR_HOME=/opt/cms/services/T2/ops/cmssam/SiteTests/testjob
export SAME_SENSOR_HOME=/cmsuf/t2/operations/opt/cms/services/T2/ops/cmssam/SiteTests/testjob
export PYTHONPATH=$PYTHONPATH:$SAME_SENSOR_HOME/../SRMv2/tests/nap
# The reference test
(
   cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
   host=cmsio3.rc.ufl.edu   
   $SAME_SENSOR_HOME/../SE/cmssam_xrootd_endpnt_uf.py -H ${host} -P 1094 -S T2_US_Florida -4 -C /dev/null -d --print-all > /opt/cms/services/T2/ops/cmssam/cmssam_xrootd_endpnt_uf.$(echo $host | cut -d. -f1).out
)

# The WebDAV SAM test on the podman
(
   cd $SAME_SENSOR_HOME/../SRMv2/tests/nap
   host=cmspodman2.rc.ufl.edu   
   $SAME_SENSOR_HOME/../SE/cmssam_xrootd_endpnt_uf.py -H ${host} -P 1094 -S T2_US_Florida -4 -C /dev/null -d --print-all > /opt/cms/services/T2/ops/cmssam/cmssam_xrootd_endpnt_uf.$(echo $host | cut -d. -f1).out
)

# This does not run with the xrootd server TLS and errs:
# XRootDStatus.code=110 "[FATAL] TLS error: resource temporarily unavailable"
</pre>
       
## [5] A Plan for the Migration from /cmsuf/data(accessible through the regular xrootd) to /cmsuf/podman/data(accessible through the contained xrootd)
<pre>
       
Within Florida, 
1 Turn cmsio2.rc.ufl.edu into a read-only XRootD
2 Configure cmsio machines for the XRootD podman containerization 
3 Two XRootD instances need to be coexist until we migrate all to the podman XRootD
4 ports 1095 and 3121 need to be open on the cmsio machines for the read-write podman XRootD

The overall picture of the migration will follow these steps:
1) would disable writes to your RSE/Lustre(T2_US_Florida) storage (A CMS thing)
2) you would copy all the data from /cmsuf/data/store to /cmsuf/podman/data/store
3) then update storage.json (PhEDEx/storage.xml and JobConfig/site-local-config.xml) with new /cmsuf/podman/data/store
4) re-enable writes to your RSE/Lustre storageT2_US_Florida)


</pre>
## [6] Troubleshooting
### [6-1] Users in the user namespace (high UID/GID users) should exist to read/write files to /cmsuf/podman/data/store/? area
<pre>

The user, bockjoo, inside the container is
       [bockjoo@cmspodman1 ~]$ podman exec -it ${container_id} id bockjoo
       uid=4575(bockjoo) gid=5000(avery) groups=5000(avery)
       
Starting uid/gid in the username space is 
       [bockjoo@cmspodman1 ~]$ grep bockjoo /etc/subuid | cut -d: -f2
       558752
       [bockjoo@cmspodman1 ~]$ grep bockjoo /etc/subgid | cut -d: -f2
       558752

The user uid/gid in the host for the contained user bockjoo would be:
       563326(=558752 + 4575 - 1)/563751(558752 + 5000 - 1)

The contained user, bockjoo, can see the CMS storage directory /cmsuf/podman/data/store:
       [bockjoo@cmspodman1 ~]$  podman exec -it ${container_id} su - bockjoo -c "ls /cmsuf/podman/data/store"
       backfill  local  temp  user
because 
       [bockjoo@cmspodman1 ~]$ stat -c %U:%G /cmsuf/podman/data/store/user/bockjoo
       podman:podman
The corresponding user, podman, in the host exist:
       [bockjoo@cmspodman1 ~]$ id podman
       uid=563326(podman) gid=563751(podman) groups=563751(podman)

On the other hand, the user, cmsprod, inside the container is
       [bockjoo@cmspodman1 ~]$ podman exec -it ${container_id} id cmsprod
       uid=723(cmsprod) gid=5001(cmsdata) groups=5001(cmsdata)
       
This user fails to see /cmsuf/podman/data/store/user/bockjoo:
       [bockjoo@cmspodman1 ~]$ podman exec -it ${container_id} su - cmsprod -c "ls /cmsuf/podman/data/store"
       ls: cannot access '/cmsuf/podman/data/store': Permission denied
       
This is because the corresponding user to the contained user, cmsprod, does not exist in the host:
The uig/gid of the contained user, cmsprod, in the host would be;
       559474(=558752 + 723 - 1)/563752(558752 + 5001 - 1)
but the uid does not exist in the host;
       [bockjoo@cmspodman1 ~]$ getent passwd 559474
       [bockjoo@cmspodman1 ~]$ echo $?
       2

<b>
We need the "mirror image" users, the hostside container users.
Thus, the need for [2-7] Preparing the podman accounts in the user namespace.
</b>
</pre>


### [6-2] Container Stops after Logout 
<pre>
Tried --restart always or nohup but didn't work
Does this issue exist on a non-VM host? The container in a physical machine did not show this issue.
</pre>

## [7] Testing in the production-like host
<pre>
       All of the road bocks above should be cleared
</pre>
## [References]
<pre>
[1] https://docs.podman.io/en/latest/
[2] https://docs.podman.io/en/latest/Tutorials.html
[3] https://github.com/containers/podman/blob/main/docs/tutorials/podman_tutorial.md
[4] https://podman.io/docs/installation
[5] https://github.com/containers/podman/blob/main/cni/README.md
[6] https://access.redhat.com/solutions/6161832
[7] https://github.com/containers/podman/blob/main/test/README.md
[8] https://github.com/containers/podman/blob/main/README.md#commands
[9] https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md
[10] https://opensource.com/article/19/2/how-does-rootless-podman-work
[11] https://awesome-workshop.github.io/docker-singularity-hats/aio/index.html
[12] https://www.redhat.com/en/blog/say-hello-buildah-podman-and-skopeo
[13] https://www.redhat.com/sysadmin/podman-docker-compose
[14] https://www.redhat.com/sysadmin/compose-podman-pods
[15] https://www.redhat.com/sysadmin/rootless-podman-makes-sense
[16] https://www.redhat.com/sysadmin/compose-kubernetes-podman
[17] How to auto-start rootless pods using systemd : https://access.redhat.com/discussions/5733161
[18] https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/
[19] https://developers.redhat.com/blog/2016/09/13/running-systemd-in-a-non-privileged-container
[20] https://developers.redhat.com/blog/2019/04/24/how-to-run-systemd-in-a-container#
[21] https://wiki.archlinux.org/title/Podman#:~:text=After%20logging%20out%20from%20machine,update(1)%20%C2%A7%20EXAMPLES.
[22] https://github.com/containers/podman/blob/main/troubleshooting.md
[23] https://docs.oracle.com/en/operating-systems/oracle-linux/podman
</pre>
