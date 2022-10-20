:tocdepth: 1

.. sectnum::

Abstract
========

This technote contains design considerations for the Catcher, an event-driven processing component for the Rubin Observatory Control System.
We briefly review the existing functionality of the system and discuss the use case for the Catcher.
The idea originated from the First-Look Analysis and Feedback Functionality (FAFF) working group, and a more in-depth analysis of the use cases can be found in their reports.


Introduction
============

One of the finding of the First-Look Analysis and Feedback Functionality (FAFF) working group :sitcomtn:`025` :cite:`SITCOMTN-025`, was the need for a service to ...

   handle the low-level coordination of identifying when a specific condition is met, then launching and monitoring an analysis process.

The idea is basically to have a component capable of executing event-driven processing and report generation.

The Rubin Observatory Control System currently contains a ``Watcher`` CSC that is capable of generating alerts from user defined rules (e.g. event-driven alerts).
Moreover, the ``ScriptQueue`` and ``OCPS`` CSCs are both able to execute command-driven processes.
While the ``ScriptQueue`` is tailored towards control system operations (e.g. moving the telescope, taking data, etc.), the ``OCPS`` is dedicated to executing DM-pipeline tasks processes.
The missing piece of functionality is the capability of mixing a ``Watcher``-like behavior (event-processing) with ``ScriptQueue``/``OCPS``-like behavior.
Furthermore, we are also interested in generating comprehensive reports from the results of these processes and alarms for particular conditions.

The following are the key functionality we are looking for in the ``Catcher``.

Commandable SAL Component
-------------------------

The ``Catcher`` needs to interact with the Rubin Observatory Control System, both by reading telemetry from the system and by commanding other CSCs (see :ref:`introduction-process-image-data`).
We also expect to be able to interact with the ``Catcher`` in a similar way we interact with other components on the system, such that we can start, configure, stop and redeploy it the same way we do with other components.

Furthermore, being a CSC we also expect it to be written using Python/salobj, which is a broadly used framework in the project. 
This ensures we have enough knowledge in the project to maintain and support the ``Catcher`` development.

One thing we must consider in this case is whether the ``Catcher`` CSC should be a single CSC that handles all conditions, or an indexed CSC were each instance handles a single case/rule.
Using our experience with the ``Watcher`` it seems like having an indexed CSC would be preferable.
Some reasons for this are:

*  A single CSC means there is a single event for each condition.
   If the CSC is handling multiple conditions the information ends up getting "lost" in the data queue and it is not easily digestible by LOVE and other systems.
   Whereas, if each instance of the ``Catcher`` handles a single condition/rule, events are well separated for each condition.

*  In the single CSC scenario, adding/removing/updating a particular rule requires restarting the entire system.
   In the indexed component scenario, it only requires bringing in a new instance or removing/updating an existing one.

To control the configuration of each ``Catcher`` instance, we can adopt an indexed-based configuration format, used by some CSCs in the system already.
Basically we map an index to a CSC configuration.
All configurations resides in the same configuration file and the CSC decides which configuration to load based on its index.

Rules
-----

The first thing the ``Catcher`` must be able to do is have a set of user-defined "activation" rules.
This basically consists of an input data stream that will be processed according to some user-defined code and be activated according to some user-define rule.

For example, one might be interested in a rule that activates when a new image finishes reading (camera publishes endReadout) or something a bit more complicated like activate when a new image finishes reading, the telescope is pointing in the direction of the wind and wind speed is above a certain threshold.

In terms of design, we have two different options we could implement; configuration-based or programmatic-based rules.

Configuration-based rules, are based on simple rules that can be combined together to form more complex behavior.
For example, we could a rule that receives a component name and event name and is activated every time that event is published or a slightly more complicated rule that takes 2 components/topic/attributes and check if their values are within a certain threshold.

The configuration for a rule that would monitor an event would look something like this:

.. code-block:: yaml

   rule:
      monitorEvent:
         component: ATCamera
         event: endReadout

Combining rules would be something like:

.. code-block:: yaml

   rule:
      monitorEvent:
         component: ATCamera
         event: endReadout
      compareTopics:
         topic1: ATMCS.tel_mount_AzEl_Encoders.azimuthCalculatedAngle
         topic2: WeatherStation.windDirection.value
         threshold: 1.0

The advantage of this approach is that, once a sufficient number of rules are developed, including new triggers is a matter of creating a new configuration.

Alternatively, the programmatic-based rules relies on having all the trigger logic developed under a single operation.
For example, we could have a rule to monitor the end readout event from the camera and a rule that monitors the end readout event from the camera and compare the position of the telescope with the wind.
These rules are developed specifically for their use-cases, and are not generally applied.
This is, for instance, the current approach implemented by the ``Watcher``.
Any new alarm requires code changes. 

Analysis Process
----------------

A key functionality of the ``Catcher``, that differentiates it from the ``Watcher``, is the ability to execute an analysis process once a certain condition is met.
In general, we assume that these analysis will consist of lightweight computations.
Whenever possible, the computation should be offloaded to the OCPS.
It is likely that the data processed by the rule will be relevant to the downstream processing, so it should  be made available to it as well.

These processing routines should be developed in the ``Catcher`` codebase using a common framework, similar to that provided by SAL Scripts (e.g. a base class with a well defined interface).
It is also important that the results/outputs of a process follow a well defined format that can be consumed downstream.

.. _introduction-process-image-data:

Process Image Data
------------------

