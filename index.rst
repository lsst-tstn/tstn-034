:tocdepth: 1

.. sectnum::

Abstract
========

The First-Look Analysis and Feedback Functionality (FAFF) working group, identified the need for a service that is capable of executing user-defined operations/computation following user-defined rules.
This service has been referred  to as Catcher.
In essence, the Catcher is not that different from the Watcher or other stream processing off-the-shelf technologies like Kapacitor.
This technote explores the idea of using influx2.0 new stream processing capabilities to as an alternative implementation for the Catcher.

Introduction
============

One of the finding of the First-Look Analysis and Feedback Functionality (FAFF) working group :sitcomtn:`025` :cite:`SITCOMTN-025`, was the need for a service to ...

   handle the low-level coordiation of identifying when a specific condition is met, then launching and monitoring an analysis process.

The original proposal from the FAFF working group was to develop a Commandable SAL Component (CSC), similar to the Watcher, that would provide be capable of monitoring data from the system and trigger certain actions (e.g. perform some computation and/or generate reports) based on some pre-defined conditions.

Following additional discussions, we have decided to experiment with an alternative approach, replacing the CSC with an off-the-shelf solution provided by the Engineering Facility Database (EFD).

As shown in :sqr:`034` :cite:`SQR-034`, the EFD has of an `influxDB <https://www.influxdata.com/>`__ time series database, that stores the data produced by the system.
The release of influx2.0, brings a number of important updates, amongst which two are relevant for stream processing; `flux`_ and `influxDB task`_.

.. _flux: https://docs.influxdata.com/flux/v0.x/
.. _influxDB task: https://docs.influxdata.com/influxdb/cloud/process-data/get-started/


`flux`_ is a new functional data scripting language that is designed to process data streams.
It contains extensive libraries that allows users to express complex operations, compatible with what can be done with programming languanges.
Furthermore, `influxDB task`_ provides a way to schedule `flux`_ scripts.


Each Catcher "rule" or "alarm" would be comprised of an `influxDB task`_ to execute a user defined `flux`_ script periodically.
The `flux`_ script performs a set of actions/calculations based on EFD data, additional processes/tasks can be triggered depending on the results of these calculations and results can be stored in the EFD for future reference.

Below we describe how some of the Catcher functionality would be implemented.

Alarms
------

One of the key features expected from the Catcher was the ability to issue alarms that can be displayed in LOVE.
One option to support this feature would be to create a custom LOVE producer that can be interacted with using a REST API.
The alert information would be persisted in the EFD by the `flux`_ script, which would then post a message to the producer of the new alert.
The producer would then read the alert informatino from the EFD and display it.

Process Image Data
------------------

Another important feature of the Catcher was the capability to process camera image data using the OCPS.
The OCPS is a CSC that is capable of executing DM-pipeline processes.
It is going to be key in allowing remote processing of Main Camera data during commissioning and operations.

If the Catcher is not a CSC, it won't be able to interact direcly with OCPS to process image data.
One alternative would be to interact with the OCPS remote backend direcly, instead of going through the CSC.

Generate reports
----------------

The Catcher should also be able to generate reports, with graphs and performing additional lightweight computation.
The original idea was to rely on parameterized Jupyter notebooks that would later be converted to webpages and rendered online so users could inspect the results, and use as reference for additional followup.

This could still be achieved using a similar solution or relying on a service like noteburst :sqr:`065` :cite:`SQR-065` and Times Square :sqr:`066` :cite:`SQR-066` to execute parameterized notebooks using a web request and publishing the results afterwards.


Service Architecture
====================

Catcher would be a service built on top of existing influxDB and LSP functionality, with ancillary support from LOVE, the OCPS backend and LSP infrastucture.
It will mostly rely on `flux`_ scripts and `influxDB task`_ to execute operations based on data streams from the EFD.
The results can be persisted in the EFD, as well as summary information to be shown on LOVE.


.. figure:: /_static/Catcher.png
   :name: fig-catcher
   :target: ../_images/Catcher.png
   :alt: Catcher Architecture.


Prototype
=========

TBD

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. rubric:: References

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
   :style: lsst_aa

.. References
.. ==========

.. .. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..   :style: lsst_aa
