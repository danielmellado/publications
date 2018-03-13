::

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

===============================
 Infrastructure Project Update
===============================

Blank Slide
-----------
.. hidetitle::

Main Title
----------
.. transition:: tilt
   :duration: 2
.. hidetitle::
.. figlet:: OpenStack Barcelona Meetup

The path to ZuulV3 within Kuryr-Kubernetes

Who am I?
---------
* Daniel Mellado
* Senior Software Engineer at Red Hat
* Kuryr PTL for Rocky cycle.

Kuryr-Kubernetes
----------------
* Kubernetes integration with OpenStack networking

  * Enables native Neutron-based networking in Kubernetes
  * Run both OpenStack VMs and Kubernetes Pods on the same Neutron network

Zuul
----
.. transition:: dissolve
.. hidetitle::

.. ansi:: zuul.ans

Zuul
----
.. transition:: pan

* Multi-cloud, scalable, elastic CI/CD engine
* In production for OpenStack

  * Also in use by Wikimedia, BMW and others

* Nodepool attaches single-use cloud slaves to Jenkins Master Pool

  * Triggers jobs on Jenkins Pool for executing job content
  * Jenkins serves no role except remote execution

Zuul
----
.. transition:: pan

* Periodic: jobs run in response to a timer
* Post: jobs run after a change
* Check: job run when someone proposes a change
* Gate: jobs run between change approval and landing

Why not Jenkins
---------------
.. transision:: pan

* Scalability
* Security

* ssh slave plugin

* Projects to try to mitigate this

  * JClouds plugin
  * Gearman Worker
  * jjb

Zuul v3
-------
.. transition:: tilt
   :duration: 2

* Replace Jenkins with Ansible
* Add github PR integration in addition to gerrit
* Jobs written and executed in Ansible
* In-repo config
* Support Bare Metal, VMs and Containers

Zuul v3 Secrets
---------------
.. transition:: pan

.. code-block::

   - secret:
       name: kolla_dockerhub_creds
       data:
         password: !encrypted/pkcs1-oaep
           - QLe52Ymma5HJg3K2kgeSEMp7TwarkH8AbEiwcnDTqZ276BUF9wrt+5gPJRfVU1BYty2lq
             CCzhawJJ09TV0WU2SEUKlicWoXQ/hcbYWNlOHVL6/gm9UxZP/GC8d1eyQfbCS7UUHfiHF
             BLAHBLAHBLAHBLAHBLAH=

   - job:
       name: kolla-publish-ubuntu-binary
       post-run: tests/playbooks/publish.yml
       secrets:
         - kolla_dockerhub_creds

   - hosts: all
     tasks:
       - name: Login to Dockerhub
         command: "docker login -u {{ kolla_dockerhub_creds.user }} -p {{ kolla_dockerhub_creds.password }}"
         no_log: true

       - shell: "for img in $(docker images  --format '{% raw %}{{ .Repository }}:{{ .Tag }}{% endraw %}' | grep kolla ); do docker push $img; done"


Zuul v3 Branches
---------------
.. transition:: pan

* Jobs in branched repos get *implied branch matchers*
* Jobs in branchless repos have no implied branch matchers
* Any job can set *explicit branch matchers*
* We're backporting some jobs to stable branches now
* In the future, branching should just DTRT

Zuul v3 GitHub
--------------
.. transition:: pan

* OpenStack's Zuul can report on projects on GitHub now
* https://github.com/ansible/ansible/pull/20974
* Pending TC resolution and docs changes describing process

Zuul v3 Job Docs
----------------
.. transition:: pan

* Zuul-sphinx plugin
* https://docs.openstack.org/infra/zuul-jobs/
* https://docs.openstack.org/infra/openstack-zuul-jobs/
* https://docs.openstack.org/infra/zuul/user/config.html

Zuul v3 Job Migration
---------------------
.. transition:: pan

* Auto-migrated jobs start with `legacy-`
* Migrate those out of openstack-zuul-jobs into your own repos
* Look into whether existing Ansible roles are useful, or if the jobs or roles
  that you develop may be generally useful
* https://docs.openstack.org/infra/manual/zuulv3.html

Zuul v3 Devstack
----------------
.. transition:: pan

* Transition auto-migrated integration jobs to Zuul v3 native
* Devstack provides *one base job* for single and multi-node setups
* Define services, plugins, configurations in YAML
* Use job inheritance, keep your jobs readable and maintainable
* Unleash the full power of Ansible for more complex scenarios

Zuul v3 Devstack
----------------
.. transition:: tilt

.. code-block::

  - job:
      name: sahara-tests-scenario
      description: |
        Run scenario tests for Sahara.
      parent: devstack-multinode
      nodeset: openstack-two-nodes
      roles:
        - zuul: openstack/sahara-image-elements
      required-projects:
        - openstack/sahara-tests
        - openstack/sahara
        - openstack/heat
        - openstack/ceilometer
        - openstack/sahara-image-elements
        - openstack-infra/shade
      run: playbooks/sahara-tests-scenario.yaml

Zuul v3 Devstack
----------------
.. transition:: tilt

.. code-block::

      vars:
        devstack_local_conf:
          post-config:
            "$SAHARA_CONF":
              DEFAULT:
                min_transient_cluster_active_time: 90
        devstack_plugins:
          sahara: 'git://git.openstack.org/openstack/sahara'
          heat: 'git://git.openstack.org/openstack/heat'
          ceilometer: 'git://git.openstack.org/openstack/ceilometer'
          shade: 'git://git.openstack.org/openstack-infra/shade'
        devstack_services:
          s-proxy: true
          tls-proxy: false
	  (...)
        sahara_image_name: 'xenial-server'
        (...)
      group-vars:
        subnode:
          devstack_services:
            tls-proxy: false

Zuul v3 Tempest
---------------
.. transition:: pan

* Tempest provides *one base job* for single and multi-node setups
* Install plugins, filter tests from the job definition
* Integrated gate jobs migrated by the QA team
* Thorough migration documentation coming soon: https://review.openstack.org/#/c/545992/
* Grenade base job work in progress

Zuul v3 Tempest
---------------
.. transition:: tilt

.. code-block::

  - job:
      name: kuryr-kubernetes-tempest-base
      parent: devstack-tempest
      description: Base kuryr-kubernetes-job
      required-projects:
        - openstack/devstack-plugin-container
        - openstack/kuryr
        - openstack/kuryr-kubernetes
        - openstack/kuryr-tempest-plugin
        - openstack/neutron-lbaas
        - openstack/tempest

Zuul v3 Tempest
---------------
.. transition:: tilt

.. code-block::

      vars:
        tempest_test_regex: '^(kuryr_tempest_plugin.tests.)'
        tox_envlist: 'all'
        devstack_localrc:
          KURYR_K8S_API_PORT: 8080
          TEMPEST_PLUGINS: '/opt/stack/kuryr-tempest-plugin'
        devstack_services:
          base: false
          kubernetes-api: true
          kubelet: true
          kuryr-kubernetes: true
          key: true
          mysql: true
          neutron: true
          (...)
          tempest: true
        devstack_plugins:
          kuryr-kubernetes: https://git.openstack.org/openstack/kuryr
          devstack-plugin-container: https://git.openstack.org/openstack/devstack-plugin-container
          neutron-lbaas: https://git.openstack.org/openstack/neutron-lbaas

At Red Hat
----------
.. transition:: pan

* SoftwareFactory!!!

Contact Info
------------
.. transition:: pan

* IRC: #openstack-kuryr on Freenode
* E-mail: openstack-dev@lists.openstack.org
* In person: here!

* Documentation: https://docs.openstack.org/infra/system-config/
* Documentation: https://docs.openstack.org/devstack/latest/

Questions
---------
.. transition:: tilt
   :duration: 2
.. hidetitle::
.. figlet:: Questions?
