

### Kerberized-NFSv3-with-Active-Directory-KDC
Using Active Directoy to Kerberize a Linux NFS v3 server and Client


####Summary

There are a lot of misconceptions about NFS and kerberos. The main point to note is that RPCSEC_GSS is available (and
therefore Kerberos support) in both NFSv3 and v4. This document explains how to use Microsoft AD as a Kerberos Realm for Linux NFS client/server.

####Requirements

 * 1 Centos 7 NFS server
 * 1 Centos 7 NFS Client
 * 1 Windows 2012 DC
 
####Kerberos in short

It is a third party that client and servers use for service authentication (not authorization). No password along the wire, so very secure.

The KDC has a database of Principals with passwords -  User (users) and Service (clients and servers) -  which define services. It does not contain UIDs, GIDs etc. as these attributes part of identity mapping for authorization and are entirely separate and not part of the KDC. Windows AD for example combines a KDC and LDAP/X.500 based database with rich user attributes for identity mapping.

The UIDs/GIDs/users that are common to the client and server can be derived from either the same AD as the KDC, NIS,LDAP or just a local password file. The client and server need to know about the same users and use Kerberos to authenticate the User Principal to a service. If successful the KDC autheticated User Principal is mapped to the local user on the client/server. This local user can then perform actions within its authorization remit, like write files to a certain directory.
 
####Identity mapping

