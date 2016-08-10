========
Overview
========

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

This role supports two tags: ``neutron-install`` and
``neutron-config``. The ``neutron-install`` tag can be used to install
and upgrade. The ``neutron-config`` tag can be used to maintain the
configuration of the service.
