---
title: "Configure  a highly available Cassandra cluster"
linkTitle: "Configure HA"
weight: 2
date: 2019-06-05
description: >
  Set up an Apache Cassandra database cluster for high availability (HA) of your API Gateway system.
---

## Cassandra HA in a production environment

To tolerate the loss of one Cassandra node and to ensure 100% data
consistency, API Gateway requires at a minimum the following Cassandra
cluster configuration running in a production class environment:

  - Three Cassandra nodes (with one seed node)
  - `QUORUM` read/write consistency to ensure that you are reading from
    a quorum of Cassandra nodes (two) every time
{{% alert title="Caution" %}}
Eventual consistency is not supported in a production environment due to a risk of stale or missing data.
{{% /alert %}}
  - `Replication factor` setting set to `3`, so each node holds 100% of
    the data

If one Cassandra node fails or needs to be restarted, the cluster
continues to operate with the remaining two nodes. This configuration
applies to all supported use cases (for example, API Manager and API
Gateway custom KPS, OAuth, and client registry data).

### Preparing a Cassandra HA environment

The following general guidelines apply to configuring a Cassandra HA
cluster:

  - Decide on the number of Cassandra nodes, and install and configure
    the Cassandra database on each node.
  - Decide on the number of API Gateway nodes, and install and configure
    them (local or remote to Cassandra).
{{% alert title="Note" %}}
It is recommended that you configure a Cassandra HA cluster with three Cassandra nodes, and at least two API Gateway instances (local or remote). For details, see [Cassandra deployment architectures](../cassandra_architecture).
{{% /alert %}}
  - Create a HA API Gateway environment.
  - Configure the Cassandra client settings in Policy Studio for the API
    Gateway group.

For details on how to install an Apache Cassandra database or how to
create a HA API Gateway environment, see [API Gateway Installation Guide](/bundle/APIGateway_77_InstallationGuide_allOS_en_HTML5/).

## <span id="Example"></span>Example of a HA setup using three Cassandra nodes

This section shows how to configure a three nodes Cassandra HA cluster.

### Configure the Cassandra seed node

Perform the following steps to configure a *seed* node:

1.  Assign one of the nodes as the *seed* node, and connect to it via
    SSH.
2.  Update the `CASSANDRA_HOME/conf/cassandra.yaml` file manually or
    using the `setup-cassandra` script.
    Manually:
    {{< highlight yaml "linenos=table, hl_lines=1 5 6, linenostart=9" >}}
    seed_provider, parameters, seeds: IP_SEED_NODE
    start_native_transport: true
    start_rpc: false
    native_transport_port: 9042
    listen_address: IP_SEED_NODE
    rpc_address: IP_SEED_NODE
    authenticator: org.apache.cassandra.auth.PasswordAuthenticator
    authorizer: org.apache.cassandra.auth.CassandraAuthorizer
    {{< / highlight >}}
    Using the `setup-cassandra` script:
    ```
    setup-cassandra --seed --own-ip=<IP_SEED_NODE> --nodes=3 --cassandra-config=CONFIG_FILE
    ```
4.  Start Cassandra.
5.  Verify the `CASSANDRA_HOME/logs/system.log` does not contain any
    errors or warnings.
6.  Run the `nodetool status` command to verify that *one* node is
    present in `UN` status, and that the IP address is correct.

### Configure the additional Cassandra nodes

The procedure to configure additional nodes uses the `seed` node that
you have previously
configured.

{{% alert title="Note" %}}
You must repeat these steps for each additional node.
{{% /alert %}}


1.  Connect to the additional node via SSH, and update the
    `CASSANDRA_HOME/conf/cassandra.yaml` file.
    Manually:
    ```
    seed_provider, parameters, seeds: IP_SEED_NODE
    start_native_transport: true
    start_rpc: false
    native_transport_port: 9042
    listen_address: IP_NODE_N
    rpc_address: IP_NODE_N
    authenticator: org.apache.cassandra.auth.PasswordAuthenticator
    authorizer: org.apache.cassandra.auth.CassandraAuthorizer
    ```
    Using the `setup-cassandra` script:
    ```
    setup-cassandra --seed-ip=<IP_SEED_NODE> --own-ip=<IP_NODE_n> --cassandra-config=<PATH_TO_CASSANDRA.YAML>
    ```
