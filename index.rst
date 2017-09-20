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

This technical note attempts to describe the DM Header Service
for LSST images. Here we outline the services, behaviour and
infrastructure for the service. The main motivation for this
service is to provide a consistent mechanism for 
normal operations and catch-up mode.

The implementation of a DM Header Service has several
advantages. Here we list the main ones.

- Provides a single method for header (i.e. meta-data) acquisition for Real-Time and Catch-up modes. 
- Provides uniform and consistent acquisition of meta-data for LSST visits and exposures for the Spectrograph, ComCam, and LSSTCam.
- Ensures the creation of header files synchronously with the image data acquisition.
- Provides the analogous metadata for cross-talked files used for L1 prompt processing and for raw files, including calibrations, being archived into the data backbone.
- It will run at the summit inside the Engineering and Facility Database (EFD) Cluster farm and will always have access to all of the live feeds from DDS.
- Supports the LSST Data Facility (LDF) services and the Data
  Managment Control System (DMCS) and is available for any other image
  acquisition software.
- Headers will contain at a `minimum` the meta-data necessary for L1 Prompt
  Processing and archiving.
- Provides a Service Abstraction Layer (SAL) Commandable Service (CSC)
  Communication Interfase for OCS.
- Separate instances of the Header Service will exist for the Spectrograph, ComCam, and LSSTCam.

Description 
============

A description of the proposed functionality follows. Figure 1 shows a
high-level diagram of the proposed flow and interconnection with other
subsystems. In Figure 2 we present a more detailed diagram of the
Header Service.

  .. figure:: /_static/HeaderService.svg
     :name: HeaderService

     Header Service Diagram and its relation to othe SAL components

  .. figure:: /_static/HeaderService-Detailed.svg
     :name: Diagram_Detailed

     Header Service diagram in more detail.
	    
The Header Service provides a single method of meta-data acquisition
for the two main operational scenarios: normal operations and catchup mode. 

Normal Operations
-----------------

Here we describe how the Header Service should work during normal
operations.

1. The Heder Service will run at the summit and thus will
   always have access to the live telemetry and other feeds from
   DDS. It will subscribe to all of channels of interest to build an
   appropriate LSST header.
2. The Header Service will run inside the EFD cluster farm at
   the summit and will be considered an `essential` service of telescope
   operations. Hence it should fulfill similar requirements of
   robustness as the DAQ, EFD and OCS.
3. Most likely at a readout Event (exact event TBD) the Header Service
   formats the metadata information collected during the exposure and
   generates a FITS header file. This is what the EFD refers to as a
   large file object (LFO).
4. The Header Service client publishes the existence of the LFO via
   DDS. This file should be named accordingly using the ``imageId`` (or
   ImageName) as part of the name using an Event like
   ``HeaderCreated`` (still TBD).
5. The EFD automatically listens to all LFO announcement Events. When
   it sees one, it retrieves the file (protocol needs to be defined,
   but a URL is already included in the LFO announcement), and stores
   it in the EFD LFO annex.  
6. The EFD publishes an LFO announcement that it has the header
   file. Both summit and base EFD's will have a copy of the new header
   at this point. (T&S will sync the EFD's after a summit/base
   outage).
7. DMCS retrieves the header from the EFD LFO annex at the Base (using
   its imageID and a standard filesystem path to locate it).
8. Alternatively the DMCS could retrievd the header file as soon as
   the Header Service publishes the existence of the LFO.

Catchup Mode
------------

This stage represent the operational mode after a commumication outage
between the summit and the base.

1. Once connection is reestablished between summit and base the EFD
   tables and LFO annex files are synced.
2. DMCS can now retrieve all previously unprocessed or non archived images from the
   DAQ and using their ``imageID`` retrieve the already constructed FITS
   header as in step 7 (8) in the previous section.

Meta-Data
=========

This list attempt to enumerate the data sources for the header
meta-data


Data Sources
------------

- In the current design a large fraction of the key/value pairs for the
  image headers will originate from static data that will come from
  configuration files, such as header templates that will be specific
  to each instrument. These can be initialized and/or defined via SAL events.
- Telemetry and Event data from the Camera Control System (CCS), Scheduler and
  the Telescope Control System (TCS) delivered via DDS/OpenSplice. The
  Header Service will subscribe to a list of telemetry topics from
  which it will update key/value pairs with real-time values from
  streams. Examples of this type include information regarding FILTER,
  TELRA, TELDEC, ImageID, visitID.
- Some additional meta-data not present in the telemetry might have to
  be computed on-the-fly based on Telemetry, for example AIRMASS. This
  is expected to be a light-weight computation.
- TBD

Data Types
----------

- Static data is data that will stay the same during the course of a
  night, such as configuration details. Examples are who the operators currently
  are, Telescope geographic location, night.
- Dynamic data is changing data that is not associated with individual
  exposure details. Dome ambient temperature and airmass details are
  example of dynamic data. However, airmass will not be provided as
  telemetry and dome temperature most likely is not needed for image processing.
- Visit/exposure data includes descriptive values that specify for the
  tile for this visit location, or pointing RA,DEC, FILTER, visitID
  etc.
- Additional data directly coupled to individual exposures, such as ImageID, EXPTIME, etc.
- Some data will be generated/computed by on-the-fly using existing
  meta-data. Examples are the CCD number, Session ID, AIRMASS, etc.

Data Delivery and Interface
-------------

- The method for passing data from the Header to each Forwarder is not
  yet final.
- The default method will be via a message. The Header Service will
  publish an Event like ``HeaderCreated`` to DDS and the existence of
  a Large File Object (LFO) that will contain the location and names
  and the files that were inserted into the EFD LFO Annex.
- The Forwarders (DMCS) will retrieve the header from the EFD LFO
  annex using the ``ImageId`` and a standard filesystem path to locate
  it.
- If above is not performant for L1 Archiving and Prompt Processing,
  an alternative can be explored where the Header Client could pass
  fitsio FITSHDR Python Objects directly to the DMCS Forwarders.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
