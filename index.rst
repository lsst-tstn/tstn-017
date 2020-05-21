

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. note::

   This document is intended for both developers and users of the Vera Rubin
   observatory control system. It consolidates information about how to handle
   CSC configuration and ancillary data. The focus is on providing a basis for
   development of a unified solution on handling data that is required for
   proper operation of CSCs in view of the different requirements and
   use-cases. After reading this document develops should be informed on how to
   proceed in developing CSCs that confirm with the system architecture design.
   Users, on the other hand, should know what to expect when interacting with
   system components, how to select a configuration for a component and, most
   importantly be aware of the process to derive new configuration parameters.


.. _section-introduction:

Introduction
============

The Vera Rubin Observatory control system is highly distributed, containing
multiple components that must operate in consonant. In order to achieve reliable
operation it is fundamental to guarantee that all components are properly
configured.

Moreover we must be able to, at any point, identify what set of configurations
a component is using, what sets are available and to be able to transition
between different sets. During commissioning and engineering runs, users will
also be interested in building, validating and verifying new sets of
configurations.

The process of building new sets of configurations will vary considerably for
each component. In some cases, the configuration may be a simple set of
host names and ports that the component connects to. In other cases the
configuration may require the calculation of complex lookup tables (e.g. M1M3
and M2) or model coefficients (e.g. pointing component), probably requiring
on-sky time for acquiring the data, and complex software to analyse and derive
the configuration products.

Verification of a new configuration, in this context, mainly involves the
process of guaranteeing that the configuration has the correct schema; the
input values have the correct types and respect any specified range. Validating
that a configuration is good for operation is a much more involved procedure
and may require on-sky time.

In this tech-note we specify how components deal with these different aspects
of configuration and the basics procedures required to build and update them.

.. _section-system-requirements:

System Requirements
===================

These are the collection of requirement documents and the requirements that
drives the discussion of this tech-note.

.. _section-lse-60:

LSE-60
------

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

his is a broad requirement specifying that components must publish operational
status information.

.. _section-lse-62:

LSE-62
------

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

See furthermore details about the adopted definition of "configuration
database" in the context of the control software architecture and more details
about the proposed implementation.

.. _section-lse-150:

LSE-150
-------

Section 2.4 of the LSST Control Software Architecture :cite:`LSE-150` describes
how to perform configuration management. The document provides two valid
alternatives for managing configuration in the LSST system; through a
configuration database or version control system.

For a configuration database, any solution is acceptable as long as the
technology allows versioning of the database.

For version control systems the adopted solution is
`git <https://git-scm.com>`__. The document also specifies that configurations
must be stored in a separate repository from that of the component source code,
to allow the configuration to evolve independently of the main code base. The
configuration for different components can be stored individually or in groups
of components to facilitate maintainance.

.. _section-interface-definition:

Interface definition
====================

.. note::

	This section describe the current implementation of the system.

Assuming the architecture specified in LSE-150 :cite:`LSE-150` for the configuration system (database or version controlled), we must now make sure enough information is captured to satisfy the traceability requirements specified in LSE-62 :cite:`LSE-62`.

LSE-209 :cite:`LSE-209` specifies the global interface a component must implement to be a valid resource of the system, including a set of events to satisfy the aforementioned configuration requirements, e.g.; ``settingVersions`` and ``settingsApplied`` events.
At the time of this writing, the information contained in LSE-209 :cite:`LSE-209` is insufficent to guide the development of the interfaces.

The diagram in :numref:`fig-csc-start` illustrates the agreed upon interface for handling CSC configuration and satifying the system requirements.

.. figure:: /_static/ConfigCSCStart.png
   :name: fig-csc-start
   :target: ../_images/ConfigCSCStart.png
   :alt: Configurable CSC start process

   Configurable CSC start process.

.. _section-setting-versions:

settingVersions
---------------

