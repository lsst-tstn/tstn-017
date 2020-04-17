

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. note::

   This document consolidates information about how to handle CSC configuration
   and ancillary data. The focus is on providing basis for development of a
   unified solution on handling data that is required for proper operation of
   CSCs in view of the different requirements and use-cases.


Introduction
============

The Vera Rubin Observatory control system is highly distributed, containing
several components that must operate in consonant. Ir order to achieve reliable
operation it is fundamental to guarantee all components are properly setup
and configured.

Moreover we must be able to, at any point, identify what set of configurations
a component is using, what set is available and to be able to transition
between different sets. During commissioning and engineering runs, users will
also be interested in building, validating and verifying new sets of
configurations.

The process of building new sets of configurations will very considerably for
each component. In some cases, the configuration may be a simple set of
host names and ports that the component connects to. In other cases the
configuration may require the calculation of complex lookup tables (e.g. M1M3
and M2) or model coefficients (e.g. pointing component), probably requiring
on-sky time for acquiring the data, and complex software to analyse and derive
the configuration products.

Validating a new configuration, in this context, mainly involves the process of
guaranteeing that the configuration has the correct schema; the input values
have the correct types and respect any specified range. Verifying that a
configuration is good for operation is a much more involved procedure and may
require on-sky time.

In this tech-note we specify how components deal with these different aspects
of configuration and the basics procedures required to build and update them.


System Requirements
-------------------

These are the collection of requirement documents and the requirements that
drives the discussion of this tech-note.

LSE-60
^^^^^^

Requirement TLS-REQ-0065, in section 2.8.1.3 from the Telescope & Site
Subsystem Requirements :cite:`LSE-60` states that:

    The Telescope and Site shall publish telemetry using the Observatory
    specified protocol (Document-2233) containing time stamped structures of
    all command-response pairs and all technical data streams including
    hardware health, and status information. The telemetry shall include all
    required information (metadata) needed for the scientific analysis of the
    survey data as well as, at a minimum, the following: Changes in the
    internal state of the system, Health and status of operating systems, and
    Temperature, rate, pressure, loads, status, and conditions at all sensed
    system components.

It is a rater generic and broad requirement that specifies that components must
publish operational status information.

LSE-62
^^^^^^

The LSST Observatory Control System Requirements Document :cite:`LSE-62`
contains three requirements regarding system configuration:

Requirement OCS-REQ-0045 in section 3.4.4 (Subsystem Latest Configuration)
states that:

        Specification: The Configuration Database shall manage the latest
        configuration for each subsystem, for the different observing modes.

        Discussion: The Configuration Database maintains also the latest
        configuration utilized during operations that can be utilized for rapid
        restoration of service in case of failure.


Requirement OCS-REQ-0069 in section 3.4.4.1 (Subsystem Parameters) state that:

    Specification: The Configuration Database shall manage the subsystem
    parameters for the different observing modes.


Requirement OCS-REQ-0070 in section 3.4.4.2 (Subsystem History) state that:

    Specification: The Configuration Database shall manage subsystem history
    for the different observing modes.

LSE-150
^^^^^^^

Section 2.4 of the LSST Control Software Architecture :cite:`LSE-150` describes
how to perform configuration management. The document provides two valid
alternatives for managing configuration in the LSST system; through a
configuration database or version control system.

For configuration database any solution is acceptable as long as the technology
allow versioning of the database.

For version control systems the adopted solution is
`git <https://git-scm.com>`__. The document also specify that configuration
must be stored in a separate repository, to allow the configuration to evolve
independently of the main code base.

Interface definition
--------------------

Assuming the architecture specified in LSE-150 :cite:`LSE-150` for the use of
configuration database or version controlled configuration repository we must
now make sure enough information is captured to satisfy the traceability
requirements specified in LSE-62 :cite:`LSE-62`.

LSE-209 :cite:`LSE-209` specify the global interface a component must implement
to be a valid resource of the system, including a set of events to satisfy the
aforementioned configuration requirements, e.g.; ``settingVersions``,
``settingsApplied`` and ``softwareVersions`` events. At the time of this writing,
the information contained in LSE-209 :cite:`LSE-209` is insufficent to guide
the development of the interfaces.

The diagram in :numref:`fig-csc-start` illustrates agreed uppon the interface
for handling CSC configuration and satisfy the system requirements.

.. figure:: /_static/ConfigCSCStart.png
   :name: fig-csc-start
   :target: ../_images/ConfigCSCStart.png
   :alt: Configurable CSC start process

   Configurable CSC start process.

settingVersions
^^^^^^^^^^^^^^^

The ``settingVersions`` event is a Generic event that is to be implemented by
every CSC. It contains three parameters: ``recommendedSettingsLabels``,
``recommendedSettingsVersion`` and ``settingsUrl``. This event must be published by
all CSCs upon entering the ``STANDBY`` state and broadcast information about
"recommended" settings (e.g. configurations). The combination of
``recommendedSettingsLabels`` and ``recommendedSettingsVersion`` allow users to
uniquely identify a set of configurations. A name/version setting must always
result in the same configuration parameters. Any changes in the configuration
repository or database should produce a new version.

