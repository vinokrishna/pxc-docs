# Encrypt PXC traffic

There are two kinds of traffic in Percona XtraDB Cluster:

1. Client-server traffic (the one between client applications and cluster
nodes),

2. Replication traffic, that includes [SST](../glossary.md#sst), [IST](../glossary.md#ist), write-set replication, and various service messages.

Percona XtraDB Cluster supports encryption for all types of traffic. Replication traffic
encryption can be configured either automatically or manually.

## Encrypt client-server communication

Percona XtraDB Cluster uses the underlying MySQL encryption mechanism
to secure communication between client applications and cluster nodes.

MySQL generates default key and certificate files and places them in the data
directory. You can override auto-generated files with manually created ones, as
described in the section [Generate keys and certificates manually](#generate-keys-and-certificates-manually).

The auto-generated files are suitable for automatic SSL configuration, but you
should use the same key and certificate files on all nodes.

Specify the following settings in the `my.cnf` configuration file
for each node:

```text
[mysqld]
ssl-ca=/etc/mysql/certs/ca.pem
ssl-cert=/etc/mysql/certs/server-cert.pem
ssl-key=/etc/mysql/certs/server-key.pem

[client]
ssl-ca=/etc/mysql/certs/ca.pem
ssl-cert=/etc/mysql/certs/client-cert.pem
ssl-key=/etc/mysql/certs/client-key.pem
```

After it is restarted, the node uses these files to encrypt communication with
clients. MySQL clients require only the second part of the configuration
to communicate with cluster nodes.

MySQL generates the default key and certificate
files and places them in the data directory. You can either use them or generate
new certificates. For generation of new certificate please refer to
[Generate keys and certificates manually](#generate-keys-and-certificates-manually) section.

## Encrypt replication traffic

*Replication traffic* refers to the inter-node traffic which includes
the [SST](../glossary.md#sst) traffic, [IST](../glossary.md#ist) traffic, and replication traffic.

The traffic of each type is transferred via a different channel, and so it
is important to configure secure channels for all 3 variants to
completely secure the replication traffic.

Percona XtraDB Cluster supports a single configuration option which helps to secure the complete
replication traffic, and is often referred to as [SSL automatic configuration](#ssl-automatic-configuration). You can
also configure the security of each channel by specifying independent
parameters.

## SSL automatic configuration

The automatic configuration of the SSL encryption needs a key and certificate
files. MySQL generates a default key and certificate files and places them in
the data directory.

!!! important

    It is important that your cluster use the same SSL certificates on all nodes.

### Enable `pxc-encrypt-cluster-traffic`

Percona XtraDB Cluster includes the `pxc-encrypt-cluster-traffic` variable that
enables automatic configuration of SSL encryption there-by encrypting
[SST](../glossary.md#sst), [IST](../glossary.md#ist), and replication traffic.

By default, `pxc-encrypt-cluster-traffic` is enabled thereby using a secured
channel for replication. This variable is not dynamic and so it cannot be changed
at runtime.

Enabled, `pxc-encrypt-cluster-traffic` has the effect of applying the following
settings: [encrypt](../manual/xtrabackup_sst.md#encrypt), ssl_key, ssl-ca, ssl-cert.

Setting `pxc-encrypt-cluster-traffic=ON` has the effect of applying the following settings in the `my.cnf` configuration file:

```text
[mysqld]
wsrep_provider_options=”socket.ssl_key=server-key.pem;socket.ssl_cert=server-cert.pem;socket.ssl_ca=ca.pem”

[sst]
encrypt=4
ssl-key=server-key.pem
ssl-ca=ca.pem
ssl-cert=server-cert.pem
```

For [`wsrep_provider_options`](../wsrep-system-index.md#wsrep_provider_options), only the mentioned options are affected (`socket.ssl_key`, `socket,ssl_cert`, and `socket.ssl_ca`), the rest is not modified.

!!! important

    **Disabling `pxc-encrypt-cluster-traffic`**

    The default value of `pxc-encrypt-cluster-traffic` helps improve the security
    of your system.

    When `pxc-encrypt-cluster-traffic` is not enabled, anyone with the
    access to your network can connect to any PXC node either as a
    client or as another node joining the cluster. This potentially
    lets them query your data or get a complete copy of it.

    If you must disable `pxc-encrypt-cluster-traffic`, you need
    to stop the cluster and update `[mysqld]` section of  the configuration file:
    ``pxc-encrypt-cluster-traffic=OFF`` of each node. Then, restart the cluster.

The automatic configuration of the SSL encryption needs key and certificate files.
MySQL generates default key and certificate
files and places them in data directory. These auto-generated files are
suitable for automatic SSL configuration, but *you should use the same key and
certificate files on all nodes*. Also you can override auto-generated files with
manually created ones, as covered in [Generate keys and certificates manually](#generate-keys-and-certificates-manually).

The necessary key and certificate files are first searched at the `ssl-ca`,
`ssl-cert`, and `ssl-key` options under `[mysqld]`. If these options are
not set, the data directory is searched for `ca.pem`,
`server-cert.pem`, and `server-key.pem` files.

!!! note

    The `[sst]` section is not searched.

If all three files are found, they are used to configure encryption.
If any of the files is missing, a fatal error is generated.

## SSL manual configuration

If user wants to enable encryption for specific channel only or
use different certificates or other mix-match, then user can opt for
manual configuration. This helps to provide more flexibility to end-users.

To enable encryption manually, the location of the required key and certificate
files shoud be specified in the Percona XtraDB Cluster configuration. If you do not have the
necessary files, see [Generate keys and certificates manually](#generate-keys-and-certificates-manually).

!!! note 

    Encryption settings are not dynamic. To enable it on a running cluster, you need to restart the entire cluster.

There are three aspects of Percona XtraDB Cluster operation, where you can enable encryption:

* [Encrypt SST traffic](#encrypt-sst-traffic)

    This refers to [SST](../glossary.md#sst) traffic during full data copy
    from one cluster node (donor) to the joining node (joiner).


* [Encrypt replication traffic](#encrypt-replication-traffic)


* [Encrypt IST traffic](#encrypt-replicationist-traffic)

    This refers to all internal Percona XtraDB Cluster communication,
    such as, write-set replication, [IST](../glossary.md#ist), and various service messages.

### Encrypt SST traffic

This refers to full data transfer
that usually occurs when a new node (JOINER) joins the cluster
and receives data from an existing node (DONOR).

For more information, see [State snapshot transfer](../manual/state_snapshot_transfer.md#state-snapshot-transfer).

!!! note 

    If `keyring_file` plugin is used, then SST encryption is mandatory: when copying encrypted data via SST, the keyring must be sent over with the files for decryption. In this case following options are to be set in `my.cnf` on all nodes:

    ```text
    early-plugin-load=keyring_file.so
    keyring-file-data=/path/to/keyring/file
    ```

    The cluster will not work if keyring configuration across nodes is
    different.

The only available SST method is `xtrabackup-v2` which uses *Percona XtraBackup*.

### xtrabackup

This is the only available SST method (the [`wsrep_sst_method`](../wsrep-system-index.md#wsrep_sst_method) is always set to `xtrabackup-v2`), which uses Percona XtraBackup to perform non-blocking transfer of files. For more information, see [Percona XtraBackup SST Configuration](../manual/xtrabackup_sst.md#xtrabackup-sst).

Encryption mode for this method is selected using the [`encrypt`](../manual/xtrabackup_sst.md#cmdoption-arg-encrypt) option:

* `encrypt=0` is the default value, meaning that encryption is disabled.

* `encrypt=4` enables encryption based on key and certificate files generated with OpenSSL. For more information, see Generating Keys and Certificates Manually.

  To enable encryption for SST using XtraBackup,
  specify the location of the keys and certificate files
  in the each node’s configuration under `[sst]`:

  ```text
  [sst]
  encrypt=4
  ssl-ca=/etc/mysql/certs/ca.pem
  ssl-cert=/etc/mysql/certs/server-cert.pem
  ssl-key=/etc/mysql/certs/server-key.pem
  ```

!!! note

    SSL clients require DH parameters to be at least 1024 bits, due to the [logjam vulnerability](https://en.wikipedia.org/wiki/Logjam_(computer_security)).
    However, versions of `socat` earlier than 1.7.3 use 512-bit parameters.
    If a `dhparams.pem` file of required length is not found during SST in the data directory, it is generated with 2048 bits, which can take several minutes.
    To avoid this delay, create the `dhparams.pem` file manually and place it in the data directory before joining the node to the cluster:

    ```shell
    openssl dhparam -out /path/to/datadir/dhparams.pem 2048
    ```

    For more information, see [this blog post](https://www.percona.com/blog/2017/04/23/percona-xtradb-cluster-dh-key-too-small-error-during-an-sst-using-ssl/).

### Encrypt replication/IST traffic

Replication traffic refers to the following:

* Write-set replication which is the main workload of Percona XtraDB Cluster
(replicating transactions that execute on one node to all other nodes).

* Incremental State Transfer ([IST](../glossary.md#ist)) which
is copying only missing transactions from DONOR to JOINER node.

* Service messages which ensure that all nodes are synchronized.

All this traffic is transferred via the same underlying communication channel
(`gcomm`). Securing this channel will ensure that [IST](../glossary.md#ist) traffic,
write-set replication, and service messages are encrypted.
(For IST, a separate channel is configured using the same configuration
parameters, so 2 sections are described together).

To enable encryption for all these processes,
define the paths to the key, certificate and certificate authority files
using the following [wsrep provider options](../wsrep-provider-index.md#wsrep-provider-index):


* [`socket.ssl_ca`](../wsrep-provider-index.md#socket.ssl_ca)

* [`socket.ssl_cert`](../wsrep-provider-index.md#socket.ssl_cert)

* [`socket.ssl_key`](../wsrep-provider-index.md#socket.ssl_key)

To set these options, use the [`wsrep_provider_options`](../wsrep-system-index.md#wsrep_provider_options) variable in the configuration file:

```shell
wsrep_provider_options="socket.ssl=yes;socket.ssl_ca=/etc/mysql/certs/ca.pem;socket.ssl_cert=/etc/mysql/certs/server-cert.pem;socket.ssl_key=/etc/mysql/certs/server-key.pem"
```

!!! note 

    You must use the same key and certificate files on all nodes, preferably those used for [Encrypt client-server communication](#encrypt-client-server-communication).

Check [upgrade-certificate](#upgrade-certificates) section on how to upgrade existing certificates.

## Generate keys and certificates manually

As mentioned above, MySQL generates default key and certificate
files and places them in the data directory. If you want to override these
certificates, the following new sets of files can be generated:

* *Certificate Authority (CA) key and certificate* to sign the server and client certificates.

* *Server key and certificate* to secure database server activity and write-set replication traffic.

* *Client key and certificate* to secure client communication traffic.

These files should be generated using [OpenSSL](https://www.openssl.org/).

!!! note 

    The `Common Name` value used for the server and client keys and certificates must differ from that value used for the CA certificate.

=== "Generate CA key and certificate"

    The Certificate Authority is used to verify the signature on certificates.

    1. Generate the CA key file:

        ```{.bash data-prompt="$"}
        $ openssl genrsa 2048 > ca-key.pem
        ```

    2. Generate the CA certificate file:

        ```{.bash data-prompt="$"}
        $ openssl req -new -x509 -nodes -days 3600
            -key ca-key.pem -out ca.pem
        ```

=== "Generate server key and certificate"

    1. Generate the server key file:

        ```{.bash data-prompt="$"}
        $ openssl req -newkey rsa:2048 -days 3600 \
            -nodes -keyout server-key.pem -out server-req.pem
        ```

    2. Remove the passphrase:

        ```{.bash data-prompt="$"}
        $ openssl rsa -in server-key.pem -out server-key.pem
        ```

    3. Generate the server certificate file:

        ```{.bash data-prompt="$"}
        $ openssl x509 -req -in server-req.pem -days 3600 \
            -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
            -out server-cert.pem
        ```

=== "Generate client key and certificate"

    1. Generate the client key file:

        ```{.bash data-prompt="$"}
        $ openssl req -newkey rsa:2048 -days 3600 \
            -nodes -keyout client-key.pem -out client-req.pem
        ```

    2. Remove the passphrase:

        ```{.bash data-prompt="$"}
        $ openssl rsa -in client-key.pem -out client-key.pem
        ```

    3. Generate the client certificate file:

        ```{.bash data-prompt="$"}
        $ openssl x509 -req -in client-req.pem -days 3600 \
           -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
           -out client-cert.pem
        ```

### Verify certificates

To verify that the server and client certificates
are correctly signed by the CA certificate,
run the following command:

```{.bash data-prompt="$"}
$ openssl verify -CAfile ca.pem server-cert.pem client-cert.pem
```

If the verification is successful, you should see the following output:

```text
server-cert.pem: OK
client-cert.pem: OK
```

### Failed validation caused by matching CN

Sometimes, an SSL configuration may fail if the certificate and the CA files contain the same .

To check if this is the case run `openssl` command as follows and verify that the **CN** field differs for the *Subject* and *Issuer* lines.

```{.bash data-prompt="$"}
$ openssl x509 -in server-cert.pem -text -noout
```

**Incorrect values**

```{.text .no-copy}
Certificate:
Data:
Version: 1 (0x0)
Serial Number: 1 (0x1)
Signature Algorithm: sha256WithRSAEncryption
Issuer: CN=www.percona.com, O=Database Performance., C=US
...
Subject: CN=www.percona.com, O=Database Performance., C=AU
...
```

To obtain a more compact output run `openssl` specifying -subject and -issuer parameters:

```{.bash data-prompt="$"}
$ openssl x509 -in server-cert.pem -subject -issuer -noout
```

??? example "Expected output"

    ```{.text .no-copy}
    subject= /CN=www.percona.com/O=Database Performance./C=AU
    issuer= /CN=www.percona.com/O=Database Performance./C=US
    ```

### Deploy keys and certificates

Use a secure method (for example, `scp` or `sftp`)
to send the key and certificate files to each node.
Place them under the `/etc/mysql/certs/` directory
or similar location where you can find them later.

!!! note 

    Make sure that this directory is protected with proper permissions.
    Most likely, you only want to give read permissions to the user running `mysqld`.

The following files are required:

* Certificate Authority certificate file (`ca.pem`)

  This file is used to verify signatures.

* Server key and certificate files (`server-key.pem` and `server-cert.pem`)

  These files are used to secure database server activity
  and write-set replication traffic.

* Client key and certificate files (`client-key.pem` and `client-cert.pem`)

  These files are required only if the node should act as a MySQL client.
  For example, if you are planning to perform SST using `mysqldump`.

!!! note 

    [Upgrade certificates](#upgrade-certificates) subsection covers the details on upgrading certificates, if necessary.

### Upgrade certificates

The following procedure shows how to upgrade certificates
used for securing replication traffic when there are two nodes in the cluster.


1. Restart the first node with the [`socket.ssl_ca`](../wsrep-provider-index.md#socket.ssl_ca) option set to a combination of the the old and new certificates in a single file.

    For example, you can merge contents of `old-ca.pem`
    and `new-ca.pem` into `upgrade-ca.pem` as follows:

    ```shell
    cat old-ca.pem > upgrade-ca.pem && \
    cat new-ca.pem >> upgrade-ca.pem
    ```

    Set the [`wsrep_provider_options`](../wsrep-system-index.md#wsrep_provider_options) variable as follows:

    ```shell
    wsrep_provider_options="socket.ssl=yes;socket.ssl_ca=/etc/mysql/certs/upgrade-ca.pem;socket.ssl_cert=/etc/mysql/certs/old-cert.pem;socket.ssl_key=/etc/mysql/certs/old-key.pem"
    ```

2. Restart the second node with the [`socket.ssl_ca`](../wsrep-provider-index.md#socket.ssl_ca), [`socket.ssl_cert`](../wsrep-provider-index.md#socket.ssl_cert), and [`socket.ssl_key`](../wsrep-provider-index.md#socket.ssl_cert) options set to the corresponding new certificate files.

    ```shell
    wsrep_provider_options="socket.ssl=yes;socket.ssl_ca=/etc/mysql/certs/new-ca.pem;socket.ssl_cert=/etc/mysql/certs/new-cert.pem;socket.ssl_key=/etc/mysql/certs/new-key.pem"
    ```

3. Restart the first node with the new certificate files, as in the previous step.

4. You can remove the old certificate files.
