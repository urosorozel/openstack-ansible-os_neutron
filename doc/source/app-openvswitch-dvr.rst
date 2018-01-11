======================================
Scenario - Using Open vSwitch with DVR
======================================

Overview
~~~~~~~~

Operators can choose to utilize Open vSwitch with Distributed Virtual
Routing (DVR) instead of Linux Bridges or plain Open vSwitch for the
neutron ML2 agent. This offers the possibility to deploy virtual routing
instances outside the usual neutron networking node. This document
outlines how to set it up in your environment.

Recommended reading
~~~~~~~~~~~~~~~~~~~

We recommend that you read the following documents before proceeding:

 * Neutron documentation on Open vSwitch DVR OpenStack deployments:
   `<https://docs.openstack.org/neutron/latest/admin/deploy-ovs-ha-dvr.html>`_
 * Blog post on how OpenStack-Ansible works with Open vSwitch:
   `<http://trumant.github.io/openstack-ansible-dvr-with-openvswitch.html>`_

Prerequisites
~~~~~~~~~~~~~

Configure your networking according the Open vSwitch setup:

* Scenario - Using Open vSwitch
  `<https://docs.openstack.org/openstack-ansible-os_neutron/latest/app-openvswitch.html>`_

OpenStack-Ansible user variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set the following user variables in your
``/etc/openstack_deploy/user_variables.yml``:

.. code-block:: yaml

  # Ensure the openvswitch kernel module is loaded
  openstack_host_specific_kernel_modules:
    - name: "openvswitch"
      pattern: "CONFIG_OPENVSWITCH"
      group: "network_hosts"

  ### neutron specific config
  neutron_plugin_type: ml2.ovs.dvr

  neutron_ml2_drivers_type: "flat,vlan"

  # Typically this would be defined by the os-neutron-install
  # playbook. The provider_networks library would parse the
  # provider_networks list in openstack_user_config.yml and
  # generate the values of network_types, network_vlan_ranges
  # and network_mappings. network_mappings would have a
  # different value for each host in the inventory based on
  # whether or not the host was metal (typically a compute host)
  # or a container (typically a neutron agent container)
  #
  # When using Open vSwitch, we override it to take into account
  # the Open vSwitch bridge we are going to define outside of
  # OpenStack-Ansible plays
  neutron_provider_networks:
    network_flat_networks: "*"
    network_types: "vlan"
    network_vlan_ranges: "physnet1:102:199"
    network_mappings: "physnet1:br-provider"

**Note:** The only difference to the Standard Open vSwitch configuration
is the setting of the ``ml2_plugin_type``.

Customization is needed to support additional network types such as vxlan,
GRE or Geneve. Refer to the `neutron agent configuration
<https://docs.openstack.org/neutron/latest/configuration/#configuration-reference>`_ for
more information on these attributes.