The ``settingsUrl`` attribute is a URL indicating how the CSC connects to its
settings.  It will start with "file:" if it is a clone of a git repo, or the
standard URL, if a database.

The ``recommendedSettingsLabels`` will contain a comma separated list of labels,
each label maps to a configuration. The same label can point to different
name/version pair over time.  This information should be available in the CSC
configuration repository or database and must match the value in
``recommendedSettingsLabels`` published to SAL. Labels must be human readable
strings that clearly state the purpose of that configuration (e.g. current,
nighttime, daytime). Labels should avoid having version numbers or dates in
them. They are group classifiers and have some relative permanence. Transient
labels with Jira ticket numbers may be used for developing new configurations.
They should be moved to standard type labels at the earliest opportunity.
**The order of the labels is important**, as the first label in the list is
assumed to be the default label for any configurable CSC.

The ``recommendedSettingsVersion`` will be filled with the version information
about the configuration repository or database. For configurations stored in
git repositories we propose to use the command;

.. prompt:: bash

    git describe --all --long --always --dirty --broken

as a descriptor. Versions may take any form necessary to convey the
appropriate information. They are individual identifiers and can change
rapidly.

The configuration repository or database may contain any number of different
configurations with different labels. Configurable CSCs must specify a list of
recommended labels. How they implement this is up to the CSC. It should be
noted that *not all configuration labels need to be related with a recommended
configuration*. For instance, old configurations that are still valid can be
kept with the repository without a label. This will allow knowledgeable users
to use them if needed.

At minimum, all configurable CSCs should pass at least one label in the
``recommendedSettingsLabels`` attribute, which can be explicitly referenced in
the ``settingsToApply`` attribute of the ``start`` command. The CSC should
understand how to use this label to retrieve the correct configuration. See
caveats to this process below.

Settings Applied
^^^^^^^^^^^^^^^^

The ``settingsApplied`` event is a Generic Event that is to be implemented by
every CSC.  It currently contains two parameters: ``settingsVersion`` and
``otherSettingsEvent``.  This event should be published between after the
``start`` command starts and before it finishes.

The ``settingsVersion`` will contain the SHA or name corresponding to the
published configuration.

The ``otherSettingsEvents`` is a comma-separated list of other specific CSC
configuration events. This may be blank if no other specific CSC events are
necessary. If ``otherSettingsEvents`` is not blank, then those event(s) must
be published by the CSC alongside the ``settingsApplied`` event. The CSC is
allowed to publish as many events as necessary to convey the information.

Other Settings Applied events
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Since it is not possible to provide a generic way for CSCs to output detailed
information about the configuration parameters they are loading, it is
recommended to create additional events (particular to each CSC) to carry
that information.

Although it is not required, for clarity, we suggest that these events be
preceded by ``settingsApplied`` followed by some description of the content,
e.g., ``settingsAppliedLUT`` or ``settingsAppliedController``.

Available solutions and frameworks
==================================

Salobj Derived CSCs
-------------------

`Salobj <https://ts-salobj.lsst.io>`__ is the framework provided by Telescope
& Site to develop CSCs in python. Extensive development documentation is
available, especially on how to create
`configurable CSCs <https://ts-salobj.lsst.io/salobj_cscs.html#writing-a-csc>`__.

Components that are written using the framework will automatically inherit the
standard behaviour implemented in the library. The main points regarding
Salobj CSCs are:

  #. Definition of the configuration repo. In general CSC configuration should
     be grouped according to the overall system architecture. For instance,
     `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__ hosts
     configurations for all the `ATCS` configurable components.
  #. The configuration package is specified in the CSC code by overriding the
     method
     `get_config_pkg <https://github.com/lsst-ts/ts_salobj/blob/301034ad249af0b0af01a884c6be205bf3a8f70b/python/lsst/ts/salobj/configurable_csc.py#L426-L429>`__.
  #. The CSC defines a schema for its configuration, which leaves with the CSC
     repository.

The configuration for a CSC is stored in the configuration repository in a
directory with the same name of the CSC, e.g.
`ATAOS <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS>`__ in
`ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__ stores the
configuration files for the `ATAOS <https://github.com/lsst-ts/ts_ataos>`__
CSC.

The first level inside a CSC configuration package will have the schema version,
e.g.,
`ATAOS/v1 <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS/v1>`__
and
`ATAOS/v2 <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS/v2>`__.

Inside a schema version the user can find the available configurations and a
`labels <https://github.com/lsst-ts/ts_config_attcs/blob/develop/ATAOS/v2/_labels.yaml>`__
file. The labels will provide the mapping between the
``recommendedSettingsLabels`` and the configuration.

