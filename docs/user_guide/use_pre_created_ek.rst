User Selected PCR Monitoring
============================

.. warning::
    This page is still under development and not complete. It will be so until
    this warning is removed.


Introduction
------------

The normal behavior of `keylime_agent` during startup is pretty forceful 
and uncompromising on "taking over" the TPM device. This takeover, includes
a "startup & clear (tpm2_startup -c), "change owner, endorsement and 
lockout authorizations" (tpm2_changeauth) and "generate a new Endorsement 
Key" (tpm2_createek). 

In this scenario, when the agent submits the public part of the EK to registrar, 
there isn't a specific way for an administrator to tie a particular (EK) 
to a specific piece of hardware. 

However, this kind of "chain of provenance",
which started at a factory, and included an "air-gapped boot measurement station",
before going into production is mandatory in certain high-security corporate
environments.

How to use
----------

In order to use this keylime feature, the TPM will have to initialized before 
the keylime agent is started.

First of all, the administrator has to issue the following commands against
following commands against a "clean" TPM (virtual or otherwise)::

    tpm2_startup --version
    tpm2_startup -c
    tpm2_changeauth -c o <TPM password>
    tpm2_changeauth -c e <TPM password> 
    tpm2_changeauth -c o -p <TPM password> <TPM password>
    tpm2_changeauth -c e -p <TPM password> <TPM password>
    tpm2_getcap handles-persistent
    tpm2_createek -c - -G rsa -u /tmp/test -p <EK password> -w <TPM password> -P <TPM password>
    tpm2_getcap handles-persistent

Using the last two `tpm2_getcap handles-persistent` commands, an administrator
can observe the handle of the newly created EK. At this point, in addition to 
recording such handle, the administrator can chose to store the public part of
the EK in some sort of store/database.

Afterwards, the value of the EK handle (`ek_handle`, default value is 
`generate`) can be added to `keylime.conf`, under the `[cloud_agent]` section::

    ek_handle = <hex value of handle, e.g. 0x81000000>

Now, when the keylime agent is started, it will "just" use this EK, and will
just create a new AIK. Equally important, the agent will make sure
that particular handle is not flushed on exit.

An additional clarification: this arrangement, of course, does not prevent the 
registrar from recording any EK submitted, but rather it makes the (tenant-side)
script (defined by the attribute `ek_check_script`) much more useful. An administrator
can now use it to check the EK against a list of properly pre-defined (and 
previously collected) values.
