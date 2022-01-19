

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. important::
    This document has been deprecated.
    Content has been merged into the `CSC Configuration Manual <tstn-020.lsst.io>`_

.. note::

   This document is intended for both developers and users of the Vera Rubin observatory control system.
   It consolidates information about how to handle CSC configuration and ancillary data.
   The focus is on providing a basis for development of a unified solution on handling data that is required for proper operation of CSCs in view of the different requirements and use-cases.
   After reading this document developers should be informed on how to proceed in developing CSCs that conform with the system architecture design.
   Users, on the other hand, should know what to expect when interacting with system components, how to select a configuration for a component and, most importantly be aware of the process to derive new configuration parameters.


.. _section-introduction:

Introduction
============

The Vera Rubin Observatory control system is highly distributed, containing multiple components that must operate in consonant.
In order to achieve reliable operation it is fundamental to guarantee that all components are properly configured.

Moreover we must be able to, at any point, identify what set of configurations a component is using, what sets are available and to be able to transition between different sets.
During commissioning and engineering runs, users will also be interested in building, validating and verifying new sets of configurations.

The process of building new sets of configurations will vary considerably for each component.
In some cases, the configuration may be a simple set of host names and ports that the component connects to.
In other cases, the configuration may require the calculation of complex lookup tables (e.g. M1M3 and M2) or model coefficients (e.g. pointing component), probably requiring on-sky time for acquiring the data, and complex software to analyze and derive the configuration products.

Verification of a new configuration, in this context, mainly involves the process of guaranteeing that the configuration has the correct schema, the input values have the correct types and respect any specified range.
Validating that a configuration is acceptable for operation is a much more involved procedure and may require on-sky time.

In this tech-note we specify how components deal with these different aspects of configuration and the basics procedures required to build and update them.

.. _section-interface-definition:

Interface Definition
====================

.. note::

	This section describes the current implementation of the system.

A description of the requirements used to guide this discussion is shown in :ref:`section-system-requirements`.

Based on the architecture specified in :ref:`LSE-150 <section-lse-150>` :cite:`LSE-150` for the configuration system (database or version controlled), we must make sure enough information is captured to satisfy the traceability requirements specified in :ref:`LSE-62 <section-lse-62>` :cite:`LSE-62`.

LSE-209 :cite:`LSE-209` specifies the global interface a component must implement to be a valid resource of the system, including a set of events to satisfy the aforementioned configuration requirements, e.g.; ``settingVersions`` and ``settingsApplied`` events.
At the time of this writing, the information contained in the released version of LSE-209 :cite:`LSE-209` is insufficient to guide the development of the interfaces.
The updated LSE-209 draft can be found as part of the `change control board process (LCR-2764)<https://project.lsst.org/groups/ccb/node/4665>`_ and is expected to be released soon.

The diagram in :numref:`fig-csc-start` illustrates the agreed upon interface for handling CSC configuration and satisfying the system requirements.

.. figure:: /_static/ConfigCSCStart.png
   :name: fig-csc-start
   :target: ../_images/ConfigCSCStart.png
   :alt: Configurable CSC start process

   Configurable CSC start process.

.. _section-setting-versions:

Setting Versions
----------------

The ``settingVersions`` event is a Generic event that is to be implemented by every CSC.
It contains three parameters: ``recommendedSettingsLabels``, ``recommendedSettingsVersion`` and ``settingsUrl``.
This event must be published by all CSCs upon entering the ``STANDBY`` state and broadcast information about "recommended" settings (e.g. configurations).
The combination of ``recommendedSettingsLabels`` and ``recommendedSettingsVersion`` allow users to uniquely identify a set of configurations.
A name/version setting must always result in the same configuration parameters.
Any changes in the configuration repository or database should produce a new version.

The ``settingsUrl`` attribute is a URL indicating how the CSC connects to its settings.
It will start with "file:" if it is a clone of a git repo, or the standard URL, if a database.

The ``recommendedSettingsLabels`` contains a comma separated list of labels, each label maps to a configuration.
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