Another important feature of the ``Catcher`` is the capability to process camera image data using the ``OCPS``.
The ``OCPS`` is a CSC that is capable of executing DM-pipeline processes.
It is going to be key in allowing remote processing of Main Camera data during commissioning and operations.

Since the ``Catcher`` is a CSC it can interact directly with ``OCPS`` to process image data, by sending commands to it.

Alarms
------

Depending of the results of the analysis process the ``Catcher`` should be able to generate alerts to the system.
For this we can follow a similar approach to that already implemented in the ``Watcher``.
In short, the ``Catcher`` will have an "alarm" event with payloads to indicate the "severity" and other relevant information.
There will also be commands to mute and acknowledge these alarms.
We can also consider to borrow the alarm escalation model from the ``Watcher``.

Generate reports
----------------

The ``Catcher`` should also be able to generate reports, with graphs and additional lightweight computation.

One of the initial ideas was to rely on parameterized Jupyter notebooks that would be processed using a `papermill`_-like application, converted to a webpage and rendered online so users could inspect the results, and use as reference for additional followup.

.. _papermill: https://papermill.readthedocs.io/en/latest/

This could still be achieved using a similar solution or relying on a service like noteburst :sqr:`065` :cite:`SQR-065` and Times Square :sqr:`066` :cite:`SQR-066` to execute parameterized notebooks using a web request and publishing the results afterwards.

Another alternative is to implement a template system, where the user writes a template report with placeholders for text and images.
Then, the report can be generated by executing code to produce the placeholder values and images and exporting the report.
The report could be a webpage, a pdf (e.g. generated with LaTeX) or any other electronic format.

Service Architecture
====================

In summary our proposal is:

*  The ``Catcher`` will be an indexed CSC where each instance handle a single rule.
*  Rules will adopt the "configuration-based" approach.
   The ``Catcher`` codebase will have a series of "simple" rules that can be chained together to form an activation flag.
   The rule is only activated if all conditions are met.
*  When a rule is activated, it executes an analysis process.
   The analysis process should generate information about alarm level and data to assemble a report.
*  A report is generated using the output of the analysis process.
*  An alarm is issued indicating the severity level and information about how to access the report.
*  Whether a report is generated and/or an alarm is published every time the rule is activated or only at certain severity levels will be a configuration parameter.

Interface
---------

The following is an initial proposal for the (xml) interface of the ``Catcher`` and may be revised later.

Commands
^^^^^^^^

``mute``

   Mute this instance of the ``Catcher``.
   It will continue to process the data stream and generating reports, but will not issue new alarm unless they have severity higher than the one specified.

   Contains the following parameters:

   ``severity``

      Severity to mute alarms.
      Alarms will only be issued if severity is higher than this value.
   
   ``duration``

      How long to mute?
   
   ``mutedBy``

      Person who muted the alarm.

   ``reason``

      Reason alarm is being muted.

``acknowledge``

   Acknowledge alarm.

   Contains the following parameters:

   ``acknowledgedBy``

      Person acknowledging the alarm.

   ``id``

      Id of the process to acknowledge.

``showAnalysisProcess``

   Publish updated information about the analysis processes.

``showAlarms``

   Publish alarm information.

Events
^^^^^^

``alarm``

   Alarm issued when an analysis process results in a severity level above configured threshold.

   ``severity``
   
      Severity of the alarm.
   
   ``reason``

      Reason for the alarm to be issued.
   
   ``reportUrl``

      Url of the report generated by the analysis process.
   
   ``processId``

      Unique identifier for the process that resulted in this alarm.
   
   ``acknowledged``

      Was this alarm acknowledged?
   
   ``acknowledgedBy``

      Last person who acknowledged the alarm.
   
   ``acknowledgedAt``

      Time when the alarm was last acknowledged.

``analysisProcessSummary``

   Summary of the analysis processes.

   ``executing``

      Number of processes currently executing.

   ``executingIds``

      List with the ids of processes currently executing.

   ``finished``

      Number of processes that finished.

   ``finishedIds``

      List with the ids of finished processes.


``analysisProcess``

   Information about an analysis process.

   ``id``

      Id of the process.

   ``status``

      Status of the process.
   
   ``startedAt``

      When the process started.
   
   ``finishedAt``

      When the process finished.
   
   ``result``

      Result of the analysis process.


Configuration
-------------

The ``Catcher`` configuration will contain the configuration for all the rules and the CSC picks up a configuration based on its index.
The following is an example of having 2 instances of the ``Catcher``, with index 1 and 2.
The configuration is inspired by the use-cases presented in  the `FAFFv2 <https://sitcomtn-037.lsst.io/#deliverable-7-catcher-development>`__ report.

.. code-block:: yaml

   1:
      rule:
         monitor:
            topic: WeatherStation.tel_windDirection.value
            interval: 60.0
            threshold:
               max: 10.0
      analysis_process:
         analyze_stream: WeatherStation.tel_windDirection.value
            interval: 1800.0
            extrapolate_interval: 600.0
            threshold:
               ok: 15.0
               warning: 20.0
               critical: 30.0
      report_severity: warning  # Only produce report if warning level
      alarm_severity: warning  # Only issue alarm if warning level
   2:
      rule:
         monitor_event:
            component: ATCamera
            event: endReadout
      analysis_process:
         at_mount_jitter:
            threshold:
               ok: 0.5
               warning: 1.0
               critical: 2.0
            store_results: True
      report_severity: ok  # Always produce report
      alarm_severity: warning  # Only issue alarm if warning level


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
