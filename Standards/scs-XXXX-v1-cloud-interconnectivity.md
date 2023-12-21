---
title: Cloud Interconnectivity
type: Decision Record
status: Draft
track: IaaS
---

<!---
This is a template striving to provide a starting point for
creating a standard or decision record adhering to scs-0001.
Replace at least all text which is _italic_.
See https://github.com/SovereignCloudStack/standards/blob/main/Standards/scs-0001-v1-sovereign-cloud-standards.md
--->

## Introduction

This document outlines a possible implementation of Cloud interconnection by creating a standard and a reference
implementation of a modular service called Cloud Interconnect Service (CIS), which enables connection between
clouds using L2 routing or IPSec VPN. By connecting different networks and hiding the complexity from the user enables
an efficient, secure and open digital ecosystem by addressing issues related to data exchange and sovereignty,
vendor lock-in, as well as cloud service dependencies. A Lot of the benefits and reason can be read in the GAIA-X
document attached at the end.

## Motivation

The motivation behind enabling simple interconnection within the Gaia-X and cloud in general
is driven by the necessity for secure and efficient data exchange in digital ecosystems.
In a world where we can provide an interconnection between clouds from various vendors and user's
own on-premise or edge infrastructure, users can choose services best to fit their needs and have everything
seamlessly working together.

![img.png](img.png)
(Image from The-role-and-the-importance-of-Interconnection-in-Gaia-X)

In the figure above, we can see the ideal situation where a user can buy front-end services from cloud1 and
back-end services from cloud2 and have everything working together.

## Design Considerations

### Use cases

#### MVP

Following use cases will be the goal of MVP. Below, there are also future use cases that were in mind while
implementing this solution.

##### Cross SCS Cloud

![img_1.png](img_1.png)

In this use case, we want to be able to connect two different SCS Clouds over the internet via VPN.

* I as an SCS Consumer want to be able to connect my networks between two clouds.

##### Cross Regions

![img_6.png](img_6.png)

We want to connect user applications across cloud regions.

* I as an SCS Consumer want to be able to connect my networks between two regions within one cloud.

##### Cross Tenants

![img_7.png](img_7.png)

Sometimes a user can obtain Services from another company operating within the same cloud.

* I as an SCS Consumer want to be able to connect my networks between two tenants within one cloud.

##### From cloud to Cloud Provider legacy infrastructure

![img_8.png](img_8.png)

A cloud provider can have customers from before they started their Cloud offering and could want to provide
network access inside their cloud from legacy VMs or customer's bare metal nodes hosted at the Cloud Provider.

* I as an SCS Consumer want to be able to connect my cloud network with other services provided by Cloud Provider.

##### On Premise Infrastructure

![img_9.png](img_9.png)

* I as an SCS Consumer want to be able to connect my cloud network with my existing on-premise infrastructure.

#### Future use cases

##### Cross SCS Cloud via private interconnect

![img_2.png](img_2.png)

In this use case, we want to be able to connect two different SCS Clouds over a private connection.

* I as an SCS Consumer want to be able to connect my networks using private connections via Exchange operators.

##### Cross SCS Clouds via private interconnect without Point-to-Point VPN

![img_4.png](img_4.png)

