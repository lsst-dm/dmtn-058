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

This Technical attempts to describe a proposed DM Header Service
for LSST images. Here we outline the services, behaviour and
infrastructure for the proposed client. The main motivation for this
proposal is to provide a consistent mechanism for catch-up and
real-time operations.

The implementation of a DM Header Service several
advantages. Here we list the main ones.

- Provides a single method for header acquisition for Real-Time and Catch-up modes. 
- Provides uniform and consistent acquisition of meta-data for LSST visits and exposures for the Spectrograph, ComCam, and LSSTCam.
- Ensures the creation of header files synchronously with the image data acquisition.
- Provides the analogous metadata for cross-talked files used for L1 prompt processing and for raw files, including calibrations, being archived into the data backbone.
- It will run at the summit inside the EFD Cluster farm and will always have access to all of the live feeds from DDS.
- Supports the LSST Data Facility (LDF) services and is available for
  any other image acquisition software.
- Headers will contain at least the meta-data necessary for L1 Processing and archiving


Description 
============

A high level description of the proposed functionality follows. Figure 1 show a diagram of the proposed flow and interconnection with other subsystems.

  .. figure:: /_static/HeaderSevice.png
     :name: Diagram

Normal Operations
-----------------

1. The header meta-data client will run at the summit and thus will always have access to the live telemetry and other feeds from DDS. It will subscribe to all of channels of interest to build an appropriate LSST header.
2. The header meta-data client will run inside the EFD cluster farm at the summit and will be considered an essential service of telescope operations. Hence it should fulfill similar requirements of robustness as the DAQ and EFD.
3. Most likely at a readout Event (event subject to change) the header meta-data client formats the metadata information collected during the exposure and generates a FITS template header file. This is what the EFD refers to as a large file object (LFO).
4. The meta-data header client publishes the existence of the LFO via DDS. This file should be named accordingly using the imageId as part of the name using an Event like HeaderCreated.
5. The EFD automatically listens to all LFO announcement Events. When it sees one, it retrieves the file (protocol needs to be defined, but a URL is already included in the LFO announcement), and stores it in the EFD LFO annex. 
6. The EFD publishes an LFO announcement that it has the header file. Both summit and base EFD's will have a copy of the new header at this point. (T&S will sync the EFD's after a summit/base outage).
7. DM retrieves the header from the EFD LFO annex at the Base (using its imageID and a standard filesystem path to locate it).

Catchup mode
------------

Once connection is reestablished between summit and base the EFD tables and LFO annex files are synced.
Retrieve all previously unprocessed or non archived images from the DAQ and using their imageID retrieve the already constructed FITS header as in step 7, above.

.. figure:: /_static/HeaderService-Detailed.png
     :name: Diagram_Detailed

Data
====

This list attempt to enumerate the data sources for the header
meta-data

Data Sources
-------

- Configuration Information from files (for examples header templates)
- Telemetry and Event data from the Camera Control System and TCS delivered via DDS/OpenSplice
- Possible Telemetry and Event data from other telescope functional areas such as Scheduling, weather station, dome environmental details...
- Some data will be from values that DM is using, such as the type of correction being used, Image ID, Visit ID, CCDNUM, date/time, etc
- TBD

Data Types
----------

- Static data is data that will stay the same during the course of a night, such as configuration details, who the operators currently are, etc.
- Dynamic data is changing data that is not associated with individual exposure details. Dome ambient temperature and airmass details are example of dynamic data.
- Logical Visit data includes descriptive values that specify the sky tile for this visit location, or pointing RA,DEC, etc.
- Live or Exposure data is data directly coupled to individual exposures, such as Image ID, Filter values, shutter values, etc.
- Some data is bookkeeping data that will be generated by DM. Examples are the CCD number, Session ID, WCS information, etc.

Data Delivery
-------------

- The method for passing data from the Header to each Forwarder is not yet final.
- The default method will be via a message. The Header Client will publish an Event like HeaderCreated to DDS and the existence of a Large File Object (LFO) that will contain the location and names and the files that were inserted into the EFD LFO Annex.
- The Forwarders (DMCS) will retrieve the header from the EFD LFO annex using the ImageId and a standard filesystem path to locate it.
- If above is not sufficiently fast for L1, an alternative can be explored where the Header Client could pass fitsio FITSHDR Python Objects directly to the Forwarders.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
