========================================
Scenario - Open Virtual Network (OVN)
========================================

Overview
~~~~~~~~

Operators can choose to utilize the Open Virtual Network (OVN) mechanism
driver instead of Linux bridges or plain Open vSwitch for the Neutron ML2
plugin. This offers the possibility to deploy virtual networks and routers
using OVN with Open vSwitch, which replaces the agent-based model used by
the aforementioned architectures. This document outlines how to set it up in
your environment.

The current implementation of OVN in OpenStack-Ansible should not be considered
production-ready and makes the following architectural assumptions:

* Each compute node will act as an OVN controller
* Each compute node is eligible to serve as an OVN gateway node

NOTE: Physical VTEP integration is not yet supported.

Recommended reading
~~~~~~~~~~~~~~~~~~~

Since this is an extension of the basic Open vSwitch scenario, it is worth
reading that scenario to get some background. It is also recommended to be
familiar with OVN and networking-ovn projects and their configuration.

* `Scenario: Open vSwitch <app-openvswitch.html>`_
* `OVN Architecture <http://www.openvswitch.org//support/dist-docs/ovn-architecture.7.html>`_
* `Networking-ovn <https://github.com/openstack/networking-ovn>`_

Prerequisites
~~~~~~~~~~~~~

* Open vSwitch >= 2.9.0

* Networking-ovn at time of writing requires neutron-lib>=1.17.0. The
  overrides described here will ensure that version is installed.

* A successful deployment of OVN requires a dedicated network
  interface be attached to the OVS provider bridge. This is not
  handled automatially and may require changes to the network
  interface configuration file.

OpenStack-Ansible user variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a group var file for your network hosts
``/etc/openstack_deploy/group_vars/network_hosts``. It has to include:

.. code-block:: yaml

  # Ensure the openvswitch kernel module is loaded
  openstack_host_specific_kernel_modules:
    - name: "openvswitch"
      pattern: "CONFIG_OPENVSWITCH"


Set the following user variables in your
``/etc/openstack_deploy/user_variables.yml``:

.. code-block:: yaml

  neutron_plugin_type: ml2.ovn

  neutron_plugin_base:
    - networking_ovn.l3.l3_ovn.OVNL3RouterPlugin

  neutron_ml2_drivers_type: "vlan,local,geneve"

  # Typically this would be defined by the os-neutron-install
  # playbook. The provider_networks library would parse the
  # provider_networks list in openstack_user_config.yml and
  # generate the values of network_types, network_vlan_ranges
  # and network_mappings. network_mappings would have a
  # different value for each host in the inventory based on
  # whether or not the host was metal (typically a compute host)
  # or a container (typically a neutron agent container)
  #
  # When using OVN w/ Open vSwitch, we override it to take into account
  # the Open vSwitch bridge we are going to define outside of
  # OpenStack-Ansible plays. All segmentation id ranges can be tweaked
  # to suit the environment. VXLAN networks are not directly supported.

  # When configuring Neutron to support only geneve tenant networks and
  # vlan provider networks the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "geneve"
    network_geneve_ranges: "1:1000"
    network_vlan_ranges: "vlan"
    network_mappings: "vlan:br-provider"

  # When configuring Neutron to support only vlan tenant networks and
  # vlan provider networks the configuration may resemble the following:
  neutron_provider_networks:
    network_types: "vlan"
    network_vlan_ranges: "vlan:102:199"
    network_mappings: "vlan:br-provider"

  repo_build_upper_constraints_overrides: [neutron-lib>=1.17.0]

The overrides are instructing Ansible to deploy the OVN mechanism driver and
associated OVN components. This is done by setting ``neutron_plugin_type``
to ``ml2.ovn``.

The ``neutron_plugin_base`` override instructions Neutron to use OVN for
routing functions rather than the standard L3 agent model.

The ``neutron_ml2_drivers_type`` override provides support for all type
drivers supported by OVN.

Open Virtual Network (OVN) commands
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following commands can be used to provide useful information about...

The ``ovs-vsctl list open_vswitch`` command provides information about the
``open_vswitch`` table in the local Open vSwitch database:

.. code-block::

  root@aio1:~# ovs-vsctl list open_vswitch
  _uuid               : 855c820b-c082-4d8f-9828-8cab01c6c9a0
  bridges             : [37d3bd82-d436-474e-89b7-705aea634d7d, a393b2f6-5c3d-4ccd-a2f9-e9817391612a]
  cur_cfg             : 14
  datapath_types      : [netdev, system]
  db_version          : "7.15.1"
  external_ids        : {hostname="aio1", ovn-bridge-mappings="vlan:br-provider", ovn-encap-ip="172.29.240.100", ovn-encap-type="geneve,vxlan", ovn-remote="tcp:172.29.236.100:6642", rundir="/var/run/openvswitch", system-id="11af26c6-9ec1-4cf7-bf41-2af45bd59b03"}
  iface_types         : [geneve, gre, internal, lisp, patch, stt, system, tap, vxlan]
  manager_options     : []
  next_cfg            : 14
  other_config        : {}
  ovs_version         : "2.9.0"
  ssl                 : []
  statistics          : {}
  system_type         : ubuntu
  system_version      : "16.04"

