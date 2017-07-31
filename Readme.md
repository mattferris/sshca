sshca
=====

*sshca* is a collection of tools to manage an SSH Certificate Authority. In addition to allowing you to issue and revoke certificates, it also provides a robust history mechanism to enable auditing.

For now, the scripts are hard-coded with `/root/sshca` as the base directory. If you want to change this directory to something else, the scripts will check the enviroment variable `SSHCA_BASEDIR` for an alternative value.

```
# cd /root
# git clone https://github.com/mattferris/sshca
```

Kick things off by running the setup script. This will create two CA key pairs; one for users, and one for hosts.

```
# bin/setup
```

Issuing Certificates
--------------------

Certificates are signed public keys with some extra metadata mixed in. To issue a certificate, you'll need access to the public key of the host or user you want to issue the certificate for. All certificates are issued using `bin/issue`.

```
# cd /root/sshca
# bin/issue user id=joe@example.com principals=joe pubkey=/home/joe/.ssh/id_rsa.pub
# bin/issue host id=host.example.com principals=host,host.example.com pubkey=/etc/ssh/ssh_host_rsa_key.pub
```

Every certificate issued will have a unique serial number. Serial number are sequential, with the first certificate issued receiving a serial number of `1`, the subsequent certificate receiving a serial number of `2`, and so on. The serial number of a certificate by be viewed using `ssh-keygen -Lf /path/to/cert`.

Certificates are a signed version of a public key and have no integral fingerprint of their own. *sshca* generates a fingerprint for each certificate for reference when managing certificates, and internally for storing certificates. The certificate fingerprint is a hash of the CA's user or host public key (depending on if the certificate is for a user or host) and the certificate's serial number.

By default, user certificates are valid for 90 days and host certificates are valid for 365 days. This can be changed by specifying `validity=period`, where `period` is a valid validity string (see ssh-keygen(1) for details on how to define a validity period).

The certificate ID (`id=joe@example.com`) is listed in the certificate as the `Key ID` but has no functional purpose beyond auditing. For that reason, a consistent approach for ID use would be necessary for any meaningful auditing to take place. Email addresses are probably a reasonable in most situations.

Principals (`principals=joe`), unlike IDs, are functional in nature. Certificates are only valid for users listed as principals. Therefore, Joe's certificate is only valid for authenticating Joe's account because his username (joe) is listed as a principal. Beware that it is possible to issue certificates without any principals defined. Certificates issued without a list of princpals are valid for *any* account on the host.

Similarily, host certificate principals define a list of hostnames that the certificate is valid for. As with user certificates, not defining any principals will allow the certificate to authenticate *any* host.

### Certificate Options

User certificates have additional options that can be applied to them. See ssh-keygen(1) for more information.

The following options are either enable (set to `yes`) or disabled (set to `no`).

- `opt_clear` - if set to `yes`, disables default options, allowing for enabling of individual options
- `opt_agent_fwd` - enable/disable ssh-agent forwarding
- `opt_port_fwd` - enable/disable port forwarding
- `opt_pty` - enable/disable PTY allocation
- `opt_user_rc` - enable/disable execution of `~/.ssh/rc`
- `opt_x11_fwd` - enable/disable X11 forwarding

The following options are enable if they have are defined.

- `opt_force_cmd=cmd` - force execution of `cmd` instead of a shell
- `opt_src_addr=addrlist` - restrict authentication to a list of source addresses defined in `addrlist`

### Certificate Renewals

Certificates can be renewed (reissued with the same information) easily using `bin/renew`. All you need is the serial number or fingerprint of the certificate you want to renew.

```
# bin/renew serial 5
# bin/renew fingerprint 8a:bc:37:b5:44:b0:b3:92:c3:8f:48:f3:5e:7f:69:e8:8a:dd:b7:55
```

When a certificate is issued, the list of arguments used to generate the certificate are stored alongside the certificate in the cache. Renewing a certificate will produce the execute the same command as the original certificate. Aside from a new serial number, the certificate will contain the same information.

Revoking Certificates
---------------------

