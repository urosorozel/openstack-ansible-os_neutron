os_neutron Role Docs
====================

The os_neutron role is used to to deploy, configure and install OpenStack
Networking.

This role will install the following:
    * neutron-server
    * neutron-agents

Basic Role Example
^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    - name: Installation and setup of Neutron
      hosts: neutron_all
      user: root
      roles:
        - { role: "os_neutron", tags: [ "os-neutron" ] }
      vars:
        neutron_galera_address: "{{ internal_lb_vip_address }}"