The ``ovn-sbctl show`` command provides information related to southbound
connections. If used outside the ovn_northd container, specify the
connection details:

.. code-block::

  root@aio1-neutron-ovn-northd-container-57a6f1a9:~# ovn-sbctl show
  Chassis "11af26c6-9ec1-4cf7-bf41-2af45bd59b03"
      hostname: "aio1"
      Encap vxlan
          ip: "172.29.240.100"
          options: {csum="true"}
      Encap geneve
          ip: "172.29.240.100"
          options: {csum="true"}

  root@aio1:~# ovn-sbctl --db=tcp:172.29.236.100:6642 show
  Chassis "11af26c6-9ec1-4cf7-bf41-2af45bd59b03"
      hostname: "aio1"
      Encap vxlan
          ip: "172.29.240.100"
          options: {csum="true"}
      Encap geneve
          ip: "172.29.240.100"
          options: {csum="true"}

The ``ovn-nbctl show`` command provides information about networks known
to OVN and demonstrates connectivity between the northbound database
and neutron-server.

.. code-block::

  root@aio1-neutron-ovn-northd-container-57a6f1a9:~# ovn-nbctl show
  switch 5e77f29e-5dd3-4875-984f-94bd30a12dc3 (neutron-87ec5a05-9abe-4c93-89bd-c6d40320db87) (aka testnet)
      port 65785045-69ec-49e7-82e3-b9989f718a9c
          type: localport
          addresses: ["fa:16:3e:68:a3:c8"]

The ``ovn-nbctl list Address_Set`` command provides information related to
security groups. If used outside the ovn_northd container, specify the
connection details:

.. code-block::

  root@aio1-neutron-ovn-northd-container-57a6f1a9:~# ovn-nbctl list Address_Set
  _uuid               : 575b3015-f83f-4bd6-a698-3fe67e43bec6
  addresses           : []
  external_ids        : {"neutron:security_group_id"="199997c1-6f06-4765-89af-6fd064365c6a"}
  name                : "as_ip4_199997c1_6f06_4765_89af_6fd064365c6a"

  _uuid               : b6e211af-e52e-4c59-93ce-adf75ec14f46
  addresses           : []
  external_ids        : {"neutron:security_group_id"="199997c1-6f06-4765-89af-6fd064365c6a"}
  name                : "as_ip6_199997c1_6f06_4765_89af_6fd064365c6a"

  root@aio1:~# ovn-nbctl --db=tcp:172.29.236.100:6641 list Address_Set
  _uuid               : 575b3015-f83f-4bd6-a698-3fe67e43bec6
  addresses           : []
  external_ids        : {"neutron:security_group_id"="199997c1-6f06-4765-89af-6fd064365c6a"}
  name                : "as_ip4_199997c1_6f06_4765_89af_6fd064365c6a"

  _uuid               : b6e211af-e52e-4c59-93ce-adf75ec14f46
  addresses           : []
  external_ids        : {"neutron:security_group_id"="199997c1-6f06-4765-89af-6fd064365c6a"}
  name                : "as_ip6_199997c1_6f06_4765_89af_6fd064365c6a"

Additional commands can be found in upstream OVN documentation.

Notes
~~~~~

The ``ovn-controller`` service on compute nodes will check in as an agent
and can be observed using the ``openstack network agent list`` command:

.. code-block::

  root@aio1-utility-container-35bebd2a:~# openstack network agent list
  +--------------------------------------+------------------------------+------+-------------------+-------+-------+----------------+
  | ID                                   | Agent Type                   | Host | Availability Zone | Alive | State | Binary         |
  +--------------------------------------+------------------------------+------+-------------------+-------+-------+----------------+
  | 4db288a6-8f8a-4153-b4b7-7eaf44f9e881 | OVN Controller Gateway agent | aio1 | n/a               | :-)   | UP    | ovn-controller |
  +--------------------------------------+------------------------------+------+-------------------+-------+-------+----------------+

The HAproxy client and server timeout values have been increased from
50 seconds to 90 minutes for all load-balanced OVN-related services.

The HAproxy implementation in use may not properly handle active/backup
failover for ovsdb-server with OVN. Work may be done to implement
pacemaker/corosync or wait for active/active support.

Warranty
~~~~~~~~

This implementation of OVN is not supported and should be considered
only for development purposes. The architecture within OSA is subject
to change. Reviews and suggestions are welcome.