Not to be confused with the KDC. I chose to simply create local users on the NFS Client and Server with the same UIDs.
You can automate this with SSSD on Linux getting identity from the AD. [Here is a good article on AD and NFS identity](http://blogs.technet.com/b/filecab/archive/2012/10/09/nfs-identity-mapping-in-windows-server-2012.aspx)

####Service Principals

I created a computer object in AD for the client and server manually (to save configuring Samba and using net ads join). I also created a user object (testuser).

####Time

The NFS server and client need to have the same time in sync with the Windows DC. This is critical for Kerberos to function properly.

####Kerberos Client Install

Install these packages on client and server

<code>yum -y install krb5-workstation pam_krb5</code>

and enable RPCSEC_GSS

<code>modprobe rpcsec_gss_krb5</code>

reboot

<code>ps -elf | grep gss</code>

if rpc.gssd is not running start it thus

<code>rpc.gssd start</code>

####Kerberos Client Config

For Kerberos to work with NFS both the client and server need a valid kerberos config file <code>krb5.conf</code>

<pre>

[libdefaults]
 default_realm = CEMSAD.LOCAL
dns_lookup_realm = false
dns_lookup_kdc = false
ticket_lifetime = 24h
renew_lifetime = 7d
forwardable = true
default_tkt_enctypes = des-cbc-md5 ; or des-cbc-crc ; or aes128-cts-hmac-sha1-96
default_tgs_enctypes = des-cbc-md5 ; or des-cbc-crc ; or aes128-cts-hmac-sha1-96

[realms]
 CEMSAD.LOCAL = {
  kdc = cems-dc1.cemsad.local
admin_server=cems-dc1.cemsad.local
 }



[domain_realm]
 cemsad.local = CEMSAD.LOCAL
 .cemsad.local = CEMSAD.LOCAL

[logging]
FILE=/var/krb5/kdc.log

</pre>

####Keytab Files

#####Keytabs for Services


Keytab files automate authentication to a Kerberized service. They contains service principal and user principal keys encryped by kerberos passwords in the AD KDC. 
For a great explanation read [this link](https://ssimo.org/blog/id_016.html)

Due to the way AD Kerberos works you need to manually create a keytab file for the NFS client and server, as a service principal needs to be mapped to a user. 


Create two new users <code>nfs-client-nfs</code> and <code>nfs-server-nfs</code>

Add new SPNs associating our new user object

<pre>
setspn -A nfs/nfs-client.cemsad.local nfs-client-nfs
setspn -A nfs/nfs-client nfs-client-nfs
setspn -A nfs/nfs-server.cemsad.local nfs-server-nfs
setspn -A nfs/nfs-server nfs-server-nfs
</pre>




Create keytab file mapping our new user objects and passwords

<pre>
        
 ktpass -princ nfs/nfs-client.cemsad.local@CEMSAD.LOCAL -mapuser nfs-client-nfs -pass ****** -ptype KRB5_NT_PRINCIPAL -crypto All -out krb5.keytab

ktpass -princ nfs/nfs-server.cemsad.local@CEMSAD.LOCAL -mapuser nfs-server-nfs -pass ****** -ptype KRB5_NT_PRINCIPAL -crypto All -out krb5.keytab

</pre>

Copy keytab files to <code>/etc</code>  on your NFS client/server


#####Keytabs for users

If we want to put files on the NFS export as a particular user in AD we will have to get a kerberos ticket for that specific user to authenticate.

You can automate this by creating a keytab file for a specific user as follows - 

Create keytab file for user testuser
<pre>
ktutil
ktutil:  addent -password -p testuser@CEMSAD.LOCAL -k 1 -e RC4-HMAC
Password for testuser@CEMSAD.LOCAL:
ktutil:  wkt testuser.keytab
ktutil:  q
ls
testuser.keytab

</pre>

Get a ticket for testuser
<pre>
kinit testuser@CEMSAD.LOCAL -k -t testuser.keytab
klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: testuser@CEMSAD.LOCAL

Valid starting     Expires            Service principal
13/08/15 12:07:28  13/08/15 22:07:28  krbtgt/CEMSAD.LOCAL@CEMSAD.LOCAL
        renew until 14/08/15 12:07:28
       
 
</pre>

###NFS Server
Just a note that I set up a standard NFSv3 server.
I created the export dir with full permissions and a sticky bit to honour file ownership in the root dir

<pre>
chmod 1777 /kerbnfs
drwxrwxrwt    2 root root   24 Aug 13 16:24 kerbnfs
</pre>

The exports file looks like this - 

<code>/kerbnfs        192.168.0.0/16(rw,sec=krb5)</code>



###Tickets

So finally we can try mounting and using the export as a user in AD - 

Mount the export

<code>[root@nfs-client ~]# mount  -o sec=krb5 nfs-server:/kerbnfs/ /mnt/kerbnfs/</code>


[root@nfs-client ~]# su - testuser

</pre>

Note that I cannot see the export as I do not have a ticket yet
<pre>
[testuser@nfs-client ~]$ ls /mnt/kerbnfs/
ls: cannot access /mnt/kerbnfs/: Permission denied
[testuser@nfs-client ~]$ klist
klist: Credentials cache file '/tmp/krb5cc_1111492162' not found
</pre>
Use the user keytab I created previously and I now have a ticket granting ticket and can see the export
<pre>
[testuser@nfs-client ~]$ ls
testuser.keytab
[testuser@nfs-client ~]$ kinit testuser@CEMSAD.LOCAL -k -t testuser.keytab
[testuser@nfs-client ~]$ klist
Ticket cache: FILE:/tmp/krb5cc_1111492162
Default principal: testuser@CEMSAD.LOCAL

Valid starting     Expires            Service principal
13/08/15 17:10:03  14/08/15 03:10:03  krbtgt/CEMSAD.LOCAL@CEMSAD.LOCAL
        renew until 14/08/15 17:10:03

[testuser@nfs-client ~]$ ls /mnt/kerbnfs/
</pre>

As soon as I interact with the NFS export I get tickets for the nfs server principal
<pre>
[testuser@nfs-client ~]$ klist
Ticket cache: FILE:/tmp/krb5cc_1111492162
Default principal: testuser@CEMSAD.LOCAL

Valid starting     Expires            Service principal
13/08/15 17:10:03  14/08/15 03:10:03  krbtgt/CEMSAD.LOCAL@CEMSAD.LOCAL
        renew until 14/08/15 17:10:03
13/08/15 17:10:20  14/08/15 03:10:03  nfs/nfs-server@
        renew until 14/08/15 17:10:03
13/08/15 17:10:20  14/08/15 03:10:03  nfs/nfs-server@CEMSAD.LOCAL
        renew until 14/08/15 17:10:03
</pre>

I can now write to the export. Note that my AD credentials own the file

<pre>
[testuser@nfs-client kerbnfs]$ cd /mnt/kerbnfs/
[testuser@nfs-client kerbnfs]$ touch me
[testuser@nfs-client kerbnfs]$ ls -la
total 0
drwxrwxrwt  2 root root 15 Aug 13 17:22 .
drwxr-xr-x. 5 root root 47 Aug 12 15:43 ..
-rw-rw-r--  1 testuser testuser  0 Aug 13 17:22 me

</pre>

At this point you would expect that destroying tickets would deny you access to the export (kdestroy), but in actual fact kdestroy only destroys the tickets in userspace.<code> rpcsec_gss_krb5</code> still caches then in the kernel as documented [here](https://fedorahosted.org/gss-proxy/ticket/1)


