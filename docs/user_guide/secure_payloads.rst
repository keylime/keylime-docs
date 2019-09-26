Secure Payloads
================

.. warning::
    This page is still under development and not complete. It will be so until
    this warning is removed.

Secure payloads offer the ability to provision encrypted data to an enrolled node.
This encrypted data can be used to deliver keys, passwords, certificate roots of
trust, etc.

Secure payloads are for anything which requires strong confidentiality and
integrity to bootstrap your system.

The payload itself is encrypted and sent via the Keylime Tenant CLI (or rest API)
to the Keylime Agent. The Agent also sends one key share, called the `u_key` or
user key, of the key needed to decrypt the payload. Only when the agent has
passed its enrolment criteria (including any `tpm_policy` or IMA whitelist),
will the other key share of the decryption key, called the `v_key` or verification
key, be passed to the agent by the Keylime Verifier to decrypt the payload.

.. note:: An alternative to secure payloads is to deliver the encrypted data to
    the node through some other mechanism like `cloud-init` or pre-embedded in a
    disk image.  The Keylime protocol described above will still run to derive
    the decryption key for this data, but the data itself will never been seen
    or transported by Keylime.  This guide does not discuss this method.

Keylime offers several modes for sending secure payloads: single file encryption
and certificate package mode. In the following sections we describe each.  If
you're interested in using the mode advanced certificate package mode, we
recommend you also read the single file encryption section as it contains
configuration options and other information that both modes share.

Single File Encryption
----------------------

In this mode, a file you specify to the `keylime_tenant` application with the `-f`
option will be encrypted by the Tenant using the bootstrap key and securely
delivered to the Agent.  Once the Keylime protocol with the Tenant and Verifier
has completed, the Keylime Agent will decrypt this file and place it in `/var/lib/keylime/secure/decrypted_payload`
This is the default file name, but you can adjust the name of this file using the
`dec_payload_file` option in `keylime.conf`.  You can also optionally specify a
zip file as the file to be securely delivered.  If the `extract_payload_zip`
option in `keylime.conf` is set (which it is by default), then Keylime will
automatically extract the zip file to `/var/lib/keylime/secure/unzipped`.Finally,
Keylime can also execute a script contained in the zip file once it has been
unzipped.  You can think of this like a very simple form of `cloud-init`. By
default this script is called `autorun.sh`. You can configure this to some other
script with the `payload_script` option in `keylime.conf`.  Note that this script
must be contained in the encrypted zip file and must be located in `/var/lib/keylime/secure/unzipped`.

Because the keys that Keylime uses to decrypt the data and the decrypted data
itself are very sensitive, Keylime will only write those files to the
memory-backed (and therefore non-persistent) `/var/lib/keylime/secure` directory.
This is a bind-mounted tmpfs partition.  Depending on how large your payload is,
you may need to increase the size of this mount by adjusting the `secure_size`
option in `keylime.conf`.

This simple mode of operation is suitable for many situations where the secrets
and other bootstrapping information is basic.  However, there are several
features that Keylime supports like revocation and certificate management that
do not work in this mode.  For those, you'll need the next mode: Certificate
Package Mode.


Certificate Package Mode
------------------------

This mode of Keylime automates many common actions that tenants will want to do
when provisioning their agents.  First, Keylime can create an X509 certificate
authority and then issue certificates and corresponding private keys to each
provisioned node.  This CA lives on the tenant and can be used to bootstrap many
other security solutions like mutual TLS or  SSH.  To use this mode, pass
the `--cert` option and a directory where the CA is located as the parameter to
this option.  Keylime will then create a certificate for the node (with the common
name set to the agents UUID) and then create a zip file containing the newly
generated X509 certificates, trust roots, and private keys. It then uses the
same process for single file encryption as described above to securely deliver
all the keys to the agent.  Optionally, the user can specify with the `--include`
option a directory of additional files to put into the certification package zip
and be securely delivered to the agent.

This mode of operation also natively supports certificate revocation. If the Keylime
Verifier detects an agent that no longer satisfies its integrity policy (e.g., it booted
an authorized kernel or ran an unauthorized binary not on the IMA whitelist), it
will create a signed revocation notification.  These revocation notifications are
signed by a special certificate/private key called the RevocationNotifier.  Keylime
will automatically create this certificate and pass it to the verifier when you add
a new agent to the verifier.  Keylime will also include the public certificate for
this key in the zip it sends to the agent. This way agents can validate the
revocation notifications they receive from the verifier.

By default all Keylime Agents listen for these revocation notifications (see
the `listen_notifications` option in `keylime.conf`). Using the keys in the
unzipped certificate package, Agents can check that the revocations are valid.
Keylime Agents can also take some actions in response to a valid revocation.
You can configure these actions by putting additional files into the delivered zip
file using `--include`.