In this use case, we have a MPLS interconnect from internet exchange. As the number of Point-to-Point VPN
connections increases in accordance [Metcalfe’s Law](https://en.wikipedia.org/wiki/Metcalfe%27s_law) user can
quite quickly run into really large number of connections, in case we trust the operator and don't need
the extra encryption from VPN. We can connect the MPLS connection directly to openstack.

* I as an SCS Consumer want to be able to connect my networks using MPLS in a highly scalable manner.

##### Cross SCS Cloud via private interconnect with fail-over

![img_5.png](img_5.png)

In this use case, we have a connection via private interconnect, and we would like to ensure that in case of a
failure, a backup connection via the internet would be established instead.

* I as an SCS Consumer want to be able to connect my networks using private connection with fail over the internet.

### Technical and security considerations

* The solution should be as much self-service as possible and should not require any intervention from the cloud
  provider.
* As the service will be exposed to public internet security and rate limiting,
  has to be a main factor while designing the solution.
* The solution should be able to run in non-SCS or at least provide option for configuration without it to enable use
  cases other than inside or two SCS Clouds.
* As there is more than one service with the routing information originating from the respective clouds, decentralized
  data synchronization is important.
* New subnets should propagate automatically, and it should be possible to set static manual subnets as well.
* It would be nice for the solution to be modular to enable automatic configuration of on-premise gateways or
  hyperscalers.
* Inspiration can be taken from currently available solutions from hyperscalers such
  as [AWS transit Gateway](https://aws.amazon.com/transit-gateway/)
  or [Azure VPN Gateway](https://learn.microsoft.com/en-us/azure/vpn-gateway/tutorial-create-gateway-portal)
* The solution should be as closely following current Gaia-X proposals as it is possible while focusing on a working
  solution first. (See [Related documents](#related-documents))
* There should be an option for having to manually approve any change in VPN if enabled.
* The solution should use similar technologies as used in OpenStack to be easy to adapt.
* It has to be capable of running in Multi-tenant mode as to save resources for cloud Provider.
* As we already need an L2 connection for establishing VPN via private connection, we would also want to utilize just
  for pure L3 as mentioned in the use
  case [Cross SCS Clouds via private interconnect without Point-to-Point VPN](#cross-scs-clouds-via-private-interconnect-without-point-to-point-vpn)
* In the future, in cases where a private connection can be established via self-service API to be able to create this
  connection on the fly without the need of manual intervention from the side of the Cloud Operator.
* The service has to be configured via API, ideally via the openstack client as well.
  * User frontend or integration with the horizon would also be great.

## Proposed solution

![img_10.png](img_10.png)

According to the stated requirements, the Django framework seems an ideal tool, given its extensive
ecosystem. This is complemented by Horizon's use of Django, making authentication and authorization processes more
straightforward, as they are already implemented. Most Openstack services use MariaDB for their database backend along
with Redis and RabbitMQ for message brokering. Openstack has used the in-house Mitaka library for asynchronous tasks,
but the use of Celery may prove more efficient outside of Openstack.

### User flow

Imagine a user with an empty openstack project in two different clouds and the CIS service installed. Here's a flowchart
for connecting these two projects; italicized actions indicate the new service and `1:` or `2:` indicate which openstack
is handling the action.

#### First bootstrap

1. 1: User creates a new network. - *Custom Neutron plugin is activated by network creation, initiating a CIS1 API call,
   which spawns an asynchronous task to manifest a new network object.*
2. 1: User creates a new subnet. - *Custom Neutron plugin is activated by subnet creation and registers it with CIS1.*
3. 1: User creates a new CIS Node - *Following node creation, an invitation JWT token is created.*
4. 2: User creates a new CIS Node with the invitation token - *CIS2 based on the url from the JWT token tries to
   establish a handshake, this handshake is bound between the openstack projects for given tenant.*
5. 1: User creates a new CIS Connected group.
6. 1: User associates network with the CIS Connected group (Marked with share all subnets.). *As there is just single
   network and no other nodes associated nothing happens, just CIS1 creates a VPN as a Service, Service connected to the
   network router.*
7. 1: User associates CIS Node with CIS Connected group. *Calls CIS2 to create new CIS Connected Group invite. If the
   second CIS has set trust policy, this invitation is accepted.*
8. (2: User accepts CIS Connected group). *CIS1 shares its network and subnet configuration together with version
   number.*
9. 2: User creates a new network. - *Neutron plugin registers it with CIS2.*
10. 2: User creates a new subnet. - *Neutron plugin registers it with CIS2.*
11. 2: User associates network with the CIS Connected group (Marked with share all subnets.).
    1. *CIS2 creates a VPN as a Service, Service connected to the network router.*
    2. *CIS2 calls CIS1 and updates the network metadata for CIS2.*
    3. *Simple leader election based on the id of the node happens to see who whill set up the VPN connections (Bigger
       uuid wins). CIS1 Wins*
    4. *CIS1 compares media set by Cloud providers in CIS1 and CIS2 and tries to find a match.*
    5. *A match for IPSec over public internet is found.*
    6. *CIS1 creates compatible IKEPOLICY and IPSECPOLICY*
    7. *CIS1 creates a new endpoint group with CIDRs of CIS2 subnets.*
    8. *CIS1 creates a new endpoint group with subnets of CIS1.*
    9. *CIS1 creates a new VPN connection.*
    10. *CIS1 updates its network metadata and shares it with CIS2*
    11. *CIS2 loads metadata from CIS1*
    12. *CIS2 creates IKEPOLICY and IPSEC policy from CIS1*
    13. *CIS2 creates a new endpoint group with CIDRs of CIS1 subnets.*
    14. *CIS2 creates a new endpoint group with subnets of CIS2.*
    15. *CIS2 creates a new VPN connection.*
    16. *Neutron plugin on CIS1 updates connection state to up.*
    17. *Neutron plugin on CIS2 updates connection state to up.*
12. Everything is now working.

Networking setup on node1

```bash
[user@os1]$ openstack network create left-network
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   | None                                 |
| availability_zones        | None                                 |
| created_at                | 2018-12-27T10:49:17Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | e4f43f87-3b31-41e4-9803-8e10edd3167e |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| location                  | None                                 |
| mtu                       | None                                 |
| name                      | left-network                         |
| port_security_enabled     | True                                 |
| project_id                | 70061ce0cd2e47ef9d7dc82174dc9923     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 2                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-12-27T10:49:17Z                 |
+---------------------------+--------------------------------------+
[user@os1]$ openstack subnet create left-subnet \
  --network left-network \
  --subnet-range 10.1.0.0/24 \
  --gateway 10.1.0.1 \
  --dns-nameserver 8.8.8.8
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.1.0.2-10.1.0.254                  |
| cidr              | 10.1.0.0/24                          |
| created_at        | 2018-12-27T13:00:49Z                 |
| description       |                                      |
| dns_nameservers   | 8.8.8.8                              |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.1.0.1                             |
| host_routes       |                                      |
| id                | 38346388-4b09-4f0a-a3d1-b1a5f6587f4c |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| location          | None                                 |
| name              | left-subnet                          |
| network_id        | e4f43f87-3b31-41e4-9803-8e10edd3167e |
| project_id        | 70061ce0cd2e47ef9d7dc82174dc9923     |
| revision_number   | 2                                    |
| segment_id        | None                                 |
| service_types     | None                                 |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2018-12-27T13:00:49Z                 |
+-------------------+--------------------------------------+
[user@os1]$ openstack router create left-router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints | None                                 |
| availability_zones      | None                                 |
| created_at              | 2018-12-27T13:13:16Z                 |
| description             |                                      |
| distributed             | None                                 |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | None                                 |
| id                      | c971c888-a0bb-47e3-a922-565899c9f090 |
| location                | None                                 |
| name                    | left-router                          |
| project_id              | 70061ce0cd2e47ef9d7dc82174dc9923     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2018-12-27T13:13:16Z                 |
+-------------------------+--------------------------------------+
[user@os1]$ openstack router add subnet left-router left-subnet
[user@os1]$ openstack router set left-router --external-gateway ext-net
```

Networking setup on node2

```bash
[user@os2]$ openstack network create right-network
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   | None                                 |
| availability_zones        | None                                 |
| created_at                | 2018-12-27T13:53:16Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 46e614d1-baaa-46cf-8e4c-c96fe63fecf2 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | None                                 |
| is_vlan_transparent       | None                                 |
| location                  | None                                 |
| mtu                       | None                                 |
| name                      | right-network                        |
| port_security_enabled     | True                                 |
| project_id                | 70061ce0cd2e47ef9d7dc82174dc9923     |
| provider:network_type     | None                                 |
| provider:physical_network | None                                 |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 2                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-12-27T13:53:16Z                 |
+---------------------------+--------------------------------------+
[user@os2]$ openstack subnet create right-subnet  \
--network right-network \
--subnet-range 10.2.0.0/24 \
--gateway 10.2.0.1
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 10.2.0.2-10.2.0.254                  |
| cidr              | 10.2.0.0/24                          |
| created_at        | 2018-12-27T13:56:54Z                 |
| description       |                                      |
| dns_nameservers   |                                      |
| enable_dhcp       | True                                 |
| gateway_ip        | 10.2.0.1                             |
| host_routes       |                                      |
| id                | a1026c99-8dd6-496a-a565-74a49f2e95ec |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| location          | None                                 |
| name              | right-subnet                         |
| network_id        | 46e614d1-baaa-46cf-8e4c-c96fe63fecf2 |
| project_id        | 70061ce0cd2e47ef9d7dc82174dc9923     |
| revision_number   | 2                                    |
| segment_id        | None                                 |
| service_types     | None                                 |
| subnetpool_id     | None                                 |
| tags              |                                      |
| updated_at        | 2018-12-27T13:56:54Z                 |
+-------------------+--------------------------------------+
[user@os2]$ openstack router create right-router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints | None                                 |
| availability_zones      | None                                 |
| created_at              | 2018-12-27T13:57:47Z                 |
| description             |                                      |
| distributed             | None                                 |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | None                                 |
| id                      | 56f95788-1c34-432f-8ad6-f304776221a2 |
| location                | None                                 |
| name                    | right-router                         |
| project_id              | 70061ce0cd2e47ef9d7dc82174dc9923     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2018-12-27T13:57:47Z                 |
+-------------------------+--------------------------------------+
```

CIS Setup on openstack 1

```bash
[user@os1]$ openstack cis node create right
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| created_at                | 2018-12-27T10:49:17Z                 |
| id                        | e4f43f87-3b31-41e4-9803-8e10edd3167e |
| name                      | right                                |
| status                    | INVITE_PENDING                       |
| trust                     | ACCEPT_CONNECTIONS                   |
| tags                      |                                      |
| updated_at                | 2018-12-27T10:49:17Z                 |
| last_online               | 2018-12-27T10:49:17Z                 |
| host                      |                                      |
| invite_token              | eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c |
+---------------------------+--------------------------------------+
[user@os1]$ openstack cis group create my-group
[user@os1]$ openstack cis group add-node my-group right
[user@os1]$ openstack cis group add-network my-group left-network
```

CIS Setup on openstack 2

```bash
[user@os2]$ openstack cis node create left --invite_token eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| created_at                | 2018-12-27T10:49:17Z                 |
| id                        | d418198c-99af-4e3c-8e99-69bbd2ae9b09 |
| name                      | left                                 |
| status                    | PAIRED                               |
| trust                     | ACCEPT_CONNECTIONS                   |
| tags                      |                                      |
| updated_at                | 2018-12-27T10:49:17Z                 |
| last_online               | 2018-12-27T10:49:17Z                 |
| host                      | os2.foo.bar                          |
| invite_token              | SECRET                               |
+---------------------------+--------------------------------------+
[user@os2]$ openstack cis group add-network my-group right-network
```

Now depending on the cloud provider setup the connection will be formed automatically.
On the openstack2 we didn't need to create my-group is it was already created in openstack1 and openstack2 was already
associated with this group.

#### Provider installation

The provider setup would look like this:

1. Cloud Provider installs the CIS service. — Run ansible.
2. Install the `cis-os-cli` package for your openstack client.
3. Install the VPN as a Service plugin to neutron. [Guide](https://docs.openstack.org/neutron/2023.2/admin/vpnaas-scenario.html)
4. Enable CIS neutron service plugin:
   1. Install the python library: `cis-neutron`
   2. Add `cis` to `service_plugin` in `/etc/neutron/neutron.conf`
5. Create a service account for CIS service
   1. Register service:
      ```bash
      $ openstack service create --name cis --description "Cloud Interconnectivity Service" interconnect
      +-------------+----------------------------------+
      | Field       | Value                            |
      +-------------+----------------------------------+
      | description | Cloud Interconnectivity Service  |
      | enabled     | True                             |
      | id          | 84c23f4b942c44c38b9c42c5e517cd9a |
      | name        | cis                              |
      | type        | interconnect                     |
      +-------------+----------------------------------+
      ```
   2. Register endpoint
      ```bash
      $ openstack endpoint create cis public http://example.com/cis/v2.1
      +--------------+----------------------------------+
      | Field        | Value                            |
      +--------------+----------------------------------+
      | enabled      | True                             |
      | id           | c219aa779e90403eb4a78cf0aa7d38b1 |
      | interface    | public                           |
      | region       | None                             |
      | region_id    | None                             |
      | service_id   | 0f5da035b8e94629bf35e7ec1703a8eb |
      | service_name | cis                              |
      | service_type | interconnect                     |
      | url          | http://example.com/cis/v2.1      |
      +--------------+----------------------------------+
      ```
   3. Create service user
      ```bash
      $ openstack user create cis --password Sekr3tPass
      +---------------------+----------------------------------+
      | Field               | Value                            |
      +---------------------+----------------------------------+
      | domain_id           | default                          |
      | enabled             | True                             |
      | id                  | 95ec3e1d5dd747f5a512d261731d29c7 |
      | name                | cis                              |
      | options             | {}                               |
      | password_expires_at | None                             |
      +---------------------+----------------------------------+
      ```
   4. Add admin privileges to the user created. This is needed for users to be able to authenticate with the service.
      ```bash
      $ openstack role add --project service --user nova admin
      +-------+----------------------------------+
      | Field | Value                            |
      +-------+----------------------------------+
      | id    | 233109e756c1465292f31e7662b429b1 |
      | name  | admin                            |
      +-------+----------------------------------+
      ```
6. Update CIS configuration in `/etc/cis/config.yaml` to have the created credentials.
7. Assuming we have a provider network called `ext-net` [Guide](https://docs.openstack.org/install-guide/launch-instance-networks-provider.html)
8. Register media with CIS
    ```bash
    $ openstack cis media create internet_connection --network ext-net --asn <Your public ASN or 0 for public internet> --location fr
    ```
    Here we have two variants based on the opened questions, if the scope is limited, the command above is enough else we need to create another media on top.
    ```bash
    $ openstack cis media create ipsec_internet --sub_media internet_connection --location fr --type virtual --media_type ipsec
    $ openstack cis media create wireguard_internet --sub_media internet_connection --location fr --type virtual --media_type wireguard
    ```

#### Adding Cross Region media

```bash
$ openstack cis media create back_bone_reg1 --network back_bone_reg1 --asn <Your private ASN> --region reg1
$ openstack cis media create back_bone_reg2 --network back_bone_reg2 --asn <Your private ASN> --region reg2
```

#### Adding private interconnect between two clouds

```bash
[user@os1]$ openstack cis media create private_connection --network prv_con_scs --asn <Connection private ASN> --tennat_id f3dcb5ba-4d27-4450-83b4-3b92f6ca2e9e
[user@os2]$ openstack cis media create private_connection --network scs_prv_con --asn <Connection private ASN> --tennat_id 5f068cd7-5412-45ba-a0f1-873a07c94aeb
```

Similarly, as with the public setup, this could be enough, or we could create either a VPN or routed connection (in case of MPLS) on top, this could be created by the customer itself.

```bash
[user@os1]$ openstack cis media create ipsec_connection --sub_media private_connection --type virtual --media_type ipsec
[user@os2]$ openstack cis media create ipsec_connection --sub_media private_connection --type virtual --media_type ipsec
```

### Components

#### Server

This server is geared towards handling API calls both internal and between CIS, as well as managing changes in the
database. It could also offer a user interface for simpler visualization of the connection, easier debugging. It could
also be used as a base for implementation of this user interface functionality inside Horizon.

#### Database

MariaDB is often used in the OpenStack context, but Django offers flexibility in database backends, ranging from SQLite
to PostgreSQL.

#### Celery / Mitaka

Either Celery, which is highly compatible outside OpenStack and used for asynchronous and periodic processing tasks, or
Mitaka, an in-house OpenStack built library, can be used.

Some of the tasks that will be run:

* Periodic synchronization of states between nodes.
* Handling of incoming node synchronization and republishing.
* Handling of state changes from Neutron plugin.
* Running Backend specific code, which can be long-running, such as running of OpenToFu.

#### Redis

This is used as a message broker for task processing and periodic task execution.

#### Custom Neutron plugin

This plugin enables automatic reporting of the user resource state changes back to the CIS service.

See details here:
https://docs.openstack.org/neutron/latest/contributor/internals/plugin-api.html#neutron.neutron_plugin_base_v2.NeutronPluginBaseV2.start_rpc_state_reports_listener

Basically the neutron when it sees changes, it will tell the CIS service API using the OpenStack neutron service token.

### Health checker

This is a potential improvement for instances where CIS operates outside OpenStack and states changes are not
automatically reported. It could also benefit OpenStack due to additional services such as latency and speed monitoring.

### CIS Objects / CIS Database

![img_11.png](img_11.png)

#### Backends

Backends enable the modular basis of the service. The objects in the diagram above are generic models that
can be extended as needed to support the needs of the backend. These backends will be installed by the Cloud Provider
and can be limited to certain users, regions or projects. In the backend storage, there are also credentials for the
interaction there it is possible from single instance to control multiple OpenStacks. Backend can be enabled just for
certain region, tenant or tenant project.

Possible backends:

* OpenStack - includes endpoint for neutrino plugin reporting
* Manual — just shows the IPSec configuration for the user to create and keep the configuration up to date by
  themselves.
  This is meant for implementation in network switches and so on.
* Ansible — This backend is meant for on-premise or legacy installations where the updates and provisioning of the VPN
  service are done by ansible.
* openTofu - openTofu backend in any of the possible supported combinations.

#### Tenant

This object provides mapping to openstack tenants.

#### Node

This object handles handshake and communication between nodes. Here might be implemented some measures to stop
man-in-the-middle attacks and so on such as creating some kind of fingerprints and certificates.
On this object, we also define a trust policy, in case we trust given node, we accept invites to connected groups
from this node automatically.

#### Media

This object basically maps
GAIA-X [Interconnection Point Identifiers](https://docs.gaia-x.eu/technical-committee/technical-committee/architecture-document/latest/conceptual_model/#interconnection-point-identifiers).
It also has some parameters as inspired by GAIA-X and are there for future GAIA-X support.
It is used to map possible connections between clouds, this objects can be nested, and the most specific (highest layer)
wins.
The matching occurs based on the layer and the ASN, if the ASN is public it is considered a connection through internet.
The same applies if this field is 0 or None,
If it is private, it is assumed that this is ASN of a transit connection such as inter-cloud or private connection.
When nested, the top one can be an IPSec connection. If the CIS finds more than one possible match, it will be used to
create multiple connections with fail over.

For the purpose of matching during metadata synchronization, this object is also created in each related CIS.

#### Network

This object is a mapping of the openstack network object, same as media during metadata synchronization, this object is
also created in each related CIS.
Here we also define the `share_policy` in case it is set to all, all subnets will be propagated to related connections.
If
set to select,
you have to enable propagation of given subnet in the subnet CIDR.

#### Subnet CIDR

This object is a mapping of the openstack subnet object it can also be created manually to provide manual static routes.
During synchronization, this object is also created in each related CIS.

#### GroupNode

This is basically an instance of the Node within the ConnectedGroup as the node object can be reused for multiple
Connected groups. Also, an object for the node of the CIS that is currently running is created to have a
metadata version.

#### Connected Group

This is a grouping of all nodes that will be automatically connected.
Here we can also find a `trust_policy` which is enabled all the connections that are initiated from other nodes to the
local CIS are automatically accepted, otherwise user has to manually confirm them.
Also on the ConnectedGroup we define if the local node is Hub or Spoke (This is copied to the local `GroupNode` object.)
In case the local node is Hub, according to the leader election between two corresponding CIS, it will initiate
connections to all other Hubs. In case it is set to Spoke only connections between Hub and Spoke will be initiated and
no connections between individual Spokes. In case you want to create extra connections in for spokes, you can create
Connection objects manually.

#### Connection

This object is a representation of connection that can be implemented in various ways according to the backed used.
So far, only two possibilities are on the radar:

* IPSec
* L2 Routing - Assumes existing L2 connection and just publishing of subnets to routers.

In the future, this or Media object could be a base to set up MPLS automatically in internet exchanges.

It is also possible to set failover in case the CIS notices that the state of the connection is down, it will try to
change to another connection and revert as soon as the connection is restored.

### Leader Election

When a change happens in the network, either by adding a new network to a Node or adding additional media to the node, a
process takes place that updates all edges between the nodes.
As this is a distributed system, a leader election has to happen. This leader election happens on every edge, and is
quite simple as it is a process between just two nodes, it is fairly easy, and the other node does not even have to be
online. Basically, the node with the higher numerical value of uuid wins and has to take care of creation and deletion
of
the appropriate connections.

### Metadata synchronization process

To avoid the issues of the distributing system, we try to limit the possible pain points. One of the ways is the leader
election happening just between the given-edge nodes. The other is this metadata synchronization process, as metadata it
is understood the Connected Group related objects such as `Media`, `Network`, `Connection`, `Subnet`. To have
this data consistent, each node has their own metadata version number, this version number is related just to the data
of the given node. When any of this metadata changes, the following process ensues:

1. Raise the version number of the current node.
2. Prepare a JSON of state of all nodes data stored in the given node.
3. Send this to all nodes.

When a node receives an update, it follows these steps:

1. Check if a local version of any of the nodes is lower to the ones in the update.
   1. If no doesn't do anything.
   2. If yes, update local data and do the same as if would local state change.

## Open questions

### Failure to communicate between nodes when the public internet is down

There is an inherent design fail, as the communication between the CIS usually happens over the public internet,
when one of the nodes loses internet connection, user will not be able to update its configuration.

I have a possible solution. In a way that maybe the Health checker could be used as HTTP proxy. In case the public
internet fails, it could be possible to tunnel the communication between CIS through the health checker and through the
connection itself. So this would mean that as long as there is some form of communication, it would be possible to do
configuration changes.

### Dynamic routing

Currently, all routings are done in a mesh configuration using static routes, except when the bare L2 MPLS connection
would be used as mentioned above. It could be beneficial to utilize some form of BGP to make it possible that even when
connection between two nodes fails, the communication could still flow through other nodes.

### VPN over VPN?

Should the media be stackable? Should a user be able to create a media on top of another media to create nested VPNs?


### Support for more then just one kind of VPNs

Example in [Provider installation](#provider-installation) and [Addding private interconnect between two clouds](#adding-private-interconnect-between-two-clouds)

It might be a good idea not only as GAIA-X describes a media but to have another layer on top to differentiate between
physical networks and any routing actions or VPNs on top of it. This could be coming out of the GAIA-X we could have
a kind of physical media which would be basically anything that would be connected to openstack cloud this could an
interconnect or VLAN. On top of it linked to it would be another layer called virtual media. This layer could be of
the type VPN_IPSEC or VPN_WIREGUARD or routed for MPLS routing as seen in the use case [Cross SCS Clouds via private interconnect without Point-to-Point VPN](#cross-scs-clouds-via-private-interconnect-without-point-to-point-vpn).

## Decision

Decision

## Related Documents

* [The-role-and-the-importance-of-Interconnection-in-Gaia-X](https://gaia-x.eu/wp-content/uploads/2022/05/The-role-and-the-importance-of-Interconnection-in-Gaia-X_13042022.pdf)
* [Gaia-X Architecture Document - 4.4 Interconnection Point Identifiers](https://docs.gaia-x.eu/technical-committee/technical-committee/architecture-document/latest/conceptual_model/#interconnection-point-identifiers)
* [Create a VPN (using VPNaaS)](https://docs.syseleven.de/syseleven-stack/en/howtos/vpnaas)
* [Horizon Policy Enforcement (RBAC: Role Based Access Control) — horizon 23.4.0.dev76 documentation](https://docs.openstack.org/horizon/latest/contributor/topics/policy.html)
* [horizon/openstack_auth at master · openstack/horizon · GitHub](https://github.com/openstack/horizon/tree/master/openstack_auth)
* [Create and manage services and service users — keystone 24.1.0.dev15 documentation](https://docs.openstack.org/keystone/latest/admin/manage-services.html)
* [Networking API v2.0 — Networking API Reference documentation](https://docs.openstack.org/api-ref/network/v2/index.html#vpnaas-2-0-vpn-vpnservices-ikepolicies-ipsecpolicies-endpoint-groups-ipsec-site-connections)
* [GitHub - salv-orlando/hdn: The neutron HDN plugin!](https://github.com/salv-orlando/hdn)
* [How to write a Neutron Plugin - if you really need to | PPT](https://www.slideshare.net/salv_orlando/how-to-write-a-neutron-plugin-if-you-really-need-to)
* [Neutron/VPNaaS – OpenStack](https://wiki.openstack.org/wiki/Neutron/VPNaaS)
* [OpenStack Docs: Networking (neutron) concepts](https://docs.openstack.org/neutron/pike/install/concepts.html)
* [Provider network — Installation Guide documentation](https://docs.openstack.org/install-guide/launch-instance-networks-provider.html)
* [AWS transit Gateway](https://aws.amazon.com/transit-gateway/)
* [Azure VPN Gateway](https://learn.microsoft.com/en-us/azure/vpn-gateway/tutorial-create-gateway-portal)
* [[Research] Security in Neutron service](https://input.scs.community/r4fOkAyaRs6CocUSaIRgVg)