Note that some configuration files are not linked to
any label. They can be either removed from the most recent version of the
configuration or kept there for historical or testing purposes. Since the
repository setup is published by the CSC in the ``settingVersions`` event, the
user can aways go back to a set of configurations.

Camera CSCs
-----------

These CSCs will also specify a set of labels to ``recommendedSettingsLabels``.
A given label will point to ``N`` available versions that will be published via
``recommendedSettingsVersion``. As an example, if a label called ``normal`` is
present, that label may be present as the following versions:
``normal-1.1``, ``normal-1.2``, ``normal-2.3``, ``normal-3.0``.

Other Handcrafted CSCs
----------------------

Unfortunately, not all CSCs provided by Telescope and Site are developed with
a framework like SalObj that handles most of the system architecture details.
Some CSCs where developed by external vendors which did not had a framework to
work with at the time the contract started. In other cases the CSC was
developed in-house using a different programming language due to performance
requirements.

In these "handcrafted CSCs" the developer is in charge of constructing their
own solution to the problem. Here we gather some information about those CSCs.

M1M3
^^^^

This CSC was developed in-house using C++ before a good understanding and
agreement of how to handle configuration was achieved. The CSC stores a
series of configuration files which includes LUTs and other general settings.

As it is the code is not following the procedure defined in this document
though, it is being updated to make it compatible.

Pointing Component
^^^^^^^^^^^^^^^^^^

The pointing component has a configuration file that resides with the code
base which, in itself, also defines a couple different files (e.g. pointing
model). Nevertheless, the CSC is not developed to be a configurable CSC,
meaning it does not accept a ``settingsToApply`` value to switch between
different configurations and does not output the required events.

The CSC is being developed by Observatory Sciences using C++.

M2
^^

M2 cell system will read “some” configuration files (csv files basically) in
disk, get the LUT values from M2 control system by TCP/IP, and hard-code many
configuration data in code.

M2 control system (e.g. CSC) will read “some” configuration files (csv, tsv,
txt) in disk and hard-code many configuration data in code. There is no
document to say where are the hard-coded data and what are they.

All configurations resides with the main code base. The CSC does not
send any of the events required to tie in the configuration version and does
not accept a ``settingsToApply`` value to switch between different
configurations.


ATMCS and ATPneumatics
^^^^^^^^^^^^^^^^^^^^^^

The ATMCS and ATPneumatics are both being developed in LabView by a
subcontract with CTIO. Both CSCs contains a couple of ``.ini`` configuration
files that are stored with the main code base. Neither accept a
``settingsToApply`` value to switch between different configurations nor
output the required events.

Non-Configurable CSCs
---------------------

Some CSCs will not be configurable at all. Examples as sparse in our current
architecture but, from Salobj point of view, a CSC can be developed on top of
a ``BaseCSC`` which makes it a non-configurable component.

A non-configurable CSC will ignore the ``settingsToApply`` attribute of the
``start`` command, as it does not contain any true meaning to it. Likewise
these CSCs will not output any of the configuration-related events.


Examples
--------

The most simple (and probably most common) case is for those where the CSC has
only a single recommended setting. For example, for the ATDome CSC we have:

::

  recommendedSettingsLabels: test
  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7
  settingsUrl: file:///home/saluser/repos/ts_config_attcs/ATDome/v1

Some CSCs may also have multiple recommended settings, one of them being the
preferred or default and another being secondary and so on.  In this case, the
purpose of those configurations should be spelled out. As an example assume
the ATAOS where we have a couple of available options for look-up tables. In
this case, we may have something like:

::

  recommendedSettingsLabels: current,constant_hex,high_degree
  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7
  settingsUrl: file:///home/saluser/repos/ts_config_attcs/ATAOS/v2

Note how both ``recommendedSettingsVersion`` have the same value and they are,
in fact, from the same configuration repo; ``ts_config_attcs``.

Imagine now that during a test run, someone connects to the computer running
the ATAOS CSC and edits the configuration. The ``recommendedSettingsVersion``
would reflect that change with something like:

::

  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7-dirty

Even thought it may be useful to edit configurations on the fly for testing,
the process should be avoided as much as possible. When this happen, it
prevents us from precisely identifying what configuration was used.
Alternatively, the user could create a branch on their work machine,
make the required changes, commit, push it to github and pull/check out the new
configuration in the CSC machine.

For a CSC that does not uses a git repository for managing configuration
version, and uses a database instead (like the ``ATCamera``), we may have
something like:

::

  recommendedSettingsLabels: default,highgain_fast,lowgain_fast,highgain_slow,lowgain_slow
  recommendedSettingsVersion: v10
  settingsUrl:  sqlite:///home/camuser/config/config.db

It might be the case where the configuration is hosted in a sql database which
enables remote connection. Is this case, we could have something like:

::

  settingsUrl: mysql://10.0.100.104:3306/CONFIG

Deriving new configuration parameters
=====================================

TBD


.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa