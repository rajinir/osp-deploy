
Dell EMC VNX Cinder driver containerization with OSP12
======================================================

Overview
--------
Dell EMC VNX driver integrates with the RedHat OpenStack Platform since OSP 6. In OSP12, all Cinder services are able to run within containers.
To fit this change, Dell EMC provides a customized container image on top of the OSP Cinder images thus easing the overall deployment and upgrading effors.

The container contains following RPM packages:

* EMC Naviseccli
* python-storops
* python-persist-queue
* python-cachez
* python-retryz


.. attention::

  Cinder containerization is still in `technical preview only <https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/12/html/release_notes/containers>`_ for OSP12.


Deployment prerequisites
------------------------

* RedHat OpenStack Platform 12.
* Dell EMC VNX Cinder container.
* VNX with Block version 5.32 or above.

Deployment steps
----------------

Prepare OSP12 Cinder containers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, the cinder containerization is disabled, the Cinder related container images are not pull from `registry.access.redhat.com`.
Some extra steps are needed to do that as below.

Enable the containerized Cinder services
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- Edit the file `openstack-tripleo-heat-templates/environments/docker.yaml` (make a copy if necessary), make following changes:

.. code-block:: diff

    git diff openstack-tripleo-heat-templates/environments/
    diff --git a/openstack-tripleo-heat-templates/environments/docker.yaml b/openstack-tripleo-heat-templates/environments/docker.yaml
    index 223a179..faef160 100644
    --- a/openstack-tripleo-heat-templates/environments/docker.yaml
    +++ b/openstack-tripleo-heat-templates/environments/docker.yaml
    @@ -51,14 +51,18 @@ resource_registry:
    OS::TripleO::Services::Horizon: ../docker/services/horizon.yaml
    # Because Cinder services currently run on bare metal (see FIXME), Iscsid
    # and Multipathd must run there as well.
    -  # OS::TripleO::Services::Iscsid: ../docker/services/iscsid.yaml
    -  # OS::TripleO::Services::Multipathd: ../docker/services/multipathd.yaml
    +  OS::TripleO::Services::Iscsid: ../docker/services/iscsid.yaml
    +  OS::TripleO::Services::Multipathd: ../docker/services/multipathd.yaml
    OS::TripleO::Services::ContainersLogrotateCrond: ../docker/services/logrotate-crond.yaml
    # FIXME: Had to remove these to unblock containers CI. They should be put back when fixed.
    -  # OS::TripleO::Services::CinderApi: ../docker/services/cinder-api.yaml
    -  # OS::TripleO::Services::CinderScheduler: ../docker/services/cinder-scheduler.yaml
    -  # OS::TripleO::Services::CinderBackup: ../docker/services/cinder-backup.yaml
    -  # OS::TripleO::Services::CinderVolume: ../docker/services/cinder-volume.yaml
    +  OS::TripleO::Services::CinderApi: ../docker/services/cinder-api.yaml
    +  OS::TripleO::Services::CinderScheduler: ../docker/services/cinder-scheduler.yaml
    +  OS::TripleO::Services::CinderBackup: ../docker/services/cinder-backup.yaml
    +  OS::TripleO::Services::CinderVolume: ../docker/services/cinder-volume.yaml
    +  # FIXME: Optionally, Added manila docker services
    +  OS::TripleO::Services::ManilaApi: ../docker/services/manila-api.yaml
    +  OS::TripleO::Services::ManilaScheduler: ../docker/services/manila-scheduler.yaml
    +  OS::TripleO::Services::ManilaShare: ../docker/services/manila-share.yaml
    #
    OS::TripleO::Services::SwiftDispersion: OS::Heat::None

- Include above environment file when `openstack overcloud container image prepare`:

.. code-block:: bash

    # Discover the tag for the latest images:
    (undercloud) $ sudo openstack overcloud container image tag discover \
    --image registry.access.redhat.com/rhosp12/openstack-base:latest \
    --tag-from-label version-release
    12.0-20180319.1
    # Copy the outputed tag, set it for below commmand --tag=12.0-20180319.1
    $ openstack overcloud container image prepare \
    --namespace=registry.access.redhat.com/rhosp12 \
    --prefix=openstack- --environment-directory /home/stack/templates \
    -e openstack-tripleo-heat-templates/environments/docker.yaml \
    --tag=12.0-20180319.1 \
    --output-images-file /home/stack/local_registry_images.yaml

