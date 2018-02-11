# About

Since GnuPG 2.1.0 the private GPG keys are handled by the `gpg-agent`. This
allows to split the `gpg2` (command line tool - handles public keys, etc.) and the
`gpg-agent` which handles the private keys.

Purpose of this project is to split `gpg2` and `gpg-agent` between two Qubes domains.
Since normally the gpg-agent is run locally no filtering is provided. So to make this
a meaningful security feature the main function is the filtering of the commands from
the less trusted domain running `gpg2`. The rest are some script to "tunnel" the
`gpg-agent`-commands through Qubes RPC.

The **server** is the domain which runs the (real) `gpg-agent`.  
The **client** is the domain which which access the server via Qubes RPC.

The server domain is generally considered more trustfull then the client domain.
This implies that the response from the server is _not_ santized.


## Building

### Debian packages

The following dependencies need to be met to build Debian packages:  
`debhelper`, `dh-systemd`, `dpkg-dev`.
 
```
cd /path/to/split-gpg2-source
dpkg-buildpackage -us -uc
```

Built packages along with some additional files will be saved in `split-gpg2` source's
parent directory.

### RPM packages

You will need the following packages to build RPM packages:  
`rpm` (on Debian-based distros) or `rpm-build` (on Fedora and alike).

```
cd /path/to/split-gpg2-source
rpmbuild -ba rpm_spec/split-gpg2.spec
rpmbuild -ba rpm_spec/split-gpg2-dom0.spec
```

Built binary packages will be saved in `~/rpmbuild/RPMS/`; source packages will be
saved in `~/rpmbuild/SRPMS/` .

## Installation

Install the the debian or the rpm on your vm-template.

Install the dom0-rpm in dom0. (Or just create a proper
`/etc/qubes-rpc/policy/qubes.Gpg2`).

Create `~user/.split-gpg2-rc` in the server and in the client domain.
See [`/usr/share/split-gpg2/examples/split-gpg2-rc.example`](./split-gpg2-rc.example) for an example and the
available options.

Enable the `split-gpg2-client` service in the client domain either via the gui or
via command line:

```
# in R3.x
qvm-service gpg-client-vm enable split-gpg2-client # in R3.x
# in R4.x
qvm-sevice --enable gpg-client split-gpg2-client
```

Restart the client domain.

You should now be able the run gpg2 in the client domain.

***NOTICE: For keys for which you have the _secret_ part you need to import the _public_
part in both the client and the server domain. The secret part should only be
imported in the server domain.***

Other public keys should only be imported in the client domain.

## Copyright

See [`LICENSE`](./LICENSE)

```
Copyright (C) 2014  HW42 <hw42@ipsumj.de>

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
```
