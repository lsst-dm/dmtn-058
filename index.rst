..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. note::

   **This technote is not yet published.**

   Design notes for the DM Header Service

.. Add content here.

Motivation
==========

This technical note attempts to describe the DM Header Service for
LSST images acquired from LSSTCam, ComCam and Spectrograph. Here we
outline the services, behavior and infrastructure for the service. The
main motivation for this entity is to provide a consistent mechanism
for normal operations (which includes Prompt Processing and Archiver),
catch-up mode and other TBD uses.

The implementation of a DM Header Service has several
advantages. Here we list the main ones.

- Provides a single method for header (i.e. meta-data) acquisition for
  Real-Time and Catch-up modes.
- Provides uniform and consistent acquisition of meta-data for LSST
  visits and exposures for the TestStand, Spectrograph, ComCam, and LSSTCam.
- Ensures the creation of header files synchronously with the image
  data acquisition.
- Provides the analogous metadata for cross-talked files used for L1
  prompt processing and for raw files, including calibrations, being
  archived into the data backbone.
- It will run at the summit inside the Engineering and Facility
  Database (EFD) Cluster farm and will always have access to all of
  the live feeds from DDS.
- If the supporting EFD cluster farm is unavailable data taking is not possible.
- Supports the LSST Data Facility (LDF) services and the Data
  Management Control System (DMCS) and is available for any other image
  acquisition software.
- Headers will contain at a `minimum` the meta-data necessary for L1 Prompt
  Processing and archiving.
- Header should also contain a small set of `courtesy` meta-data. This
  is meta-data not considered crucial for prompt processing or
  archival operations that provides a context for the files.
- Provides a Service Abstraction Layer (SAL) Commandable Service (CSC).
- Separate instances of the Header Service will exist for the
  TestStand, Spectrograph, ComCam, and LSSTCam.

Description 
============

A description of the proposed functionality follows. Figure 1 shows a
high-level diagram of the proposed flow and interconnection with other
subsystems. In Figure 2 we present a more detailed schematic diagram
of the flow of the Header Service.

  .. figure:: /_static/HeaderService.svg
     :name: HeaderService

     Header Service Diagram and its relation to the DDS/SAL
     communication middleware and the EFD.

  .. figure:: /_static/HeaderService-Detailed.svg
     :name: Diagram_Detailed

     Header Service diagram in more detail, showing its relation to
     telemetry streams from other CSC such as the DMCS.
	    
The Header Service provides a single method of meta-data acquisition
for the two main operational scenarios: normal operations and catchup mode. 

Normal Operations
-----------------

Here we describe how the Header Service should work during normal
operations.

1. The Header Service will run at the summit and thus will
   always have access to the live telemetry and other feeds from
   DDS. It will subscribe to all of channels of interest to build an
   appropriate LSST header.
2. The Header Service will run inside the EFD cluster farm at
   the summit and will be considered a service within the critical
   operations enclave. Hence it should fulfill similar requirements of
   performance and availability as the DAQ, EFD and OCS.
3. Most likely at a camera readout Event (exact event TBD) the Header Service
   formats the metadata information collected during the exposure and
   generates a FITS header file per CCD. This is what the EFD refers to as a
   large file object (LFO).
4. The Header Service client publishes the existence of the LFO via
   DDS. This file should be named accordingly using the ``imageId``
   (or ``ImageName``) as part of the name using an Event like
   ``LargeFileObjectAvailable``.
5. The EFD automatically listens to all LFO announcement Events. When
   it sees one, it retrieves the file (protocol needs to be defined,
   but a URL is already included in the LFO announcement), and stores
   it in the EFD LFO annex. 
6. The EFD publishes an LFO announcement that it has the header
   file. Both summit and base EFD's will have a copy of the new header
   at this point. In the case of a communications outage T&S will sync
   the EFD's after summit/base are re-connected.
7. The DMCS retrieves the header from the EFD LFO annex at the Base using
   its imageID and a standard filesystem path to locate it.

Catchup Mode
------------

This stage represents the operational mode after a communication outage
between the summit and the base.

1. Once connection is re-established between summit and base the EFD
   tables and LFO annex files are synced.
2. DMCS can now retrieve all previously unprocessed or non archived images from the
   DAQ and using their ``imageID`` retrieve the already constructed FITS
   header as in step 7 in the previous section.

Meta-Data
=========