Other Settings Applied Events
-----------------------------

Since it is not possible to provide a generic way for CSCs to output detailed information about the configuration parameters they are loading, it is recommended to create additional events which are particular to each CSC to carry that information.

Although it is not required, for clarity, we suggest that these events be preceded by ``settingsApplied`` followed by some description of the content, e.g., ``settingsAppliedLUT`` or ``settingsAppliedController``.

.. _section-available-solutions-and-frameworks:

Available Solutions and Frameworks
==================================

.. _section-salobj:

Salobj Derived CSCs
-------------------

`Salobj <https://ts-salobj.lsst.io>`__ is the framework provided by Telescope & Site to develop CSCs in Python.
Extensive development documentation is available, especially on how to create `configurable CSCs <https://ts-salobj.lsst.io/salobj_cscs.html#writing-a-csc>`__.

Components that are written using the framework will automatically inherit the standard behavior implemented in the library.
The main points regarding Salobj CSCs are:

  #. Definition of the configuration repository.

        - In general CSC configuration should be grouped according to the overall system architecture.
          For instance, `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`__ hosts configurations for all the `ATCS` configurable components.

  #. The configuration package is specified in the CSC code by overriding the method `get_config_pkg <https://github.com/lsst-ts/ts_salobj/blob/301034ad249af0b0af01a884c6be205bf3a8f70b/python/lsst/ts/salobj/configurable_csc.py#L426-L429>`__.
  #. The CSC defines a schema for its configuration, which lives in the CSC repository.

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

The ATMCS and ATPneumatics are both being developed in LabVIEW under a subcontract with CTIO.
Both CSCs contain a couple of ``.ini`` configuration files that are stored with the main code base.
Neither CSC accepts a ``settingsToApply`` value to switch between different configurations, nor outputs the configuration specific events.

.. _section-non-configurable-cscs:

Non-Configurable CSCs
---------------------

Some CSCs will not be configurable at all.
Examples are sparse in our current architecture but, the from Salobj point of view, a CSC can be developed on top of a ``BaseCSC`` which makes it a non-configurable component.

A non-configurable CSC will ignore the ``settingsToApply`` attribute of the ``start`` command, as it does not contain any true meaning to it.
Likewise these CSCs will not output any of the configuration-related events.

As can be seen from previous sections, most of the :ref:`handcrafted CSCs <section-handcrafted>` written in C++ or LabVIEW are not "Configurable CSCs", in the sense that they either ignore the ``settingsToApply`` value on the ``start`` command or does not output all the appropriate events.

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

Proposal for Improvements
=========================

The sections above describe the implementation of how CSC configuration is handled by the system, at the time of this writing.
During initial integration and tests we realized that the solution has some critical weaknesses that we need to address.
This section describes some of the issues we found and proposes changes to the system to improve the user experience and system reliability.

The following suggested implementation is still open for discussion and we encourage developers and users to comment and provide feedback before starting the implementation process.
The formal change will be submitted as an LCR to LSE-209.
It should also be noted that these changes will require work from Telescope and Site and other sub-systems.
For components written in Salobj it should be straightforward to implement these changes but those :ref:`handcrafted CSCs <section-handcrafted>` will need to be updated case by case.

.. _section-renaming:

Topic and Attribute Renaming
----------------------------

The clarity and purpose of the interface would be improved by some renaming of generic topics and attributes to better reflect their true meaning.

.. note::
    These changes are being proposed as part of LCR-2764.
    Upon completion of this LCR, this tech-note will be deprecated and the applicable information will be moved to `a CSC Configuration Manual <tstn-020.lsst.io>`_

For instance, one of the things to point out is the use of words like "recommended" and "settings" in attributes that are related to configuration information.
Users will usually count on being able to easily enable a component with appropriate defaults first and then, what different configurations they have available to fine tune the behavior of the system.
The use of *recommended* gives the impression that not everything that is shown is what is available (which is true in some cases), and also means users must look into the configuration repository to know what else is available.
On the other hand *settings* really seems like a misnomer for *configuration*.

The following list details the changes to the current implementation.
Besides the renaming of numerous events and attributes, the most significant change is the removal of the labels associated with configuration files.

#.  Rename ``settingsVersions`` event to ``configurationsAvailable``.

    This topic presents **all** the available configurations that can be loaded by the CSC (see :ref:`the proposal <section-default-configuration>` to change the way CSC handles configuration).
    As will be discussed in the following section, only the files that override the initial and site-specific values will be displayed.

    #.  Remove all notions of labels, including the ``recommendedSettingsLabels`` attribute
    #.  Add a new ``overrides`` attribute

        - This will consist of a comma separated list of all configuration files in the configuration repo that can be loaded as overrides (discussed below)

    #.  Rename the ``recommendedSettingsVersion`` attribute to ``version``

        - This will consist of the git hash associated with the commit of configuration repo that can be loaded as overrides (discussed below).

    #.  Rename ``settingsUrl`` attribute to ``url``
    #.  Add ``schemaVersion`` which indicates the schema version in use (e.g. v3)

#.  Rename ``settingsApplied`` event to ``configurationApplied``

    #.  Add ``configurations`` attribute
        - This will consist of a comma separated list of between one and three file names (discussed below)

    #.  Add ``version`` attribute
    #.  Add ``url`` attribute
    #.  Add ``schemaVersion`` attribute
    #.  Rename the ``otherSettingsEvent`` event to ``otherInfo``.

    The event will publish the selected values once the CSC is configured.

#.  In the ``start`` command, rename keyword parameter ``settingsToApply`` to ``configurationOverride``.


.. _section-continuous-monitoring:

Monitoring of the Configuration Repository
------------------------------------------

Right now CSCs are required to publish ``configurationsAvailable``  (former ``settingsVersions``, see :ref:`renaming proposal <section-renaming>`) when they transition to ``STANDBY`` state.
Nevertheless, while in ``STANDBY`` state it is possible for someone to update the available configuration, which would make the information out of sync.
We propose that, while in ``STANDBY`` state, CSCs continuously monitor the configuration repository and update/publish new topics as needed.
This monitoring should only happen while the CSC is in ``STANDBY`` and should not interfere with any other state.
For instance, when transitioning from ``DISABLE`` to ``STANDBY``, the CSC shall not start monitoring until the transition is completed and the command acknowledged.

.. _section-default-configuration:

Initial Configuration and Handling of the Default Configuration Values
----------------------------------------------------------------------

This is mainly a proposal to update Salobj's management of default configuration values.
Other :ref:`handcrafted CSCs <section-handcrafted>` are encouraged to follow this proposal as closely as possible to maintain uniformity across the system.

As described :ref:`above <section-salobj>`, CSCs written with Salobj define a configuration schema (e.g. `ts_atdome <https://github.com/lsst-ts/ts_ATDome/blob/develop/python/lsst/ts/ATDome/config_schema.py>`__).
The configuration schema currently contains default values for the configuration parameters which are loaded if the ``start`` command is sent with an empty ``configurationOverride`` keyword (the default value).
Nevertheless, the values in the schema are seldom valid beyond a unit testing environment, which requires users to provide some kind of *operational defaults* or *default label*.
One can see how this can cause confusion when operating the system since "default" can be interpreted in two different ways, e.g.; *schema default* and *operational default*.
Furthermore, it is usually enough to override a small subset of the *schema defaults* for operations.
Therefore, to get a full set of applied configurations, users must look at two distinct repositories; the configuration repository (for the modified parameters) and the CSC repository (for the schema defaults).

The proposal to improve this aspect of the system is:

#.  Remove all default values from configuration schema definition in the CSC repository.

    - See this :download:`example schema <_static/ATSpectrograph_schema.yaml>` for the ATSpectrograph CSC.
    - Unit tests will need to utilize configuration files stored in the `tests/data/config` directory, as is done for the `ATDome CSC <https://github.com/lsst-ts/ts_ATDome/tree/develop/tests/data/config>`_.
      See `Salobj documentation <https://ts-salobj.lsst.io>`__ for more details.