Certificates can be revoked via their serial number, public key, or key ID. Note that revoking by public key or key ID doesn't affect a single certificate, it affects all certificates that share the public key or key ID. Effectively, you are revoking the public key or key ID itself, rather then specific certificates. All revocations are completed via `bin/revoke`.

```
# bin/revoke cert 10
# bin/revoke cert ad:6c:33...
# bin/revoke key 01:ac:d4...
# bin/revoke id joe@example.com
```

To generate a Key Revocation List (KRL) for both `ssh` and `sshd` to use, `bin/genkrl` must be run.

```
# bin/genkrl user
# bin/genkrl host
```

User KRLs are used by `sshd` to reject user certificates, while host KRLs are used by `ssh` to reject host certificates.

Auditing
--------

Using certificates doesn't necessarily provide any more security if you can't track the certificates, keys, and IDs used to issue them. *sshca* provides a facility for looking up information stored within the cache: `bin/lookup`. Not only is it possible to, for instances, see what certificates have been issued for a give public key, it's also possible to display the history of a said public key.

```
# bin/lookup key history fingerprint ce:f9:7c:c0:9b:5c:f0:91:16:5d:64:b8:94:68:0a:c4:bf:28:12:4c
[Mon Jul 31 00:09:52 PDT 2017] event 5df27164de5fe85c0db89ab7bbac251607c0f555: issued certificate 1 <sha1 9a:81:50:d8:38:fa:c6:8b:0e:65:11:ef:d4:31:63:c4:15:ad:0e:eb> for user 'joe@example.com': public key <sha1 ce:f9:7c:c0:9b:5c:f0:91:16:5d:64:b8:94:68:0a:c4:bf:28:12:4c>, principals joe, validity +90d, options permit-agent-forwarding,permit-port-forwarding,permit-pty,permit-user-rc,permit-x11-forwarding
[Mon Jul 31 00:29:04 PDT 2017] event 005b0c8593a84c2e22b1f7181127a6db7ff65fbf: issued certificate 2 <sha1 90:59:48:cc:f5:96:3e:7c:4f:91:e3:c0:7a:a7:6b:cc:4a:d1:da:e9> for user 'joe@example.com': public key <sha1 ce:f9:7c:c0:9b:5c:f0:91:16:5d:64:b8:94:68:0a:c4:bf:28:12:4c>, principals joe, validity +90d, options permit-agent-forwarding,permit-port-forwarding,permit-pty,permit-user-rc,permit-x11-forwarding
[Mon Jul 31 14:28:26 PDT 2017] event acc7aadbb3570b587c598549830ee30b92af3178: issued certificate 3 <sha1 f0:a6:02:93:6b:13:6b:31:ae:73:19:0a:54:d9:7d:63:4b:c3:56:9e> for user 'joe@example.com': public key <sha1 ce:f9:7c:c0:9b:5c:f0:91:16:5d:64:b8:94:68:0a:c4:bf:28:12:4c>, principals joe, validity +90d, options permit-agent-forwarding,permit-port-forwarding,permit-pty,permit-user-rc,permit-x11-forwarding: renewal of certificate <sha1 9a:81:50:d8:38:fa:c6:8b:0e:65:11:ef:d4:31:63:c4:15:ad:0e:eb>
```

Events are stored as first-class objects in cache, and are linked to their related certificate, key, and ID objects. In fact, it's possible to lookup specific events using the event's fingerprint (displayed in the history).

```
# bin/lookup event contents 5df27164de5fe85c0db89ab7bbac251607c0f555
[Mon Jul 31 00:09:52 PDT 2017] event 5df27164de5fe85c0db89ab7bbac251607c0f555: issued certificate 1 <sha1 9a:81:50:d8:38:fa:c6:8b:0e:65:11:ef:d4:31:63:c4:15:ad:0e:eb> for user 'joe@example.com': public key <sha1 ce:f9:7c:c0:9b:5c:f0:91:16:5d:64:b8:94:68:0a:c4:bf:28:12:4c>, principals joe, validity +90d, options permit-agent-forwarding,permit-port-forwarding,permit-pty,permit-user-rc,permit-x11-forwarding
# bin/lookup event keys 5df27164de5fe85c0db89ab7bbac251607c0f555
cef97cc09b5cf091165d64b894680ac4bf28124c
```
