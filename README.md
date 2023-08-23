# PodmanXRootD

Bockjoo Kim, August 23, 2023


## [1] Introduction
This is the doocumentation on an XRootD deployment in the Florida CMS Tier2</b>

## [2] Preparing the Machine for the XRootD podman Needs Privileged Actions: Puppetization
### [2-1] Hardware and OS
<pre>
cmsio machines can be used as the host machines.
RHEL8 or RHEL9 host needs to be prepared.
</pre>
### [2-2] Package Installation
<pre>
<b>A. Explannation</b>
podman, podman-compose, buildah, and fuse-overlayfs are the primary packages needed.
The others are either automatic subsidiary packages or something convenience packages.
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
There are a multiple way of doing this.
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
podman XRootD container to read and write files to the Lustre, /cmsuf
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

[3-1-3] cgroup2fs : # RC
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
#### [3-1] One time setup
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
#### check with xrdmapc behavior with the server
<pre>
[bockjoo@cms server]$ xrdmapc --list all cmspodman1.rc.ufl.edu:1094 
0**** cmspodman1.rc.ufl.edu:1094
      Srv cmspodman1.ufhpc:1094
</pre>
#### check with xrdmapc behavior with the redirector
<pre>
[bockjoo@cms server]$ xrdmapc --list all cmspodman2.rc.ufl.edu:1094 
0**** cmspodman2.rc.ufl.edu:1094
      Srv cmspodman1.rc.ufl.edu:1094
</pre>
### xrdcp upload
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
[bockjoo@cms ~]$ export XrdSecPROTOCOL="ztn,unix" ; xrdcp -d 1 -f root://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list?authz=Bearer%20$BEARER_TOKEN ./ 
[2023-08-21 20:04:31.108354 -0400][Info   ][AsyncSock         ] [cmspodman2.rc.ufl.edu:1094.0] TLS hand-shake done.
[2023-08-21 20:04:31.122071 -0400][Info   ][AsyncSock         ] [cmspodman1.rc.ufl.edu:1094.0] TLS hand-shake done.
[26.51kB/26.51kB][100%][==================================================][26.51kB/s]  
[bockjoo@cms ~]$ ls -al sitedb.list 
-rwxr-xr-x 1 bockjoo avery 27150 Aug 21 20:04 sitedb.list


##### gfal-copy download (upload does not work as I can not get the storage.write:/ scope)
<pre>
[bockjoo@cms ~]$ export XrdSecPROTOCOL="ztn,unix" ; /usr/bin/python3 $(which gfal-copy) -f davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list ./
Copying davs://cmspodman2.rc.ufl.edu:1094//store/user/bockjoo/sitedb.list   [DONE]  after 0s                 
[bockjoo@cms ~]$ ls -al sitedb.list 
-rwxr-xr-x 1 bockjoo avery 27150 Aug 21 20:08 sitedb.list
</pre>

## [5] A Plan for the Migration from /cmsuf/data(accessible through the regular xrootd) to /cmsuf/podman/data(accessible through the contained xrootd)
<pre>
  In prepration
</pre>
## [6] Troubleshooting
### [6-1] Container Stops after Logout 
<pre>
Tried --restart always or nohup but didn't work
Does this issue exist on a non-VM host? The container in a physical machine did not show this issue.
</pre>