#.  In the configuration repository for the given CSC (e.g `ts_config_attcs <https://github.com/lsst-ts/ts_config_attcs>`_ for the ATDome) there shall be a ``_init.yaml`` file that specifies values that are expected to be common to all sites and/or be relatively static in operations (we intentionally use "_init" instead of "_default").

    - See this :download:`example _init.yaml <_static/_init.yaml>` for the ATSpectrograph CSC.
    - This file is the first configuration file loaded by the CSC
    - Providing the ``_init.yaml`` file (or any file with a ``_`` prefix) to the ``configurationOverride`` parameter will return an error
    - Note that all CSCs having multiple algorithms [2]_, each with different required configuration parameters, must have an initial set of defaults in this file.

#.  Also in the configuration repository for the given CSC, when applicable, there will be a file corresponding to each site where the CSC is used (e.g. ``_summit.yaml, _ncsa.yaml, _base.yaml``).
    These files contain site-specific configuration parameters such as IP addresses and ports.
    However, if no site-specific parameters exist for the CSC, then the use of this file is not required.
    Items in the ``_<site>.yaml`` file will override values that may have been declared in the ``_init.yaml`` file
    SalObj determines which site-specific file should be loaded automatically by parsing the ``LSST_DDS_PARTITION_PREFIX`` environment variable

    - See this :download:`example _summit.yaml <_static/_summit.yaml>` for the ATSpectrograph CSC.
    - This file is the second configuration file to get loaded by the CSC and will override any previously declared values.
    - Providing the ``_<site>.yaml`` file (or any file with a ``_`` prefix) to the ``configurationOverride`` parameter will return an error
    - The combination of the ``_<site>.yaml`` and ``_init.yaml`` files **must fully populate all configuration parameters**.

#.  An additional configuration file provides overrides for the configuration parameters set by the previous files.

    - See this :download:`configuration parameter override example file <_static/ATSpectrograph_example_config.yaml>` for the ATSpectrograph CSC.
    - This file is the third configuration file to get loaded by the CSC and will override any previously declared values.
    - These files are loaded using the ``configurationOverride`` parameter in the ``start`` command
    - These are not expected to be required as part of regular operations and are meant to be used when a non-standard configuration is required
    - If an override configuration file is also site-specific, then a prefix should be added indicating which site it belongs with (e.g. ``summit_reduced_stage_travel.yaml``)

#.  The labels file, and all notions of labels, shall be deprecated. Only filenames shall be used.

    - No file shall exist having the name ``default.yaml``.
      There are other invalid names for files (e.g. ``init.yaml``) which are to be verified by continuous integration tests in the configuration repository.

#.  If a CSC receives a ``start`` command with an empty ``configurationOverride`` (see :ref:`renaming proposal <section-renaming>`) parameter, it shall load the values in ``_init.yaml`` then the site-specific file (e.g. ``_summit.yaml``).

#.  If a CSC receives a ``start`` command with a ``configurationOverride`` parameter equal to a valid filename, it loads the values in ``_init.yaml``, then the site-specific file (e.g. ``_summit.yaml``) if it exists, and lastly the override file.
    An invalid filename will return as a failed command with an appropriate error message saying the file was not readable and no state transition will occur.

#.  The configuration repository shall not contain configurations used for unit testing.
    Configurations needed for unit testing shall be added to the ``test`` directory in the CSC repository and use the override feature in CSCs (see `Salobj documentation <https://ts-salobj.lsst.io>`__).

#.  Override configurations that are site-specific should contain the site name as a prefix to the filename (e.g. ``summit_simple_algorithm.yaml``).

#.  All configuration files shall have a header metadata fields explaining that they are loading basic values from ``_init.yaml``, as shown in the :download:`example configuration file <_static/ATSpectrograph_example_config.yaml>` mentioned above.

#.  Unit or integration tests requiring specific information shall utilize an override file that is specific to the test.