The ``settingVersions`` event is a Generic event that is to be implemented by every CSC.
It contains three parameters: ``recommendedSettingsLabels``, ``recommendedSettingsVersion`` and ``settingsUrl``.
This event must be published by all CSCs upon entering the ``STANDBY`` state and broadcast information about "recommended" settings (e.g. configurations).
The combination of ``recommendedSettingsLabels`` and ``recommendedSettingsVersion`` allow users to uniquely identify a set of configurations.
A name/version setting must always result in the same configuration parameters.
Any changes in the configuration repository or database should produce a new version.

The ``settingsUrl`` attribute is a URL indicating how the CSC connects to its settings.
It will start with "file:" if it is a clone of a git repo, or the standard URL, if a database.

The ``recommendedSettingsLabels`` will contain a comma separated list of labels, each label maps to a configuration.
The same label can point to different name/version pair over time.
This information should be available in the CSC configuration repository or database and must match the value in ``recommendedSettingsLabels`` published to SAL.
Labels must be human readable strings that clearly state the purpose of that configuration (e.g. current, nighttime, daytime).
Labels should avoid having version numbers or dates in them.
They are group classifiers and have some relative permanence.
Transient labels with Jira ticket numbers may be used for developing new configurations.
They should be moved to standard type labels at the earliest opportunity.
**The order of the labels is important**, as the first label in the list will be the one selected by the high-level control system for any configurable CSC.