3.  Start Cassandra.
4.  Verify the `CASSANDRA_HOME/logs/system.log` does not contain any
    errors or warnings.
5.  Run the `nodetool status` command to verify that the *new* node is
    present in `UN` status, and that the IP address is correct.

### Create a new Cassandra database user

From the command line, execute the following commands to create a new
user:

```
>./cqlsh <IP_NODE_1> -u cassandra -p cassandra
> ALTER KEYSPACE "system_auth" WITH REPLICATION = { 'class': 'SimpleStrategy', 'replication_factor': 3 };
> CREATE USER <USERNAME> WITH PASSWORD '<PASSWORD>' SUPERUSER;
> QUIT;
>./cqlsh <IP_NODE_1> -u <ADMIN_USERNAME> -p <PASSWORD>
> ALTER USER cassandra WITH PASSWORD '<PASSWORD>' NOSUPERUSER;
> QUIT
```

### Configure the API Gateway Cassandra client connection

{{% alert title="Note" %}}
This section assumes that you have already set up a HA API Gateway environment. For details, see [API Gateway Administrator Guide](/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/).
{{% /alert %}}

In Policy Studio, open your API Gateway group configuration.

1.  Select **Server Settings > Cassandra > Authentication**, and enter
    the cassandra database user name and password.
2.  Select **Server Settings > Cassandra > Hosts**, and add a host for
    each Cassandra node in the cluster.
3.  Select **Server Settings > Cassandra > Keyspace**, and set the
    **Initial replication factor** option to `3`.
4.  Deploy the configuration to the group.

## Configure the client settings for API Gateway or API Manager

{{% alert title="Note" %}}
You need at least two API Gateways in a group for HA.
{{% /alert %}}

