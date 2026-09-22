## Install the following packages:

`slapd`- the OpenLDAP server

`ldap-utils`- tools for interacting with, querying and modifying entries in local or remote LDAP servers

`debconf` will prompt you for a password for the database administrator (or, in case of a noninteractive installation, a random password will be set).

after the installation.

To check the database suffix, once the server is running, user `ldapsearch` to read the `namingContexts` attribute of the root DSE.

```bash
ldapsearch -x -LLL -s base -b "" namingContexts

`output :`

dn:

namingContexts: dc=example,dc=id
```


## Tools
After the above installation, two groups of tools will be available on your system:

# OpenLDAP spcific

The OpenLDAP specific tools are low-level, and meant to be executed directly on the systems where slapd has been installed (they can generally be executed while slapd isn't running as they access the underlying database(s) directly).


