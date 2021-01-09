..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
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

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   This note describes the design of a client/server Butler.

.. Add content here.

Introduction
============

:cite:`DMTN-169` describes a need for a Butler user to be able to access the registry and datastore using the Rubin Science Platform authentication system and not rely on direct SQL access to a Butler Registry.
To implement this scheme it is necessary to have a version of Butler that forwards registry calls to a server with the authentication and authorization control being implemented in that server.
This would, for example, mediate access to Butler collections (both in registry and object store) and respect group management allowing multiple users to access shared collections.
It would also allow write access to specific collections without requiring detailed access controls to be implemented in the SQL database.

Requirements
============

* Any client/server Butler implementation should not require any different code path for the user than using a local Butler or a Butler that uses direct SQL access to the Registry.
* A Butler YAML configuration file that declares the server details is all that is required to enable client/server Butler. This configuration could be retrieved from the server itself (but would not be identical to the configuration for the server itself).
* The server should use the same A&A system as used by the Rubin Science Platform.
* The client can assume that the server has already been configured and has all relevant datastore tables. This means that there is no requirement to support the registry table creation APIs.

A design goal is to make the server interface as usable as possible for users who do not want to use the Python ``Butler`` class.
It is conceivable, for example, that some of the registry query methods are genuinely useful for building up tables of results in the portal from a web site natively using Javascript.
This is not possible if the server API is so low-level that significant butler logic has to be layered on top of the results to make them useful.

Overview
========

The Butler is implemented as three distinct components.

1. A ``Datastore`` that is responsible for reading and writing datasets to an object store or local filesystem.
2. A ``Registry`` that records the relationship between datasets and how they relate to astronomical concepts such as the observing filter, instrument, or region on sky.
3. A ``Butler`` class that combines the ``Datastore`` and ``Registry`` to allow a user to fetch and store datasets whilst retaining their relationships to astronomical concepts.

In general a user will never interact directly with the underlying ``Datastore`` methods (always going through ``Butler``) but can be expected to interact with ``Registry`` when querying the system to determine what datasets or dimensions are available and also to define collections of datasets.

The default Registry implementation involves the use of manager classes that mediate access to SQL database tables.
There are, for example, distinct manager classes for managing collections, datasets, dimensions, and the datastore usage (Datastore uses Registry to record where a dataset was stored in the file system or object store).
These manager classes are versioned and can be individually declared in the butler configuration file.
This plugability simplifies adoption of new schemas and implementations in the future whilst leaving existing repositories with the older versions.

Implementation Options
======================

Client/Server Registry
----------------------

One option for implementation is to write a new subclass of ``~lsst.daf.butler.Registry`` where the public methods are implemented as thin methods that convert the parameters to JSON and then use a simplified call to a server.
The server would then take the serialized JSON, convert it back to the appropriate Python types and then use the normal ``Registry`` implementation to do the communication with the SQL database.
The results would then be converted back to JSON and sent back to the client before being converted back to the expected Python type.

A key concept of a registry is the concept of a "dimension universe" that describes the relationship between all the scientific concepts.
This is stored in the database in JSON and retrieved by the client when a connection is made.
StorageClass definitions would be retrieved via the Butler YAML configuration file.

.. note::
  There is currently no API to support a StorageClass defined solely in the client being registered with the server.
  This would currently require that the ``storageClasses.yaml`` definitions on the server be updated if the server was given a ``DatasetType`` that included a new definition.
  That is not possible at this time.

When serializing a ``DatasetRef`` the universe would not be included and would initially be assumed to match the universe in the server.

Registry has public methods that take the following types as arguments:

* ``str``
* ``int``
* ``bool``
* ``set``
* ``re.pattern``
* ``Timespan``
* ``DatasetType`` (and sequence thereof)
* ``DatasetRef`` (and sequence thereof)
* ``DataId``
* ``Dict[str, Any]`` where ``Any`` is a scalar type that can be stored in a database table.
* ``DimensionGraph``
* ``Dimension``
* ``DataCoordinate``
* ``DimensionElement``
* ``DimensionRecord``
* ``QuerySummary``
* ``CollectionType``

and can return:

* ``DataCoordinate``
* ``DatasetRef``
* ``DatasetType``
* ``Dict[str, Any]`` where ``Any`` is a scalar type that can be stored in a database table.
* ``CollectionSearch``
* ``CollectionType`` enum
* ``str`` or sequence thereof
* ``bool``
* ``DatastoreRegistryBridgeManager``
* ``QueryBuilder``
* ``DataCoordinateQueryResults``
* ``DatasetAssociation``

There will be no requirement for the client to be able to register an opaque table since that should already have been done on the server side.

Almost all of these will have to be converted into JSON representations over the wire.

Datastore
^^^^^^^^^

The ``Datastore`` implementation is complicated by the fact that it does not use the ``Registry`` APIs directly but instead uses a ``DatastoreRegistryBridge`` class.
This API uses direct database SQL queries and would also have to be replaced with a client/server implementation.
It is also possible that a special remote Datastore subclass could be written that reimplements the few methods that require direct table access (``getStoredItemsInfo``, ``addStoredItemsInfo``, ``removeStoredItemsInfo``, and ``_registered_refs_per_artifact``) as REST calls on the server.

The dataset artifact side of Datastore already knows how to transfer data to and from a remote object store or file server via the generic ``ButlerURI`` implementation so there is no need to duplicate that support into a special butler server.

What will be required though is to include in the butler server a means for translating the URIs stored in the datastore registry to `signed URLs`_.
The URIs would likely be written as ``gcs://`` or ``s3://`` form in the registry and would be converted to signed ``https`` URLs for use by the client.
For read this could be handled directly in the server implementation of ``getStoredItemsInfo`` such that the URIs are always returned as signed URLs.
For write, the URI is constructed using the file template and ``LocationFactory`` called from ``Datastore._prepare_for_put()``.
This code would have to be factored out into its own method such that a client ``Datastore`` subclass could ask the server to convert that URI to a signed URL.
It has to be decided whether file templates are specified by the client or by the server configuration.

.. _signed URLs: https://cloud.google.com/storage/docs/access-control/signed-urls

Example Configuration
^^^^^^^^^^^^^^^^^^^^^

The Butler should be configured by the URL of the server and the server should have a ``butler.yaml`` file available at its root.
This configuration file is not the configuration of the server itself but is the configuration that clients should use.
Its contents should be simple and could be something like:

.. code-block:: yaml

  datastore:
    cls: lsst.daf.butler.datastores.client.ClientDatastore
    root: <butlerRoot>
  registry:
    cls: lsst.daf.butler.registry.RegistryClient
    db: <butlerRoot>
  storageClasses:
    # Storage classes known to the server

where ``<butlerRoot>`` would automatically be replaced by the URL of the server.

Problems
^^^^^^^^

Transaction handling might be an issue since it is very hard to implement rollbacks of registry changes if there is a problem on the datastore side without requiring that the connection to the server stays open.
This is a particular issue during ``Butler.put()`` and to a lesser extent ``Butler.pruneDatasets()``.

Client/Server Butler
--------------------

Another option is to change the ``Butler`` constructor to be a factory method that can return a classical ``Butler`` or a ``RemoteButler``.
This remote butler would implement the get, put, ingest, and prune methods directly as calls to the remote server.
It would be easier to handle transactions in this context beacuse all of the required work that requires the transactions would be handled by the server since the APIs are at a much higher level than registry.

Implementing a client/server ``Butler.put()`` is difficult because there is no way to convert the Python in-memory dataset to serialized form without involving the formatter infrastructure that is called within ``Datastore``.
One option is for the server implementation of ``Butler.put()`` to not take the dataset at all, but to instead store a placeholder entry in the datastore registry and then return a signed URL, along with the ``DatasetRef``, that can be used by the client code to push the file.
The client code would then use a local Datastore implementation to create the file from the relevant formatter and then upload it.
The downside of this approach is that it is not immediately clear how to handle composite disassembly (where the file is split into multiple components and each is stored separately in the file store) since that is a datastore configuration.

A ``Butler.get()`` would most logically be handled as a call to ``Butler.getURIs()`` (probably via the datastore ``getStoredItemsInfo`` method) to obtain the signed URL, download the file, and then use a local Datastore implementation to read that file.
This could conceivably leverage the purported "caching datastore" concept.

Even if all this is made to work, users still expect to be able to use many of the registry methods for querying the collections, datasets and dimension records.
This suggests that it might be better to implement the client/server registry first and build on that, and then subsequently add explicit ``Butler`` overrides if performance is an issue (for example if ``Butler`` calls registry methods in a loop there may be significant run time overheads).

Manager Classes
---------------

The final option is to implement each of the manager classes as client/server implementations.
The registry is already designed to support pluggable managers and they are already designed to isolate database access.
This seems like the cleanest way forward but the interfaces are very low level and this makes it significantly harder for non-Butler clients to do anything useful with the data being returned.
It also raises the possibility of the interfaces being very slow when called in loops and may require significant caching of results and also the addition of new methods that move loops into the server.


Access Controls
===============

Part of the benefit of using a client/server approach is that the server can control access to collections and datastores without having to use fine-grained database permissions on specific tables or add ACLs to the object store.

This does though mean that there must be code in the server that can take the user name and determine which information can be used.
This does not simply mean checking that the collection name includes the user name since the checks must also be able to look at collections that have group access controls (one person may wish to give access of their processed data to another user but no-one else).




.. Do not include the document title (it's automatically added from metadata.yaml).

.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