When finished, make sure the Cinder container images appear in the `/home/stack/local_registry_images.yaml`:

.. code-block:: bash

    $ cat ~/remote_registry_images.yaml  | grep cinder
    - imagename: registry.access.redhat.com/rhosp12/openstack-cinder-api:12.0-20180319.1
    - imagename: registry.access.redhat.com/rhosp12/openstack-cinder-backup:12.0-20180319.1
    - imagename: registry.access.redhat.com/rhosp12/openstack-cinder-scheduler:12.0-20180319.1
    - imagename: registry.access.redhat.com/rhosp12/openstack-cinder-volume:12.0-20180319.1


- Pull the images binary from the `registry.access.redhat.com` to local registry.

.. code-block:: bash

    (undercloud) $ sudo openstack overcloud container image upload \
    --config-file  /home/stack/local_registry_images.yaml \
    --verbose

Pulling the required images might take some time depending on the speed of your network and your undercloud disk.

- Create a template for using the images in the local registry on the undercloud. For example:

.. code-block:: bash

    openstack overcloud container image prepare \
    --namespace=192.168.139.1:8787/rhosp12 \
    --prefix=openstack- \
    --tag=<TAG> \
    --output-env-file=/home/stack/templates/overcloud_images.yaml

This creates an `overcloud_images.yaml` environment file, which contains image locations on the undercloud. Include this file with your deployment.

Prepare Dell EMC container
~~~~~~~~~~~~~~~~~~~~~~~~~~
Before starting the deployment with VNX driver, User needs to build and push the Dell EMC container image into the local docker registry.
There are 2 options to do it: 1) `From the Dockerfile`_, 2) `From the binary container image`_.

.. attention::

    in below examples, the 192.168.139.1:8787 acts as the local registry.

From the Dockerfile
^^^^^^^^^^^^^^^^^^^
The recommended way is to build via a standalone `Dockerfile` if the overcloud is able to access the Internet.

- First, download the `Dockerfile <./Dockerfile>`_ to the director, and build a local image:

.. code-block:: bash

  $ docker build -t 192.168.139.1:8787/rhosp12/openstack-cinder-volume-dellemc .

Above command will feched and install all the VNX dependencies from Dell EMC repos.

- Then, push the image to the local docker registry.

.. code-block:: bash

  $ docker push 192.168.139.1:8787/rhosp12/openstack-cinder-volume-dellemc


From the binary container image
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- First, download the binary `vnx_container.tar <.vnx_container.tar>`_ to the director,

.. code-block:: bash

  $ docker load -i ./vnx_container.tar
  Loaded image ID: sha256:<IMAGE ID>

This returns an **<IMAGE ID>** for later use.

- Then, tag and push it to the local docker registry.

.. code-block:: bash

  $ docker tag <IMAGE ID> 192.168.139.1:8787/rhosp12/openstack-cinder-volume-dellemc
  $ docker push 192.168.139.1:8787/rhosp12/openstack-cinder-volume-dellemc


Prepare custom environment yaml
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


- Define the customer docker registry.

*/home/stack/templates/custom-vnx-container.yaml*

.. code-block:: yaml

  parameter_defaults:

    DockerCinderVolumeImage: 192.168.139.1:8787/rhosp12/openstack-cinder-volume-dellemc
    DockerInsecureRegistryAddress:
    - 192.168.139.1:8787

Above adds the director local registry IP `192.168.139.1:8787` to the `undercloud`.

- Define the VNX driver backend options.

The following sample environment file defines two VNX back ends, namely *vnx1* and *vnx2*:

*/home/stack/templates/custom-vnx-cinder.yaml*