This list attempt to enumerate the data sources for the header
meta-data


Data Sources
------------

- In the current design a large fraction of the key/value pairs for the
  image headers will originate from static data that will come from
  configuration files, such as header templates that will be specific
  to each instrument. These can be initialized and/or defined via DDS/SAL events.
- Telemetry and Event data from the Camera Control System (CCS), Scheduler and
  the Telescope Control System (TCS) delivered via DDS. The
  Header Service will subscribe to a number of telemetry topics from
  which it will update key/value pairs with real-time values from
  DDS streams. Examples of this type include information regarding FILTER,
  TELRA, TELDEC, ImageID and visitID.
- Some additional meta-data not present in the telemetry or in the
  static data will have to be computed on-the-fly based on telemetry
  or other current information. The image AIRMASS is an example of
  this type of meta-data. This is expected to be a light-weight
  computation. Some of this additional information will be critical
  for production while the rest can be considered `courtesy`
  meta-data.
- TBD

Data Delivery and Interface
---------------------------

`The method for passing data from the Header to each Forwarder is not yet final.`

- The default method will be via a message. The Header Service will
  publish an Event like ``LargeFileObjectAvailable`` to DDS and the existence of
  a Large File Object (LFO) that will contain the location and name
  and the file that was inserted into the EFD LFO Annex.
- The Forwarders (DMCS) will retrieve the header from the EFD LFO
  annex using the ``ImageId`` and a standard filesystem path to locate
  it.

  .. figure:: /_static/SpectrographHeaderService.png
     :name: SpectrographHeaderService

     Header Service diagram using Archimate modeling language for the
     the Spectrograph.


Concerns
========

This is an incomplete list of concerns regarding the current design
that we list so they can be addressed in a timely manner before or
during the next design review. We separate this into primary (crucial) and
secondary concerns.


Primary concerns
----------------

- In the current design, the Header Service will gather the dynamic
  meta-data from telemetry streams at either the beginning of the
  integration, at the end of the readout of an exposure, or at both
  times. It is unlikely that more granular (in time) telemetry is needed for
  Prompt Processing or archiving.

- If the above statement is incorrect, we need to investigate in which
  cases this will not be satisfied and gather requirements for the
  amount of granularity required.

- We do not have set of required key/value pairs for header
  meta-data. It is important that before or during the design review
  we establish a procedure to define the content definition of the
  header files for all instruments that will be supported. We propose
  to start with an initial list based on standard key/value pairs from
  recent astronomy surveys that should be augmented to satisfy at a
  minimum the DM software and LDF production requirements.

Secondary concerns
------------------

- In the current design description the EFD retrieves the file published by
  the Header Service. Alternatively the Header Service could write
  directly into the Large File Annex filesystem.

- In the current design description the EFD publishes the path with
  the location of the new header file(s). Alternatively the DMCS could
  retrieved the header file as soon as the Header Service publishes
  the existence of the LFO. This would require that the DMCS have
  read access to the Large File Annex filesystem.

- Some light-weighted meta-data required for archiving or prompt
  processing will not be provided by the telemetry emanating from SCS
  or configuration files. We plan to compute these and additional
  `courtesy meta-data` on-the-fly and insert them to the
  headers. Examples of such cases are information regarding AIRMASS,
  NITE (as a string) and a default plate solution based on the
  telescope pointing.

- If the delivery method described above is not performant for L1
  Archiving and Prompt Processing, an alternative can be explored
  where the Header Client could pass fitsio FITSHDR Python Objects
  directly to the DMCS Forwarders.

Specifications and Requirements
===============================

Specifications
--------------

Here are the set of minimum specifications that are needed for a full
Design of the Header Service.

1.	Header definition for L1 Test Stand
2.	Header definition for Spectrograph
3.	Header definition for ComCam
4.	Header definition for LSSTCam
5.	Capture time definition for key/pairs values for L1 Test Stand
6.	Capture time definition for key/pairs values for Spectrograph
7.	Capture time definition for key/pairs values for ComCam
8.	Capture time definition for key/pairs values for LSSTCam


Requirements 
-------------

Here we present a list of proposed requirements, test and validation
of those.

  .. figure:: /_static/Requirements.png
     :name: Requirements

     Requirements Validation Matrix

Implementation
==============

The current implementation of the Header Service supports the creation
of headers for the Camera Stand is here:

https://github.com/lsst-dm/HeaderService

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