This section describes the following options:

  - [Configure API Gateway Cassandra client settings](#configure-api-gateway-cassandra-client-settings)
  - [Configure API Manager Cassandra client settings](#configure-api-manager-cassandra-client-settings)

### Configure API Gateway Cassandra client settings

To update the Cassandra client configuration for API Gateway, perform
the following steps:

#### Configure the API Gateway domain

1.  Ensure API Gateway has been installed on the API Gateway 1 and API
    Gateway 2 nodes. For details, see the [API Gateway Installation Guide](/bundle/APIGateway_77_InstallationGuide_allOS_en_HTML5/).
2.  Ensure an API Gateway domain has been created on the API Gateway 1
    node using `managedomain`. For more details, see [Configure an API Gateway domain](/csh?context=102&product=prod-api-gateway-77) in the
    [API Gateway Administrator Guide](/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/).

#### Configure the API Gateway Cassandra client connection

1.  In Policy Studio, open your API Gateway group configuration.
2.  Select **Server Settings > Cassandra > Authentication**, and enter
    your Cassandra user name and password (both default to `cassandra`).
3.  Select **Server Settings > Cassandra > Hosts**, and add an address
    for each Cassandra node in the cluster (`ipA`, `ipB` and `ipC` in this example).

{{% alert title="Tip" %}}
You can automate these steps by running the `updateCassandraSettings.py` script against a deployment package (`.fed`). For more details, see [Configure a highly available Cassandra cluster](../cassandra_config).
{{% /alert %}}

#### Configure the API Gateway Cassandra consistency levels

1.  Ensure that the **API Server** KPS collection has been created under
    **Environment Configuration > Key Property Stores**. This is
    required to configure Cassandra consistency levels, and is created
    automatically if you installed the **Complete** setup type.

    If you installed the **Custom** or **Standard** setup, you must
    configure OAuth or API Manager settings in Policy Studio to create
    the required KPS collections. For more details, see:

      - [Deploy OAuth configuration](/csh?context=400&product=prod-api-gateway-77) in the [API Gateway OAuth User Guide](/bundle/APIGateway_77_OAuthUserGuide_allOS_en_HTML5/)
      - [Configure API Manager Cassandra client settings](#configure-api-manager-cassandra-client-settings)

3.  Select **Environment Configuration > Key Property Stores > API Server > Data Sources > Cassandra Storage**, and click **Edit**.

4.  In the **Read Consistency Level** and **Write Consistency Level**
    fields, select
        **QUORUM**:
    ![](/Images/docbook/images/api_mgmt/api_mgmt_embedded_kps.png)

1.  Repeat this step for each KPS collection using Cassandra (for
    example, **Key Property Stores > OAuth**, or **API Portal** for API
    Manager). This also applies to any custom KPS collections that you
    have created.
2.  If you are using OAuth and Cassandra, you must also configure quorum
    consistency for all OAuth2 stores under **Libraries > OAuth2 Stores**:
      - **Access Token Stores > OAuth Access Token Store**
      - **Authorization Code Stores > Authz Code Store**
      - **Client Access Token Stores > OAuth Client Access Token Store**

{{% alert title="Note" %}}
By default, OAuth uses EhCache instead of Cassandra. For more details on OAuth, see the [API Gateway OAuth User Guide](/bundle/APIGateway_77_OAuthUserGuide_allOS_en_HTML5/).
{{% /alert %}}

#### Deploy the configuration

1.  Click **Deploy** in the toolbar to deploy this configuration to the
    API Gateway group.
2.  Restart each API Gateway in the group.

For details on any connection errors between API Gateway and Cassandra, see [Configure a highly available Cassandra cluster](../cassandra_config).

### Configure API Manager Cassandra client settings

To update the Cassandra client configuration for API Manager, perform
the following steps:

1.  Ensure the API Gateway and API Manager components have been
    installed on the API Gateway 1 and API Gateway 2 nodes. These can be
    local or remote to Cassandra installations. For details, see the [API Gateway Installation Guide](/bundle/APIGateway_77_InstallationGuide_allOS_en_HTML5/).
2.  Ensure an API Gateway domain, group, and instance have been created
    on the API Gateway 1 node using `managedomain`. For more details, see [Configure an API Gateway domain](/csh?context=102&product=prod-api-gateway-77) in the [API Gateway Administrator Guide](/bundle/APIGateway_77_AdministratorGuide_allOS_en_HTML5/).

1.  Start the first API Gateway instance in the group. For example:
```
startinstance -n "my_gw_server_1" -g "my_group"
```
2.  Configure the Cassandra connection on the API Gateway 1 node. For
    details, see [Configure the API Gateway Cassandra client connection](#configure-the-api-gateway-cassandra-client-connection).

1.  Configure the Cassandra consistency levels for your KPS Collections.
    For details, see [Configure the API Gateway Cassandra consistency levels](#configure-the-api-gateway-cassandra-consistency-levels).
2.  In the Policy Studio tree, select **Server Settings** > **API Manager** > **Quota Settings**, and ensure that **Use Cassandra**
    is selected.
3.  Under **Cassandra consistency levels**, in both the **Read** and
    **Write** fields, select `QUORUM`:
    ![Three node HA full consistency](/Images/docbook/images/api_mgmt/api_mgmt_quota_settings.png)

4.  Add the API Gateway 2 host machine to the domain using
    `managedomain`.
5.  Create the second API Gateway instance in the same group on the API
    Gateway 2 node.
{{% alert title="Note" %}}
Do not start this instance, and do not configure API Manager for this instance in Policy Studio.
{{% /alert %}}


4.  Before starting the second API Manager-enabled instance, ensure that
    each instance has unique ports in the `envSettings.props` file. For
    example:
    1.  Edit the `envSettings.props` file for the API Gateway instance
        in the following directory:
        ```
        INSTALL_DIR/apigateway/groups/<group-n>/<instance-m>/conf/envSettings.props
        ```
    2.  Add the API Manager ports. For example, the defaults are:
    ```
    #API Manager Port
    env.PORT.APIPORTAL=8075

    #API Manager Traffic Port
    env.PORT.PORTAL.TRAFFIC=8065
    ```

1.  Start the second API Gateway instance. For example:
    ```
    startinstance -n "my_gw_server_2" -g "my_group"
    ```
2.  On startup, this instance receives the API Manager configuration for
    the group. It now shares the same KPS and Cassandra configuration
    and data, and uses the ports specified in the `envSettings.props`
    file.
