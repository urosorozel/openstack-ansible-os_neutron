==================================
Neutron role for OpenStack-Ansible
==================================

.. toctree::
   :maxdepth: 2

   overview.rst
   app-nuage.rst
   app-plumgrid.rst

:tags: openstack, neutron, cloud, ansible
:category: \*nix

This role will install the following Upstart services:
    * neutron-server
    * neutron-agents

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

This role supports two tags: ``neutron-install`` and ``neutron-config``

The ``neutron-install`` tag can be used to install and upgrade.

The ``neutron-config`` tag can be used to manage configuration.
