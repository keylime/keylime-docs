Secure Payloads
================

.. warning::
    This page is still under development and not complete. It will be so until
    this warning is removed.

Secure payloads offer the ability to securely provision secrets to an enrolled node.

The payload itself is sent via the Keylime Tenant to the verifier and only when a node
has passed its enrolment criteria (including any `tpm_policy` or IMA whitelist),
will the `v_key` half be passed to the agent to decrypt the payload.

The follow flow will occur during execution of a Secure payload:

* The tenant will first set up a folder containing the files to be copied and the scripts to be executed by the agent
* The tenant will generate a new CA (if one hasn't been generated yet) and then generate a new cert/key combo for the agent to be bootstrapped
* The tenant will include the payload files with the `--include` option
* Tenant and verifier will bootstrap the agent and derive a key as normal
* Once the bootstrap key has been derived and tpm_policy and IMA verification has passed, Keylime will decrypt and extract the zipped payload that includes both the cert/key and files situated in the payload folder.
* Then Keylime will run the provided autorun.sh script.

The scripts work by leveraging two features of Keylime:

* Automatic certificate generation/delivery
* The ability to run scripts provided in the payload after successful bootstrapping


For the following example, we will provision some SSH keys onto the agent.

1. Create a directory to host the files and `autorun.sh` script. For this example, we will use `payload`
2. Create an `autorun.sh` script

```
#!/bin/bash

mkdir -p /root/.ssh/
cp payload_id_rsa* /root/.ssh/
chmod 600 /root/.ssh/payload_id_rsa*
```

3. Copy the files you wish to provision into the `payload` directory.

```
$ ls payload/
payload_id_rsa.pub
payload_id_rsa
```

Send the files using the Keylime Tenant

```
keylime_tenant -t <agent-ip> --cert myca --include payload
```

 The `--cert` option tells Keylime to create a certificate authority at the default location `/var/lib/keylime/ca`
 and give this machine an X509 identity with its UUID. Keylime will also create a revocation certificate for this CA
 and make it available to the verifier. Finally, the `--include` option tells Keylime to securely deliver the files
 in the payload directory (`payload` in our case) along with the X509 certs to the targeted agent machine.

If the enrolment has been successful you will be able to see that the contents of the `payload` directory in `/var/lib/keylime/secure/unzipped`
along with the certs and included files. You should also see the ssh keys we included made in `/root/.ssh` directory from where
the autorun.sh script was ran.

Revocation

It is also possible to configure scripts for execution should a node fail any given criteria (IMA measurements for example).

To configure this, we will use our `payload` directory again.

First create a python script with the preface of `local_action`

For example `local_action_rm_ssh.py`

Within this script create an `execute` function:

```
import os

async def execute(json_revocation):
	os.remove("/root/.ssh/payload_id_rsa")
	os.remove("/root/.ssh/payload_id_rsa.pub")
```

And finally include your code within the `execute` function.

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

Secondly many actions can be committed based on CA revocation. For more details and examples, please refer to the
[Revocation](revocation.md) page.