Revocation actions are small python scripts that will run on an Agent when a valid
revocation is received.  They should contain an `execute` function that takes
one argument.  This argument is a python dictionary of metadata that can be used
to tailor what the revocation action does.  In the cert package mode, Keylime
will specify the certificate serial number and common name (aka UUID) of the node
that has failed its integrity check inside this metadata passed to the revocation
action.  For example, you can use this info to revoke the the offending X509
certificate.

One subtlety to revocation actions is that they are not intended for the agent
that has been revoked.  If an agent has failed its integrity check, then we
really can't trust that it won't ignore the revocations and do arbitrarily
malicious things.  So, revocation actions are for other well-behaving agents in
the system to take action against the revoked agent.  For example, by revoking
its certificate as described above or blacklisting it from the network, etc.

There are several revocation actions implemented in Keylime that you can look at
to get an idea of how to write your own.  See the files starting with `local_action`
in https://github.com/keylime/keylime/tree/master/auto-ipsec/libreswan/src

There are some conventions to specifying revocation actions. As described above,
their names must start with `local_action` to be executed. They also must be
listed (without `.py` extensions) in a comma separated list in a file called
`action_list` in the zip file.  For example to run `local_action_a.py` and
`local_action_b.py` the `action_list` file should contain `local_action_a,local_action_b`.

So far we've described all the details of this in fine detail, but much of this
automation will happen by default.

Certificate Package Example
---------------------------

Lets put all of the above together with an example. For the following example,
we will provision some SSH keys onto the agent.

1. Create a directory to host the files and `autorun.sh` script. For this example, we will use the directory `payload`
2. Create an `autorun.sh` script in the `payload` directory:

```
#!/bin/bash

# this will make it easier for us to find our own cert
ln -s `ls *-cert.crt | grep -v Revocation` mycert.crt

mkdir -p /root/.ssh/
cp payload_id_rsa* /root/.ssh/
chmod 600 /root/.ssh/payload_id_rsa*
```

3. Copy the files you wish to provision into the `payload` directory.

```
$ ls payload/
autorun.sh
payload_id_rsa.pub
payload_id_rsa
```

Send the files using the Keylime Tenant

```
keylime_tenant -t <agent-ip> --cert myca --include payload
```

Recall the `--cert` option tells Keylime to create a certificate authority at the
default location `/var/lib/keylime/ca` and give this machine an X509 identity
with its UUID. Keylime will also create a revocation notifier certificate for this
CA and make it available to the verifier. Finally, the `--include` option tells
Keylime to securely deliver the files in the specified directory (`payload` in
our case) along with the X509 certs to the targeted agent machine.

If the enrolment has been successful you will be able to see that the contents
of the `payload` directory in `/var/lib/keylime/secure/unzipped` along with the
certs and included files. You should also see the ssh keys we included made
in `/root/.ssh` directory from where the autorun.sh script was ran.

Now, lets extend this example with revocation.  In this example, we're going to
execute a simple revocation action on the node that was revoked.

It is also possible to configure scripts for execution should a node fail any
given criteria (IMA measurements for example).

To configure this, we will use our `payload` directory again.

First create a python script with the preface of `local_action`

For example `local_action_rm_ssh.py`

Within this script create an `execute` function:

```
import os
from M2Crypto import X509
import keylime.secure_mount as secure_mount

async def execute(json_revocation):
    if json_revocation['type']!='revocation':
        return

    serial = revocation.get("metadata",{}).get("cert_serial",None)
    if revocation.get('type',None) != 'revocation' or serial is None:
        logger.error("Unsupported revocation message: %s"%revocation)

    # load up my own cert
    secdir = secure_mount.mount()
    ca = X509.load_cert('%s/unzipped/mycert.crt'%secdir)

	# is this revocation meant for me?
	if serial === mycert.get_serial_number():
		os.remove("/root/.ssh/payload_id_rsa")
		os.remove("/root/.ssh/payload_id_rsa.pub")
```

Next create the `action_list` file with `local_action_rm_ssh` in it in the
`payload` directory.

.. warning::
    In the above example, the node that fails its integrity check is the same one
    that we're expecting to run the revocation action to delete the key. Since
    the node is potentially compromised, we really can't expect that it will
    actually do this and not just ignore the revocation. A more realistic
    scenario for ssh keys is to provision one node with the ssh key generated
    as above, then provision a second server and add `payload_id_rsa.pub` to `.ssh/authorized_keys`
    using an autorun script. At this point, you can ssh from the first server to
    the second one. Should the first machine fail its integrity, then an
    revocation action  on the second server can remove the compromised first
    machine from its list of Secure machines in `.ssh/authorized_keys`

Secondly many actions can be committed based on CA revocation. For more details
and examples, please refer to the [Revocation](revocation.md) page.
