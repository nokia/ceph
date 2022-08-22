======================
  Ceph Release Process
======================

Prequisites
===========

Signing Machine
---------------
The signing machine is a virtual machine in the `Sepia lab<https://wiki.sepia.ceph.com/doku.php?id=start>`_.  SSH access is limited to the usual Infrastructure Admins along with a few other component leads (e.g., nfs-ganesha, ceph-iscsi).

The ``ubuntu`` user on the machine has a few `build scripts`<https://github.com/ceph/ceph-build/tree/main/scripts>`_ to ease the pulling, pushing, and signing of packages.

Finally, the GPG signing key permanently lives on a `Nitrokey Pro`<https://shop.nitrokey.com/shop/product/nkpr2-nitrokey-pro-2-3>`_ and is passed through to the VM via RHV.  This helps ensure the key can not be exported or leave the datacenter in any way.

New Major Releases
------------------
For new major (alphabetical) releases, RPM repos need a `ceph-release` RPM created.  The chacra repos are configured to include this RPM but it must be built separately. You must make sure that chacra is properly configured to include this RPM though for the release you are doing. See `this PR<https://github.com/ceph/chacra/pull/219>`_ for an example.

If chacra is not configured correctly please make that change and deploy it before starting the build of ceph or ceph-release.

https://jenkins.ceph.com/view/all/job/ceph-release-rpm/

Summarized build process
========================

1. QE finishes testing and finds a stopping point.  That commit is pushed to the ``$release-release`` branch in ceph.git (e.g., quincy-release).  This allows work to continue in the working ``$release`` branch without having to freeze it during the release process.
2. The Ceph Council approves and notifies the Build Lead.
3. Build Lead starts [Jenkins multijob](https://jenkins.ceph.com/view/all/job/ceph) which triggers all builds.
4. Packages are pushed to chacra.ceph.com.
5. Packages are pulled from chacra.ceph.com to the Signer VM.
6. Packages are signed.
7. Packages are pushed to download.ceph.com.
8. Release containers are built and pushed to quay.io.

1. Starting the build
=====================

We'll use a 17.2.4 release of Quincy as an example.

Browse to https://jenkins.ceph.com/view/all/job/ceph/build?delay=0sec.

::

    BRANCH=quincy
    TAG=checked
    VERSION=17.2.4
    RELEASE_TYPE=STABLE
    ARCHS=x86_64 arm64

Use https://docs.ceph.com/en/latest/start/os-recommendations/?highlight=debian#platforms to determine the ``DISTROS`` parameter.  In Quincy's case, ``DISTROS=centos8 centos9 focal bullseye``.

Click ``Build``!

2. Release Notes
================

Packages take hours to build so now would be a good time to create the Release Notes.

See https://tracker.ceph.com/projects/ceph-releases/wiki/HOWTO_write_the_release_notes

3. Signing and Publishing the Build
===================================

Obtain the sha1 of the version commit from the `build job<https://jenkins.ceph.com/view/all/job/ceph>`_.

::

    ssh ubuntu@signer.front.sepia.ceph.com
    sync-pull ceph octopus <sha1>

Sign the DEBs::

    merfi gpg /opt/repos/ceph/quincy-17.2.4/debian

Sign the RPMs::

    sign-rpms quincy

Publish the packages to download.ceph.com::

    sync-push quincy

4. Build Containers
===================

Start the following two jobs

https://2.jenkins.ceph.com/job/ceph-container-build-ceph-base-push-imgs/

https://2.jenkins.ceph.com/job/ceph-container-build-ceph-base-push-imgs-arm64/

5. Announce the Release
=======================

Version Commit PR
-----------------

The `ceph-tag<https://jenkins.ceph.com/job/ceph-tag>`_ Jenkins job will create a Pull Request in ceph.git targeting the release branch.

If this was a regular (not hotfix/security) release, the only commit in that Pull Request should be the version commit.  e.g., `PR#47520<https://github.com/ceph/ceph/pull/47520>`_.  Request a review and merge.

Announcing
----------

Publish the Release Notes on ceph.io first since the e-mail announcement references the ceph.io blog post.