.. code-block:: yaml

    parameter_defaults:
      CinderEnableIscsiBackend: false
      CinderEnableRbdBackend: false
      CinderEnableNfsBackend: false
      NovaEnableRbdBackend: false
      GlanceBackend: file
      ControllerExtraConfig:
        cinder::config::cinder_config:
            vnx1/volume_driver:
                value: cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
            vnx1/san_ip: #
                value: 192.168.1.50
            vnx1/san_login:
                value: admin
            vnx1/san_password:
                value: password
            vnx1/naviseccli_path:
                value: /opt/Navisphere/bin/naviseccli
            vnx1/initiator_auto_registration:
                value: True
            vnx1/storage_protocol:
                value: iscsi
            # second VNX backend
            vnx2/volume_driver:
                value: cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver
            vnx2/san_ip:
                value: 192.168.1.50
            vnx2/san_login:
                value: admin
            vnx2/san_password:
                value: password
            vnx2/naviseccli_path:
                value: /opt/Navisphere/bin/naviseccli
            vnx2/initiator_auto_registration:
                value: True
            vnx2/storage_protocol:
                value: iscsi
        cinder_user_enabled_backends: ['vnx1','vnx2']

For a full detailed instruction of options, please refer to `VNX back end configuration <https://docs.openstack.org/cinder/pike/configuration/block-storage/drivers/emc-vnx-driver.html#back-end-configuration>`_

- Deploy the configured changes.

.. code-block:: bash

  (undercloud) $ openstack overcloud deploy --templates \
  -e /home/stack/templates/overcloud_images.yaml \
  -e /home/stack/templates/custom-vnx-container.yaml \
  -e /home/stack/templates/custom-vnx-cinder.yaml \
  -e <other templates>

The sequence of `-e` matters, Make sure the `/home/stack/templates/custom-vnx-container.yaml` appears after the `/home/stack/templates/overcloud_images.yaml`, so that
custom VNX container can be used instead of the default one.


- Verify the configured changes.

After the deployment finishes succesfully, in the Cinder container, the `/etc/cinder/cinder.conf` should reflect the changes made above.

.. code-block:: ini

  ...
  enabled_backends=vnx1,vnx2
  ...
  [vnx1]
  initiator_auto_registration=True
  naviseccli_path=/opt/Navisphere/bin/naviseccli
  san_ip=192.168.1.50
  san_login=admin
  san_password=password
  storage_protocol=iscsi
  volume_driver=cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver

  [vnx2]
  initiator_auto_registration=True
  naviseccli_path=/opt/Navisphere/bin/naviseccli
  san_ip=192.168.1.50
  san_login=admin
  san_password=password
  storage_protocol=iscsi
  volume_driver=cinder.volume.drivers.dell_emc.vnx.driver.VNXDriver

On the controller node, check the output of the Cinder container.

.. code-block:: bash

  $ tail -f /var/log/containers/cinder/cinder-volume.log
  2018-04-10 02:56:03.386 38 INFO storops.vnx.navi_command [req-ad774477-17d4-4579-8c89-bbcf5755af80 - - - - -] call command: /opt/Navisphere/bin/naviseccli -h 192.168.1.50 -user sysadmin -password *** -scope global -np connection -getport -all

Security file support
~~~~~~~~~~~~~~~~~~~~~

VNX supports authentication via security file. This helps to get rid of the plain text credentials in the `/etc/cinder/cinder.conf`.
Since the security file encrypted the host specific data and related username/password together, user needs to generated it within container
after a successful deployment.

.. attention::

  Below steps need to be performed on all controller nodes.

- First get the Dell EMC Cinder <CONTAINER ID>.

.. code-block:: bash

  $ sudo docker ps | grep cinder-volume

- Run the naviseccli command to generate the security file:

.. code-block:: bash

  $ sudo docker exec -it <CONTAINER ID> /bin/bash

  # run the command
  ()[cinder@osp12ctl0 /]$ sudo -u cinder /opt/Navisphere/bin/naviseccli -AddUserSecurity -user admin -scope 0 -secfilepath /var/lib/cinder/[back end name]

It's suggested to create a security file folder for each back end.

- Edit the `/var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf` on the controller host, add `storage_vnx_security_file_dir` option under the back end section.

.. code-block:: ini

  [vnx1]
  ...
  storage_vnx_security_file_dir = /var/lib/cinder/[back end name]
  ...

.. warning::
  Do NOT change `/etc/cinder/cinder.conf` in the container directly, it will be overwriten by a container restart.

- Restart the Cinder container to enable the changes.

.. code-block:: bash

  $ sudo docker restart <CONTAINER ID>