.. [2] A schema must be constant; it cannot change as a result of setting configuration values.
       A variable schema makes it impossible to specify a full set of default values in the _init.yaml and _<site>.yaml files.
       Consider ATDomeTrajectory: it supports selecting a following algorithm, and each algorithm can have different configuration parameters.
       In order to keep the schema constant, the schema includes configuration parameters for all following algorithms, rather than changing the schema based on which following algorithm is selected.


Required Unit and Continuous Integration (CI) Testing
-----------------------------------------------------

Due to the dependence of the configuration files on the defined schema, which are located in different repositories, CI tests are required to ensure there is no breakage when making modifications in either repository.
The verification of a configuration requires that the files are syntactically correct and that all fields are populated with correctly formatted values.
This verification is what is performed in the following tests.
The validation of a configuration requires that the input values are indeed the correct values required by the user.
Validation is out of scope for CI tests.

The following CI tests are required on all configuration repos (e.g. ``ts_config_attcs``):

    #. Verify that if site-specific configuration files exist, then they exist for all sites, and the site names are valid
    #. Verify that ``_init.yaml`` + ``_<site>.yaml`` results in a complete configuration.
       This is performed for each site-specific file.
    #. Verify that ``_init.yaml`` + ``_<site>.yaml`` + ``<override>.yaml`` is valid for all combinations of site and override files.
    #. Verify that new and/or updated configurations have updated metadata
    #. Verify that "default" is never used as a filename

The following CI tests are required on all configurable CSC repos (e.g. ``ts_ATDome``):

    #. Verify that no defaults are set in the schema.
    #. Verify that all configuration files in the configuration repository (e.g. ``ts_config_attcs``) are verified against the current schema.

.. _section-system-requirements:

Appendix: System Requirements
=============================

These are the collection of requirement documents and the requirements that drives the discussion of this tech-note.

.. _section-lse-60:

LSE-60
------

Requirement TLS-REQ-0065, in section 2.8.1.3 from the Telescope & Site Subsystem Requirements :cite:`LSE-60` states that:

    The Telescope and Site shall publish telemetry using the Observatory specified protocol (Document-2233) containing time stamped structures of all command-response pairs and all technical data streams including hardware health, and status information.
    The telemetry shall include all required information (metadata) needed for the scientific analysis of the survey data as well as, at a minimum, the following:
    Changes in the internal state of the system, Health and status of operating systems, and Temperature, rate, pressure, loads, status, and conditions at all sensed system components.

This is a broad requirement specifying that components must publish operational status information.

.. _section-lse-62:

LSE-62
------

The LSST Observatory Control System Requirements Document :cite:`LSE-62` contains three requirements regarding system configuration:

Requirement OCS-REQ-0045 in section 3.4.4 (Subsystem Latest Configuration) states that:

        Specification: The Configuration Database shall manage the latest configuration for each subsystem, for the different observing modes.

        Discussion: The Configuration Database maintains also the latest configuration utilized during operations that can be utilized for rapid restoration of service in case of failure.

Requirement OCS-REQ-0069 in section 3.4.4.1 (Subsystem Parameters) state that:

    Specification: The Configuration Database shall manage the subsystem parameters for the different observing modes.

Requirement OCS-REQ-0070 in section 3.4.4.2 (Subsystem History) state that:

    Specification: The Configuration Database shall manage subsystem history for the different observing modes.

See furthermore details about the adopted definition of "configuration database" in the context of the control software architecture and more details about the proposed implementation.

.. _section-lse-150:

LSE-150
-------

Section 2.4 of the LSST Control Software Architecture :cite:`LSE-150` describes how to perform configuration management.
The document provides two valid alternatives for managing configuration in the LSST system; through a configuration database or version control system.

For a configuration database, any solution is acceptable as long as the technology allows versioning of the database.

For version control systems the adopted solution is `git <https://git-scm.com>`__.
The document also specifies that configurations must be stored in a separate repository from that of the component source code, to allow the configuration to evolve independently of the main code base.
The configuration for different components can be stored individually or in groups of components to facilitate maintainance.



.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
