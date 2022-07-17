# split-gpg2

## About split-gpg2

This software allows you to separate the handling of private key material from the rest of GnuPG's processing.
This is similar to how smartcards work, except that in this case the handling of the private key is not put into some small microcontroller but into another Qubes domain.

Since GnuPG 2.1.0 the private gpg keys are handled by the gpg-agent.
This allows to split `gpg` (the cmdline tool - handles public keys, etc.) and the `gpg-agent` which handles the private keys.
This software implements this for Qubes.
This mainly consists of a restrictive filter in front of `gpg-agent`, written in a memory safe language (python).

The server is the domain which runs the (real) `gpg-agent` and has access to your private key material.
The client is the domain in which you run `gpg` and which accesses the server via Qubes RPC.

The server domain is generally considered more trustful then the client domain.
This implies that the response from the server is _not_ sanitized.


## Requirements

 - Python 3.5 or newer
 - GnuPG 2.1 or newer

## Building

Add it as a component to [qubes-builder](https://www.qubes-os.org/doc/qubes-builder/) and built it.

For development you can also build the Debian packet in tree.
Just run `dpkg-buildpackage -us -uc` at the top level.

## Installation

Install the deb or the rpm on your TemplateVM(s).

Create/Edit `/etc/qubes-rpc/policy/qubes.Gpg2` in dom0, it should contain something like:

```
gpg-client-vm gpg-server-vm allow
```

## Configuration

Import/Generate your secret keys in the server domain.
For example:
```
gpg-server-vm$ gpg --import /path/to/my/secret-keys-export
gpg-server-vm$ gpg --import-ownertrust /path/to/my/ownertrust-export
```
or
```
gpg-server-vm$ gpg --gen-key
```

Now configure which domain the client VM should use as server. Either:
 - Set the target for `@default` in qubes.Gpg2. or
 - Write `SPLIT_GPG2_SERVER_DOMAIN=<gpg-server>` into `~/.config/.split-gpg2-rc`.

In dom0 enable the `split-gpg2-client` service in the client domain, for example via the command-line:
```shell
dom0$ qvm-service <SPLIT_GPG2_CLIENT_DOMAIN_NAME> split-gpg2-client on
```

To verify if this was done correctly:
```shell
dom0$ qvm-service <SPLIT_GPG2_CLIENT_DOMAIN_NAME>
```

Output should be:
```shell
split-gpg2-client on
```

Restart the client domain.

Export the **public** part of your keys and import them in the client domain.
Also import/set proper "ownertrust" values.
For example:
```
gpg-server-vm$ gpg --export > public-keys-export
gpg-server-vm$ gpg --export-ownertrust > ownertrust-export
gpg-server-vm$ qvm-copy public-keys-export ownertrust-export

gpg-client-vm$ gpg --import ~/QubesIncoming/gpg-server-vm/public-keys-export
gpg-client-vm$ gpg --import-ownertrust ~/QubesIncoming/gpg-server-vm/ownertrust-export
```

This should be enough to have it running:
```
gpg-client-vm$ gpg -K
/home/user/.gnupg/pubring.kbx
-----------------------------
sec#  rsa2048 2019-12-18 [SC] [expires: 2021-12-17]
      50C2035AF57B98CD6E4010F1B808E4BB07BA9EFB
uid           [ultimate] test
ssb#  rsa2048 2019-12-18 [E]
```

If you want change some server option copy `/usr/share/doc/split-gpg2/examples/split-gpg2-rc.example` to `~/.config/split-gpg2-rc` and change it as desired.

If you have a passphrase on your keys and `gpg-agent` only shows the "keygrip" (something like the fingerprint of the private key) when asking for the passphrase, then make sure that you have imported the public key part in the server domain.

## Migrating from split-gpg1

Unlike split-gpg1, split-gpg2 does *not* know what is being signed.  Therefore, a client can sign other keys as well as data.  In particular, a client can create a signing-capable subkey and use this to sign whatever they want without further prompting.  They can even export that key and send it elsewhere.

To prevent this from happening, split-gpg2 must not have access to the primary key.  You can ensure this by by setting up a separate GnuPG home directory for split-gpg2, and running `gpg2 --export-secret-subkeys | gpg2 --homedir "$NEWHOMEDIR" --import-secret-keys`.  split-gpg2 will do this automatically if you have not configured it to do otherwise.

You **must** have a signing-capable subkey configured for this to work.  split-gpg2 will not generate one automatically, as it does not know if you are using a smart card or what parameters should be used to generate the key.  Also, if you generate a new key in your normal home directory, it will not automatically be made available in split-gpg2.  Future releases may lift both of these limitations.  In the meantime, you can run `splitgpg2-migrate-subkeys` to make all of your secret subkeys available to split-gpg2, even if the split-gpg2 home directory already exists.

## Advanced usage

There are a few option not described in this README.
See the comments in the example config and the source code.

Similar to a smartcard split-gpg2 only tries to protect the private key.
For advanced usages consider if a specialized RPC service would be better.
It could do things like checking what data is singed, detailed logging, exposing the encrypted content only to a VM without network, etc.

## Allow key generation

By setting `SPLIT_GPG2_ALLOW_KEYGEN=yes` in `split-gpg2-rc` you can allow the client to generate new keys.
Normal usage should not need this.

**Warning**: This feature is new and not much tested.
Therefore it's not security supported!

## Copyright

Copyright (C) 2014 HW42 <hw42@ipsumj.de>\
Copyright (C) 2019 Marek Marczykowski-Górecki <marmarek@invisiblethingslab.com>

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License along
with this program; if not, write to the Free Software Foundation, Inc.,
51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.