The ``recommendedSettingsVersion`` will be filled with the version information about the local configuration repository or database.
For configurations stored in git repositories the following *branch description*\ [#git_version]_ is used:

.. prompt:: bash

    git describe --all --long --always --dirty --broken

.. [#git_version] The option ``--broken`` was introduced in git 2.13.7.

The repository branch (or tag) name forms the first part of the branch description.
It may take any form necessary to convey the appropriate information.
They are individual identifiers and can change rapidly.

The configuration repository or database may contain any number of different configurations with different labels.
Configurable CSCs must specify a list of recommended labels.
How they implement this is up to the CSC.
It should be noted that *not all configurations need to be associated with a label*.
For instance, old configuration files that are still valid can be kept with the repository without a label.
This will allow knowledgeable users to use them if needed.

At minimum, all configurable CSCs should pass at least one label in the ``recommendedSettingsLabels`` attribute, which can be explicitly referenced in the ``settingsToApply`` attribute of the ``start`` command.
The CSC should understand how to use this label to retrieve the correct configuration.
See caveats to this process below.

.. _section-settings-applied:

Settings Applied
----------------

The ``settingsApplied`` event is a Generic event that is to be implemented by every CSC.
It currently contains two parameters: ``settingsVersion`` and ``otherSettingsEvent``.
This event should be published between the ``start`` command response starts to execute and before it finishes.

When the configuration is managed using git, ``settingsVersion`` will contain the SHA of the repository.
For a database configuration ``settingsVersion`` will have TBD.

The ``otherSettingsEvents`` is a comma-separated list of other specific CSC configuration events.
This may be blank if no other specific CSC events are necessary.
If ``otherSettingsEvents`` is not blank, then those event(s) must be published by the CSC alongside the ``settingsApplied`` event.
The CSC is allowed to publish as many events as necessary to convey the information.

.. _section-other-settings-applied:

Other Settings Applied events
-----------------------------

Since it is not possible to provide a generic way for CSCs to output detailed information about the configuration parameters they are loading, it is recommended to create additional events which are particular to each CSC to carry that information.

Although it is not required, for clarity, we suggest that these events be preceded by ``settingsApplied`` followed by some description of the content, e.g., ``settingsAppliedLUT`` or ``settingsAppliedController``.

.. _section-available-solutions-and-frameworks:

Available solutions and frameworks
==================================

.. _section-salobj:

Salobj Derived CSCs
-------------------

`Salobj <https://ts-salobj.lsst.io>`__ is the framework provided by Telescope & Site to develop CSCs in Python.
Extensive development documentation is available, especially on how to create `configurable CSCs <https://ts-salobj.lsst.io/salobj_cscs.html#writing-a-csc>`__.

Components that are written using the framework will automatically inherit the standard behavior implemented in the library.
The main points regarding Salobj CSCs are:

  #. Definition of the configuration repo.
     In general CSC configuration should be grouped according to the overall system architecture.
     For instance, `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__ hosts configurations for all the `ATCS` configurable components.
  #. The configuration package is specified in the CSC code by overriding the method `get_config_pkg <https://github.com/lsst-ts/ts_salobj/blob/301034ad249af0b0af01a884c6be205bf3a8f70b/python/lsst/ts/salobj/configurable_csc.py#L426-L429>`__.
  #. The CSC defines a schema for its configuration, which lives with the CSC repository.

The configuration for a CSC is stored in the configuration repository in a directory with the same name as the CSC, e.g. `ATAOS <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS>`__ in `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__ stores the configuration files for the `ATAOS <https://github.com/lsst-ts/ts_ataos>`__ CSC.

The first level inside a CSC configuration package will have the schema version, e.g., `ATAOS/v1 <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS/v1>`__ and `ATAOS/v2 <https://github.com/lsst-ts/ts_config_attcs/tree/develop/ATAOS/v2>`__.

Inside a schema version the user can find the available configurations and a `labels <https://github.com/lsst-ts/ts_config_attcs/blob/develop/ATAOS/v2/_labels.yaml>`__ file.
The labels will provide the mapping between the ``recommendedSettingsLabels`` and the configuration.

Note that some configuration files are not linked to any label.
They can be either removed from the most recent version of the configuration or kept there for historical or testing purposes.
Since the repository setup is published by the CSC in the ``settingVersions`` event, the user can aways go back to a set of configurations.

.. _section-camera:

Camera CSCs
-----------

These CSCs will also specify a set of labels to ``recommendedSettingsLabels``.
A given label will point to ``N`` available versions that will be published via ``recommendedSettingsVersion``.
As an example, if a label called ``normal`` is present, that label may be present as the following versions: ``normal-1.1``, ``normal-1.2``, ``normal-2.0``, ``normal-3.0``.

.. _section-handcrafted:

Other Handcrafted CSCs
----------------------

Unfortunately, not all CSCs provided by Telescope and Site are developed with a framework like Salobj that handles most of the system architecture details.
Some CSCs where developed by external vendors which did not have a framework to work with at the time the contract started.
In other cases the CSC was developed in-house using a different programming language due to performance requirements.

In these "handcrafted CSCs" the developer is in charge of constructing their own solution to the problem.
Here we gather some information about those CSCs.

.. _section-m1m3:

M1M3
^^^^

This CSC was developed in-house using C++ before a good understanding and agreement of how to handle configuration was achieved.
The CSC stores a series of configuration files which includes LUTs and other general settings.

While the code is currently not following the procedure defined in this document, it is being updated to make it compatible.

.. _section-pointing-component:

Pointing Component
^^^^^^^^^^^^^^^^^^

The pointing component has a configuration file that resides with the code base which, in itself, also defines a couple different files (e.g. pointing model).
Nevertheless, the CSC is not developed to be a configurable CSC, meaning it does not accept a ``settingsToApply`` value to switch between different configurations and does not output the required events.

The CSC is being developed by Observatory Sciences using C++.

.. _section-m2:

MTM2
^^^^

M2 cell system will read “some” configuration files (csv files basically) from disk, get the LUT values from M2 control system by TCP/IP, and hard-code many configuration data in code.

M2 control system (e.g. CSC) will read “some” configuration files (csv, tsv, txt) from disk and has several of hard-coded internal configuration.
There is no documentation specifying the location of all the hard-coded data and what they are.

All configurations reside with the main code base.
The CSC does not send any of the events required to tie in the configuration version and does not accept a ``settingsToApply`` value to switch between different configurations.

Telescope and Site developers are working to update the M2 controller to fix the different issues with how it handles configuration, e.g. removing the hard-coded values, and to make sure it follows the appropriate guidelines.

.. _section-atmcs-atpneumatics:

ATMCS and ATPneumatics
^^^^^^^^^^^^^^^^^^^^^^

The ATMCS and ATPneumatics are both being developed in LabView by a subcontract with CTIO.
Both CSCs contain a couple of ``.ini`` configuration files that are stored with the main code base.
Neither CSC accepts a ``settingsToApply`` value to switch between different configurations nor outputs the required events.

.. _section-non-configurable-cscs:

Non-Configurable CSCs
---------------------

Some CSCs will not be configurable at all.
Examples are sparse in our current architecture but, the from Salobj point of view, a CSC can be developed on top of a ``BaseCSC`` which makes it a non-configurable component.

A non-configurable CSC will ignore the ``settingsToApply`` attribute of the ``start`` command, as it does not contain any true meaning to it.
Likewise these CSCs will not output any of the configuration-related events.

As can be seen from previous sections, most of the :ref:`handcrafted CSCs <section-handcrafted>` written in C++ or LabView are not "Configurable CSCs", in the sense that they either ignore the ``settingsToApply`` value on the ``start`` command or does not output all the appropriate events.

.. _section-examples:

Examples
--------

The most simple (and probably most common) case is for those where the CSC has only a single recommended setting.
For example, for the ATDome CSC we have:

::

  recommendedSettingsLabels: test
  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7
  settingsUrl: file:///home/saluser/repos/ts_config_attcs/ATDome/v1

Some CSCs may also have multiple recommended settings, one of them being the preferred or default and another being secondary and so on.
In this case, the purpose of those configurations should be spelled out.
As an example, the ATAOS has a couple of available options for look-up tables.
In this case, we may have something like:

::

  recommendedSettingsLabels: current,constant_hex,high_degree
  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7
  settingsUrl: file:///home/saluser/repos/ts_config_attcs/ATAOS/v2

Note how the ``recommendedSettingsVersion`` from both CSCs have the same value.
Both configurations reside in the same repository: ``ts_config_attcs``.

Imagine now that during a test run, someone connects to the computer running the ATAOS CSC and edits the configuration.
The ``recommendedSettingsVersion`` would reflect that change with something like:

::

  recommendedSettingsVersion: v0.3.0-0-g6fbe3c7-dirty

Even though it may be useful to edit configurations on the fly for testing, the process should be avoided as much as possible.
When this happen, it prevents us from precisely identifying what configuration was used.
Alternatively, the user could create a branch on their work machine, make the required changes, commit, push it to GitHub and pull/check out the new configuration in the CSC machine.

For a CSC that uses a configuration database, like the ATCamera, we may have something like:

::

  recommendedSettingsLabels: normal,highgain_fast,lowgain_fast,highgain_slow,lowgain_slow
  recommendedSettingsVersion: 1.1,1.2,2.0,3.0
  settingsUrl:  sqlite:///home/camuser/config/config.db

It might be the case where the configuration is hosted in a sql database which enables remote connection.
Is this case, we could have something like:

::

  settingsUrl: mysql://10.0.100.104:3306/CONFIG

.. _section-proposed-changes:

Proposal for improvements
=========================

The sections above describes the implementation of how CSC configuration is handled by the system, at the time of this writing.
During initial integration and tests we realized that the solution has some critical points and, as such, we would like to address them.
This section describes some of the issues we found and propose changes to the system to improve the user experience and system reliability.

The following should be seen as an open floor for discussions and we expect developers and users to comment and provide feedback before we can start implementation.
It should also be noted that these changes will require work from Telescope and Site and other sub-systems.
For components written in Salobj it should be straightforward to implement these changes but those :ref:`handcrafted CSCs <section-handcrafted>` will need to be updated case by case.

.. _section-default-configuration:

Base configuration (handling default configuration values)
----------------------------------------------------------

This is mainly a proposal to update how Salobj manages default configuration values.
Other :ref:`handcrafted CSCs <section-handcrafted>` are encouraged to follow these proposal as closely as possible to maintain uniformity across the system.

As described :ref:`above <section-salobj>`, CSCs written with Salobj define a configuration schema (e.g. `ts_atdome <https://github.com/lsst-ts/ts_ATDome/blob/develop/schema/ATDome.yaml>`__).
The configuration schema contains default values for the configuration which are loaded if the ``start`` command is sent with an empty ``settingsToApply`` attribute (the default value).
Nevertheless, the values in the schema are seldom valid beyond a unit testing environment, which requires users to provide some kind of *operational defaults* or *default label*.
At the same time, overriding only a small subset of the *schema defaults* is usually enough for operations.
Therefore, to get a full set of applied configurations, users must look at two distinct repositories; the configuration repository (for the modified parameters) and the CSC repository (for the schema defaults).

The proposal to improve this aspect of the system is:

#.  Remove default values from the configuration schema.
#.  On the configuration repository there must be a ``_base.yaml`` file defining all the base configuration values (we use "base" instead of "default").
#.  The labels file (e.g. ``_labels.yaml``) will continue to exist with the same format and purpose.
#.  Additional configuration files can provide new values for individual configuration parameters.
#.  If a CSC receives a ``start`` command with an empty ``configuration`` (see :ref:`<section-renaming>`) attribute, it will load the values in ``base.yaml``.
#.  If a CSC receives a ``start`` command with a ``configuration`` attribute equal to a label in in ``_labels.yaml``, it will load the values in ``base.yaml`` first and override those values defined in the mapped configuration file.
#.  The name ``default`` should not be used for labels.

.. _section-appendix-recommended:

Use of "recommended" in settingVersions event
---------------------------------------------

The use of "recommended" in both the main attributes of ``settingVersions`` event is a bit of a misnomer. The ``recommendedSettingsLabels`` actually specify available labels and ``recommendedSettingsVersion`` specify the version of the available configuration.

One suggestion would be to rename those to ``labels`` and ``version`` only, see :ref:`section-appendix-renaming` furthermore.

.. _section-appendix-mapping:

Configuration mapping
---------------------

The mapping between label, configuration file, and branch description is not sufficiently straightforward, especially when trying to analyze configurations used in the past.

For instance, to perform a full reconstruction using the current framework on a salobj CSC, the user needs to know the configuration repository (which is specified in ``settingVersions.settingsUrl``), the label used to configure the CSC (specified in ``start.settingsToApply`` command) and the version of the configuration (specified in both ``settingVersions.recommendedSettingsVersion`` and ``settingsApplied.settingsVersion``).
To get the label to configuration file mapping the user then need to go to the configuration repository, find the version that matches the one used by the CSC at that time, check the ``_label.yaml`` file so they can finally find out what configuration was used.

This information should probably be present in both ``settingVersions`` and ``settingsApplied``.
For instance ``settingVersions`` should contain an additional entry, e.g. ``mapping``, with the names of the files the labels maps to (in case of git-based configuration) or any other appropriate mapping information (e.g. for database configuration).
Then, ``settingsApplied`` should also have an entry for ``label`` and ``mapping`` that will contain information about the selected label.

.. _section-appendix-available-vs-selected:

Available vs. selected
----------------------

The ``settingsApplied`` and ``settingsVersion`` events represent what is available and what was selected, respectively.
These topics should have *nearly* identical attributes; available labels vs selected label, available mapping vs selected map, available branch description vs selected branch description and so on.

.. _section-appendix-renaming:

Topic renaming
--------------

The interface would be much clear if we rename some topics and attributes to reflect more closely their true meaning.

The following is a renaming suggestion for discussion:

    #.  Rename ``settingsVersions``to``configurationsAvailable``

        #.  Rename ``recommendedSettingsLabels`` to ``labels``.
        #.  Rename ``recommendedSettingsVersion`` to ``version`` or ``branchDescription``.
        #.  Rename ``settingsUrl`` to ``url``.
        #.  Add ``mapping`` or ``filename``.
            Filename might not be suitable for database based configuration.

    #. Rename ``settingsApplied``to ``configurationApplied``

        #.  Add ``label``.
        #.  Add ``version`` or ``branchDescription``.
        #.  Add ``mapping`` or ``filename``.
        #.  Add ``url``.
        #.  Rename ``otherSettingsEvent`` -> ``otherInfo``.

.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
