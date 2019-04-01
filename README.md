Signed SSH Host Keys Ansible Role
---------------------------------

We constantly are spinning up new machines, using Ansible to provision them,
and one of our pain points has been managing the ssh `known_hosts` file.  We
had roles and scripts to try to build and update a `known_hosts`, but it was
continuously out of date.

We have solved this by using the SSH ability to sign host keys, and list the
signing key in the `known_hosts` file instead of individual host keys.  This
works brilliantly.

Using This Role
===============

To use this role, you need to bring it into your Ansible setup.  If you are
using "git" to manage your Ansible setup, you can set this up as a submodule
with:

    git submodule add https://github.com/linsomniac/ssh_signed_hostkeys.git roles/ssh_signed_hostkeys

Or, simply clone it:

    git clone add https://github.com/linsomniac/ssh_signed_hostkeys.git roles/ssh_signed_hostkeys

To begin using it, you will need to:

  - Add `ssh_signed_hostkeys` as a role to your playbook.
  - Put a `ca_key` file in `roles/ssh_signed_hostkeys/files` directory,
    preferably encrypted.  See below for more information.
  - Add the public key to your `known_hosts` file.  See below for more information.
  - Optional: Set a key identifier as `signed_ssh_certificate_identity` in
    `group_vars/all`.  The default is "ssh_signed_hostkeys".  This can be any
    string, it is used to identify what CA key was used to sign the keys.

Then just run Ansible and your host keys should be signed and ready to go.

CA Setup
========

To set up a SSH CA, you just need to generate a key for use.  For example:

    ssh-keygen -t ecdsa -b 521 -N '' -f roles/ssh_signed_hostkeys/files/ca_key

Ideally, you'll want to encrypt this file using the Ansible vault:

    ansible-vault encrypt roles/ssh_signed_hostkeys/files/ca_key

It will ask for a passphrase that will be needed to decrypt the file in the
future.  You can use vault helpers to automate this decryption for convenience
or automation.

`Known_Hosts` Line
==================

Once the host key is signed, you will need to add the public key to your
`known_hosts` file(s).  You will need a line something like:

    @cert-authority <HOSTNAME>[,<ANOTHER HOSTNAME] <CONTENTS OF ca_key.pub>

The HOSTNAME can be a wildcard like `*.example.com`, and you can list
multiple hostnames like: `localhost,*.example.com,*.example.org`.  These need
to match the HostName used to ssh to the machine.

At this point you should be able to ssh to the host that has the signed key,
without being prompted to verify the host key or needing it's key in your
`known_hosts` file.  My `known_hosts` file went down from 400 lines to 12.

You will probably want to deploy `/etc/ssh/ssh_known_hosts` to your systems so
that you don't have to add entries to each individual user file.

To test, run ssh with the `-v` option, and it should show that it is using
the signed host key.

Role Settings
=============

The following settings are available to override the default workings of this
role:


  - `signed_ssh_keyfiles` -- Host keyfiles to sign.  Defaults to:
        - /etc/ssh/ssh_host_rsa_key
        - /etc/ssh/ssh_host_ecdsa_key
        - /etc/ssh/ssh_host_ed25519_key
  - `signed_ssh_hostnames` -- Names to associate with the signature.  This
    must match one of the names used as the "hostname" for SSH, typically
    specified on the ssh command-line or via the "HostName" directive in the
    config.  Default:
        - localhost
        - "{{ansible_fqdn}}"
        - "{{ansible_fqdn.split('.')[0]}}"
        - "{{'.'.join(ansible_fqdn.split('.')[:2])}}"
  - `signed_ssh_nuke_existing_sigs` -- Remove existing signatures and resign?
    Use this if you change the hostnames on the signature (via
    `signed_ssh_hostnames`) to update the certificates with the new names.
    Run with `--extra-vars 'signed_ssh_nuke_existing_sigs=true'`
    Default: false
  - `signed_ssh_valid_time` -- How long the signature is valid for, as in the
    "-V" argument to `ssh-keygen`.  Default: `+520w`
  - `signed_ssh_certificate_identity` -- A string identifying the signature,
    possibly your name or  company name.  Default: `ssh_signed_hostkeys`
