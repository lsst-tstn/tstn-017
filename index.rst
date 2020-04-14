

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
===================

These are the collection of requirements that drives the discussion of this
tech-note.

TLS-REQ-0065
------------

This requirement is found in :cite:`LSE-60`, section 2.8.1.3 Telescope and
Site Telemetry Requirements. It states that:

    The Telescope and Site shall publish telemetry using the Observatory specified protocol
    (Document-2233) containing time stamped structures of all command-response pairs and all technical
    data streams including hardware health, and status information.
    The telemetry shall include all required information (metadata) needed for the scientific analysis of the
    survey data as well as, at a minimum, the following:
    Changes in the internal state of the system,
    Health and status of operating systems, and
    Temperature, rate, pressure, loads, status, and conditions at all sensed system components.

It is a rater generic and broad requirement that specifies that components must
publish operational status information.

OCS-REQ-0045
------------

This requirement is specified in :cite:`LSE-62`, section 3.4.4 Subsystem Latest
Configuration. It states that:

    Specification: The Configuration Database shall manage the latest configuration for each subsystem, for
    the different observing modes.

    Discussion: The Configuration Database maintains also the latest configuration utilized during operations
    that can be utilized for rapid restoration of service in case of failure.


OCS-REQ-0069
------------

This requirement is specified in :cite:`LSE-62`, section 3.4.4.1 Subsystem
Parameters. It state that:

    Specification: The Configuration Database shall manage the subsystem parameters for the different
    observing modes.


OCS-REQ-0070
------------

This requirement is specified in :cite:`LSE-62`, section 3.4.4.2 Subsystem
History. It state that:

    Specification: The Configuration Database shall manage subsystem history for the different observing
    modes.

.. .. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa