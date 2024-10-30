# SIAv2 over Butler FastAPI service

```{abstract}
In this technote we describe a proposed design for implementing an IVOA Simple Image Access Version 2.0 service directly over a Butler repository using a Safir/FastAPI application and the dax_obscore package.
```

## 1. Introduction

The existing CADC-based implementation which uses the ObsCore table is limited by the fact that QServ does not support certain ADQL functions required by the implementation, specifically the <em>INTERSECTS</em> function.
This alternative approach circumvents this by using the [Butler](https://github.com/lsst/daf_butler) {cite:p}`2022SPIE12189E..11J` & [dax_obscore](https://github.com/lsst-dm/dax_obscore) package to generate the links for a given query without requiring access through the [ObsCore](https://www.ivoa.net/documents/ObsCore/) table.
This has several potential benefits including performance, simplicity & decoupling of the various components, allowing Butler changes to take immediate effect and appear in the ObsCore results, without having to be first propagated to the <em>ivoa.ObsCore</em> Table.

## 2. Prototype Requirements

The prototype is meant to demonstrate that the most important search capabilities can be implemented directly over the Butler registry database and the existing Butler Python APIs
- Support at least one positional-search query parameter (see above).
- Support at least one of an exposure time (not duration) and/or wavelength-coverage query parameter.
- Return the absolute minimum set of ObsCore columns to support a reasonable display of the results in Firefly.
This is believed to be <em>s_ra</em> , <em>s_dec</em> , <em>s_region</em> , <em>access_url</em> , and <em>access_format</em>. Supplying some time and wavelength information as well would be very useful.
- Support either the direct-URL or RSP-datalinker (CADC-based) model for returning a pointer to the actual image data (eventually should support both).
- May be implemented over either a "local" or "client-server" Butler (eventually should support both)


## 3. Implementation Goals

This design satisfies the following high-level goals:
- Follows SQuaRE’s process for web APIs to use FastAPI & publish an OpenAPI v3 service description
- Decouples the process of generating the image links from the IVOA API layer which clients will interact with, allowing them to be scaled, modified as well as tested individually.
- We will be using python, the preferred implementation language of the Rubin observatory
- Service will not require the full Rubin Observatory Stack.


## 4. Architecture Summary

The SIAv2 application will be another FastAPI Python service running in the RSP as Kubernetes a deployment, and will as other services use Gafaelfawr for authentication and authorization.

Queries to the SIAv2 query endpoint will interface with and interact with the <em>dax_obscore</em> middleware package which uses <em>Butler</em> to fetch the relevant image links and return them in the ObsCore format expected by the SIA v2 protocol.

The SIA service will support multiple Butler repository configurations, with two access modes:

1. Direct Mode: Will connect directly to Butler's PostgreSQL database, requiring credential configuration through Phalanx
2. Remote (Client/Server) Mode: Will use Butler's client/server interface, requiring no additional credentials.
   In this case authentication is handled by parsing the user token and attaching it to the request to the Butler server.

The Butler access mode will be specified in the Phalanx configuration to ensure proper secret management for Direct Mode connections.
Individual repository configurations will specify their mode type, which will be used by the Butler Factory to instantiate the appropriate Butler object.
The service implementation will abstract the Butler access mode from application logic where possible.
During startup, frequently accessed data like repository-specific ObsCore configurations will be cached to optimize performance.


Initially no persistent storage will be required as the results will be streamed to the HTTP client but not stored locally.

The initial implementation will use a single-tier architecture without worker nodes, as the Butler server handles the intensive computations.
Performance bottlenecks are expected to be primarily I/O-bound, so the service is designed with asynchronous support where possible.
While the current middleware packages don't support async operations, we've implemented a future-proof design:

- Query processing logic will  be encapsulated in a method that can operate either synchronously or asynchronously.
- All preparation tasks will be built with async support.
- Assuming we do have preparation tasks (for example monitoring tasks), the FastAPI endpoints will be implemented asynchronously and preparation tasks will be run before the query processing handler.
- Blocking operations will be handled using Starlette's <em>run_in_threadpool</em>, preventing event loop blockage.

This architecture will allow for a smooth transition to fully async operations when middleware support becomes available.



```{figure} diagram.png
:figclass: technote-wide-content

System overview
```


## 5. Protocol Summary

Summary of the SIA v2 protocol from https://www.ivoa.net/documents/SIA/:

The SIAv2 IVOA standard defines a web service interface for discovering and retrieving image data from archives and data collections.

It enables users to search for images based on various metadata criteria, such as sky position, time of observation, wavelength range, etc.
The protocol defines a core set of query parameters and response metadata, but also allows for extensions to support additional features or data types as needed by specific archives or communities.
SIA services are implemented as RESTful web services with a {query} resource for data discovery and an optional {metadata} resource for obtaining detailed dataset metadata conforming to the ImageDM.
Both of these resources are synchronous resources conforming to the DALI-sync [1] specification.

An SIA service must have at least one {query} resource; it could have multiple {query} resources (e.g., to support alternate authentication schemes where the path is different).


Required Endpoints:

- **{query}**
There is no requirement on what to name this endpoint.
This assumes that the endpoint can be found via the capabilities endpoint.
This endpoint is synchronous and params can be submitted either via POST or GET HTTP requests
- **/availability**
Return the appropriate response indicating whether the status is currently available or not for use
- **/capabilities**
List the capabilities of the service in the standard VOSI capabilities XML format.
An example of the minimum capabilities expected can be found here:


The specification also defines a list of parameters and states that:

<em>All parameters for the {query} resource defined below must be supported by the service.
Services must accept parameters and apply the constraints such that if a (ObsCore) record does not satisfy the constraints it is not included in the response.
If the metadata for a field is not known (null), the constraint cannot be satisfied.
The ObsCore data model [7] defines which fields may be null and which must have a value.
For example, if dataset(s) have unknown time coverage (`t_min` and `t_max` in ObsCore), a query with the TIME parameter must not return the record(s); queries without the TIME constraint could still return such records, so the caller can discover such dataset(s).</em>

Client requests may include zero or more of the query parameters.



## 6. API Service

The service frontend providing the SIAv2 API will use the FastAPI framework.

### 6.1 Query

**Endpoint**: /query

**HTTP Method**: GET & POST

**Parameters**:
<table class="custom-table" style="max-width:100%; white-space: collapse;">
  <thead>
    <tr>
      <th><strong>Parameter</strong></th>
      <th><strong>Arguments</strong></th>
      <th><strong>ObsCore target</strong></th>
      <th><strong>RSP interpretation</strong></th>
      <th><strong>Butler Registry metadata</strong></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>POS</strong></td>
      <td>CIRCLE, POLYGON, or RANGE<br>Units: degrees<br>Always defined w.r.t. ICRS coordinates in degrees.<br>"Valid coordinate values are in [0,360] for longitude and [-90,90] for latitude"</td>
      <td>s_region</td>
      <td>As expected;<br><strong>critical MVP capability</strong>.</td>
      <td>Spatial coverage is native to Butler operation; Some concern about the accuracy of image-boundary data in the Butler in practice; must be resolved at Butler level, not in the SIAv2 service</td>
    </tr>
    <tr>
      <td><strong>BAND</strong></td>
      <td>Units: meters<br>Can be a single value or a value pair.<br>+Inf/-Inf must be supported in pairs.</td>
      <td>em_min, em_max</td>
      <td>As expected;<br><strong>critical MVP capability</strong>.</td>
      <td>Butler understands filter name for single-epoch images and bands for co-adds. Configuration maps wavelengths to filter and thence to band.</td>
    </tr>
    <tr>
      <td><strong>TIME</strong></td>
      <td>Units: days with MJD epoch, UTC time scale<br>Can be a single value or a value pair.</td>
      <td>t_min, t_max</td>
      <td>As expected;<br><strong>critical MVP capability</strong>.</td>
      <td>Temporal coverage is native</td>
    </tr>
    <tr>
      <td><strong>POL</strong></td>
      <td>Single value from I/Q/U/V/RR/LL/etc. (see ObsCore)</td>
      <td>pol_states</td>
      <td>Returns nothing, if specified.</td>
      <td>N/A</td>
    </tr>
    <tr>
      <td><strong>FOV</strong></td>
      <td>Units: degrees<br>Must be a value pair.<br>+Inf/-Inf must be supported.</td>
      <td>s_fov</td>
      <td>Should be supported based on an approximate FOV computed from the</td>
      <td>Not natively supported; mapped from detailed spatial coverage</td>
    </tr>
    <tr>
      <td><strong>SPATRES</strong></td>
      <td>Units: arcseconds<br>Must be a value pair.<br>+Inf/-Inf must be supported.</td>
      <td>s_resolution</td>
      <td>Not very interesting since our pixel scale is approximately the same for both single-epoch and coadded images.</td>
      <td>Not explicitly available; mapped from dataset type</td>
    </tr>
    <tr>
      <td><strong>SPECRP</strong></td>
      <td>Dimensionless resolving power<br>Must be a value pair.<br>+Inf/-Inf must be supported.</td>
      <td>em_res_power</td>
      <td>Returns nothing, if specified.</td>
      <td>N/A at least at first. If SIAv2 is extended to DAP as expected and we make time series data available explicitly, then it may one day be supported. Could be mapped from dataset type and filter for LATISS spectra (NB: interpretation for SPHEREx LVFs is TBD)</td>
    </tr>
    <tr>
      <td><strong>EXPTIME</strong></td>
      <td>Units: seconds<br>Must be a value pair.<br>+Inf/-Inf must be supported.</td>
      <td>t_exptime</td>
      <td>Should work as expected.</td>
      <td>Works for single-epoch images. Does not work for co-adds since that metadata is not available.</td>
    </tr>
    <tr>
      <td><strong>TIMERES</strong></td>
      <td>Units: seconds<br>Must be a value pair.<br>+Inf/-Inf must be supported.</td>
      <td>t_resolution</td>
      <td>Returns nothing, if specified.</td>
      <td>N/A at least at first. If SIAv2 is extended to DAP as expected and we make time series data available explicitly, then it may one day be supported.</td>
    </tr>
    <tr>
      <td><strong>ID</strong></td>
      <td>string<br><strong>case-insensitive</strong></td>
      <td>obs_publisher_did</td>
      <td>As expected;<br><strong>critical MVP capability</strong>.</td>
      <td>IVO identifier will map to Butler dataset UUID within a named butler repository. A dataset can exist in multiple repositories if transferred.<br>Must be usable to construct query URLs referring to specific images.<br>Contrary to the nudge in the ObsCore spec, we probably want the same ID to work across multiple sites.</td>
    </tr>
    <tr>
      <td><strong>COLLECTION</strong></td>
      <td>string<br>case-sensitive</td>
      <td>obs_collection</td>
      <td></td>
      <td>Cannot map "generically" onto Butler collection names, as in ObsCore a given dataset can only be in a single collection, and limiting the functionality of COLLECTION= to Butler <strong>run collections</strong> would be user-hostile.</td>
    </tr>
    <tr>
      <td><strong>FACILITY</strong></td>
      <td>string<br>case-sensitive</td>
      <td>facility_name</td>
      <td>Intent is to distinguish real and simulated data here</td>
      <td>Uses [AAS facility keywords](https://journals.aas.org/facility-keywords/). Mapped from Butler instrument.</td>
    </tr>
    <tr>
      <td><strong>INSTRUMENT</strong></td>
      <td>string<br>case-sensitive</td>
      <td>instrument_name</td>
      <td>Functions in the obvious way</td>
      <td>Uses Butler instrument names.</td>
    </tr>
    <tr>
      <td><strong>DPTYPE</strong></td>
      <td>string, taken from limited vocabulary<br>case-sensitive</td>
      <td>dataproduct_type</td>
      <td>Strictly speaking only "image" and "cube" are meaningful; the other ObsCore dataproduct types may become usable in a future "DAP" evolution of SIAv2. At present we don’t have "cube" data at all.</td>
      <td>Mapped from dataset type via configuration.</td>
    </tr>
    <tr>
      <td><strong>CALIB</strong></td>
      <td>Integer between 0 and 4</td>
      <td>calib_level</td>
      <td>Roughly: 1: raw, 2: calexp, 3: everything else</td>
      <td>Mapped from dataset type via configuration.</td>
    </tr>
    <tr>
      <td><strong>TARGET</strong></td>
      <td>string<br>case-sensitive</td>
      <td>target_name</td>
      <td>TBD; perhaps initially:<br>Returns nothing, if specified?<br>Is it possible to link to a scheduler concept?<br>(E.g., the original "field center" concept, or perhaps just a broad category meaning something like "main survey" vs. "DDF" vs. one of the "extensions" like the NES?)<br>Perhaps this is only meaningful for TOO observations?</td>
      <td>Butler has target name for single-epoch images. No target name for coadds. Tract/patch coordinate is equivalent.</td>
    </tr>
    <tr>
      <td><strong>FORMAT</strong></td>
      <td>string<br>case-sensitive</td>
      <td>access_format</td>
      <td>Intended to be the format of the data returned (e.g., "FITS") but this is incompatible with the spec when used "DataLink style" (as Rubin and CADC do). If used DataLink-style, always the DataLink MIME type.</td>
      <td>If used direct-URL style, not via DataLink, mapped from dataset type</td>
    </tr>
    <tr>
      <td><strong>MAXREC</strong></td>
      <td>Non-negative integer</td>
      <td>N/A</td>
      <td>Overflow indicator must be set if value is positive and the query result is <strong>actually</strong> truncated.</td>
      <td>Implemented in dax_obscore.</td>
    </tr>
  </tbody>
</table>

<h3>Recommended extensions:</h3>

<table class="custom-table">
  <thead>
    <tr>
      <th><strong>Parameter</strong></th>
      <th><strong>Arguments</strong></th>
      <th><strong>ObsCore target</strong></th>
      <th><strong>RSP interpretation</strong></th>
      <th><strong>Butler Registry metadata</strong></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>DPSUBTYPE</strong></td>
      <td>string case-sensitive</td>
      <td>dataproduct_subtype</td>
      <td>As in ObsTAP</td>
      <td>Maps to "lsst." + datasetType</td>
    </tr>
  </tbody>
</table>


Note: The API params need to be case-insensitive to follow the IVOA recommendations.

**Response (Success)**:

**HTTP status code**:  200 (OK)

**Content-Type**: application/x-votable+xml

**Content**:

The content of a successful query is a table consistent with ObsTAP responses.
The response should contain all the required ObsTAP fields and may contain additional fields outside the defined ObsTAP data model.
An initial set of metadata fields identified as the minimum list include: s_ra , s_dec , s_region , access_url , and access_format.
However this may require further investigation.

Regarding returning datalinks in the response, relevant here is the following section from the spec:

<em>
If the provider implements a DataLink service for the data being found via SIA, the {query} response should include a description for invoking the DataLink service, usually using values from the obs_publisher_did column.

If the data provider implements a DataLink service for the data being found via the SIA {query} capability, they may put a URL to invoke the DataLink {links} capability (with ID parameter and value) in the access_url column; if they do this, they must also put the standard DataLink MIME type [9] in the access_format column.
</em>

To indicate that this is a URL to a DataLink service, the access_format column is set to application/x-votable+xml;content=datalink

**Response (Failure)**:

**HTTP status code**:  200 (OK)

**Content-Type**: application/x-votable+xml

**Content**:

A failed query should produce a response where the file format matches the requested format.
The possible error codes are:

- **UsageFault**: Invalid input (e.g., invalid input parameter value)
- **TransientFault**: Service is not currently able to function
- **FatalFault**: Service cannot perform requested action
- **DefaultFault**: General error (not covered above)


### 6.2 Availability

**Endpoint**: /availability

**HTTP Method**: GET

**Response**:

**HTTP status code**:  200 (OK)

**Content**: VOSI-availability XML content describing the status of the availability of the service.

**Example**:

```
<availability xmlns:vosi="http://www.ivoa.net/xml/VOSIAvailability/v1.0">
  <available>true</available>
  <note>The SIAv2 service is accepting queries</note>
</availability>
```

Assuming the SIAv2 service is running without issues the availability endpoint should in theory always respond with `available=true`. However, as the service depends on Butler being available and able to fetch images, one possibility here is to have the availability status be generated through a health check of the Butler client/server connection & repository.
This assumes a health-check endpoint being available on the server which may be a part of a future improvement rather than the initial implementation.

### 6.3 Capabilities

**Endpoint**: /capabilities
**HTTP Method**: GET
**Response**:
**HTTP status code**:  200 (OK)
**Content**:
List the capabilities of the service in the standard VOSI capabilities XML format.

**Example**:

```
<?xml version="1.0" encoding="UTF-8"?>
<vosi:capabilities
 xmlns:vosi="http://www.ivoa.net/xml/VOSICapabilities/v1.0"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:vs="http://www.ivoa.net/xml/VODataService/v1.1">
 <capability standardID="ivo://ivoa.net/std/VOSI#capabilities">
   <interface xsi:type="vs:ParamHTTP" version="1.0">
     <accessURL use="base">http://example.com/sia2/capabilities</accessURL>
   </interface>
 </capability>
 <capability standardID="ivo://ivoa.net/std/VOSI#availability">
   <interface xsi:type="vs:ParamHTTP" version="1.0">
     <accessURL use="full">http://example.com/sia2/availability</accessURL>
   </interface>
 </capability>
 <capability standardID="ivo://ivoa.net/std/SIA#query-2.0">
   <interface xsi:type="vs:ParamHTTP" role="std" version="2.0">
     <accessURL>http://example.com/sia2/query</accessURL>
   </interface>
   <!-- service details from extension schema could go here -->
 </capability>
</vosi:capabilities>
```

### 6.4 Examples Endpoint

In this initial iteration we will not be implementing the /examples (DALI-example) endpoint, but this can be easily added later.

### 6.5 Other Result Formats

The result format will initially be VOTable, but may be extended to support additional formats in the future.
The implementation should support a plugin architecture where different formats can be plugged in and added with minimal changes to existing code.

### 6.6 Service Self-Description

The SIAv2 implementation will include service self-descriptions with the VOTable response, which include information and descriptions of allowed ranges and values for certain parameters.
Core examples where this will be useful in describing BAND (i.e., u/g/r/l/z/y) & COLLECTION (all available or just the “publicized”).
To obtain a self-description VOTable response, clients can specify `MAXREC=0`.

## 7. Obscore over dax_obscore API

This section describes the API through which the SIAv2 FastAPI app will interact with dax_obscore (& Butler) to retrieve the relevant Obscore table for each query.

```python
def siav2_query(
    butler: Butler,
    config: ExporterConfig,
    parameters: SIAv2Parameters,
    *,
    collections: Iterable[str] = (),
    dataset_type: Iterable[str] = (),
) -> astropy.io.votable.tree.VOTableFile:
    """Run SIAv2 query with parsed parameters and return results as VOTable.

    Parameters
    ----------
    butler : `lsst.daf.butler.Butler`
        Butler repository to query.
    config : `ExporterConfig`
        Configuration for this ObsCore system.
    parameters : `SIAv2Parameters`
        Parsed SIAv2 parameters.
    collections : `~collections.abc.Iterable` [ `str` ]
        Optional collection names, if provided overrides one in ``config``.
    dataset_type : `~collections.abc.Iterable` [ `str` ]
        Names of dataset types to include in query.

    Returns
    -------
    votable : `astropy.io.votable.tree.VOTableFile`
        Results of query as a VOTable.
    """
```
Note: This API is subject to change and may be extended in the future.

The SIAv2Parameters model encapsulates all possible SIAv2 Query parameters as described in the specification, with all the query specific parameters (excluding MAXREC) being an Iterable to allow querying on multiple values for a given parameter.
There will be a mapping task on the FastAPI application, as the app has to take in the parameters as "Query" types and create an SIAv2Parameters instance to pass on to the dax_obscore query API.


## 8. Example Query

An example query taken from the IVOA spec to demonstrate usage:


<em>How do I query a SIAV2 service containing  IRAS-IRIS images in a circle of 0.1 deg around position 2.8425 +74.4846 selecting 200 and 60 micron bands ?</em>


```http://dalservices.ivoa.net/sia2/query?POS=CIRCLE 2.8425 74.4846 0.1 &BAND=0.0002&BAND=0.00006&COLLECTION=IRAS-IRIS```


```
<?xml version="1.0" encoding="UTF-8" ?>
<VOTABLE version="1.2" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://www.ivoa.net/xml/VOTable-1.2.xsd">
  <RESOURCE type="results">
    <INFO name="QUERY_STATUS" value="OK"/>
    <TABLE>
      <FIELD name="dataproduct_type" ucd="meta.id" datatype="char" utype="obscore:ObsDataSet.dataProductType" arraysize="*">
        <DESCRIPTION>Data product type</DESCRIPTION>
      </FIELD>
      <FIELD name="calib_level" ucd="meta.code;obs.calib" datatype="int" utype="obscore:ObsDataSet.calibLevel">
        <DESCRIPTION>Calibration level</DESCRIPTION>
      </FIELD>
      <FIELD name="obs_collection" datatype="char" ucd="meta.id" utype="obscore:DataID.Collection" arraysize="*">
        <DESCRIPTION>Data collection to which dataset belongs</DESCRIPTION>
      </FIELD>
      <FIELD name="obs_id" ucd="meta.id" datatype="char" utype="obscore:DataID.observationID" arraysize="*">
        <DESCRIPTION>Free syntax Observation Identifier</DESCRIPTION>
      </FIELD>
      <FIELD name="obs_publisher_did" ucd="meta.ref.url;meta.curation" datatype="char" utype="obscore:Curation.PublisherDID" arraysize="*">
        <DESCRIPTION>Publisher's ID for the dataset ID</DESCRIPTION>
      </FIELD>
      <FIELD name="access_url" ucd="meta.ref.url" datatype="char" utype="obscore:Access.Reference" arraysize="*">
        <DESCRIPTION>URL used to access dataset</DESCRIPTION>
      </FIELD>
      <FIELD name="access_format" datatype="char" ucd="meta.code.mime" utype="obscore:Access.Format" arraysize="*">
        <DESCRIPTION>Content or MIME type of dataset</DESCRIPTION>
      </FIELD>
      <FIELD name="access_estsize" datatype="int" ucd="phys.size;meta.file" utype="obscore:Access.Size">
        <DESCRIPTION>Dataset estimated size</DESCRIPTION>
      </FIELD>
      <FIELD name="target_name" datatype="char" ucd="meta.id;src" utype="obscore:Target.Name" arraysize="*">
        <DESCRIPTION>Target name</DESCRIPTION>
      </FIELD>
      <FIELD name="s_ra" datatype="double" ucd="pos.eq.ra" utype="obscore:Char.SpatialAxis.Coverage.Location.Coord.Position2D.Value2.C1" unit="deg">
        <DESCRIPTION>Spatial Position RA</DESCRIPTION>
      </FIELD>
      <FIELD name="s_dec" datatype="double" ucd="pos.eq.dec" utype="obscore:Char.SpatialAxis.Coverage.Location.Coord.Position2D.Value2.C2" unit="deg">
        <DESCRIPTION>Spatial Position Dec</DESCRIPTION>
      </FIELD>
      <FIELD name="s_fov" datatype="char" ucd="phys.angSize;instr.fov" utype="obscore:SpatialAxis.Coverage.Bounds.Extent.diameter" unit="deg">
        <DESCRIPTION>Spatial Field of view "diameter"</DESCRIPTION>
      </FIELD>
      <FIELD name="s_region" datatype="char" ucd="phys.angArea;obs" utype="obscore:Char.SpatialAxis.Coverage.Support.Area" arraysize="*" unit="deg">
        <DESCRIPTION>Spatial support</DESCRIPTION>
      </FIELD>
      <FIELD name="s_resolution" datatype="double" ucd="pos.angResolution" utype="obscore:Char.SpatialAxis.Resolution.refval.value">
        <DESCRIPTION>Spatial resolution FWHM</DESCRIPTION>
      </FIELD>
      <FIELD name="t_min" datatype="double" ucd="time.start;obs.exposure" utype="obscore:Char.TimeAxis.Coverage.Bounds.Limits.StartTime" unit="s">
        <DESCRIPTION>Time coordinate Lower limit</DESCRIPTION>
      </FIELD>
      <FIELD name="t_max" datatype="double" ucd="time.end;obs.exposure" utype="obscore:Char.TimeAxis.Coverage.Bounds.Limits.StopTime" unit="s">
        <DESCRIPTION>Time coordinate Higher limit</DESCRIPTION>
      </FIELD>
      <FIELD name="t_exptime" ucd="time.duration;obs.exposure" datatype="double" utype="obscore:Char.TimeAxis.Coverage.Support.Extent" unit="s">
        <DESCRIPTION>Exposure time</DESCRIPTION>
      </FIELD>
      <FIELD name="t_resolution" datatype="double" ucd="time.resolution" utype="obscore:Char.TimeAxis.Resolution.refval.value" unit="s">
        <DESCRIPTION>Time resolution</DESCRIPTION>
      </FIELD>
      <FIELD name="em_min" datatype="double" ucd="em.wl;stat.min" utype="obscore:Char.SpectralAxis.Coverage.Bounds.Limits.LoLimit" unit="m">
        <DESCRIPTION>Spectral coordinate Lower limit</DESCRIPTION>
      </FIELD>
      <FIELD name="em_max" datatype="double" ucd="em.wl;stat.max" utype="obscore:Char.SpectralAxis.Coverage.Bounds.Limits.HiLimit" unit="m">
        <DESCRIPTION>Spectral coordinate Higher limit</DESCRIPTION>
      </FIELD>
      <FIELD name="em_res_power" datatype="double" ucd="spect.resolution" utype="obscore:Char.SpectralAxis.Coverage.Resolution.ResolPower.refval">
        <DESCRIPTION>SPECTRAL Resolving power</DESCRIPTION>
      </FIELD>
      <FIELD name="o_ucd" datatype="char" ucd="meta.ucd" utype="obscore:Char.ObservableAxis.ucd" arraysize="*">
        <DESCRIPTION>UCD specifying the quantity on Observable axis</DESCRIPTION>
      </FIELD>
      <FIELD name="pol_states" datatype="char" ucd="meta.code;phys.polarization" utype="obscore:Char.PolarizationAxis.stateList" arraysize="*">
        <DESCRIPTION>Enumeration of Polarization states</DESCRIPTION>
      </FIELD>
      <FIELD name="facilty_name" datatype="char" ucd="meta.id;instr.tel" utype="obscore:Provenance.ObsConfig.facility.name" arraysize="*">
        <DESCRIPTION>Facility name</DESCRIPTION>
      </FIELD>
      <FIELD name="instrument_name" ucd="meta.id;instr" datatype="char" arraysize="*" utype="obscore:Provenance.ObsConfig.instrument.name">
        <DESCRIPTION>Instrument name</DESCRIPTION>
      </FIELD>
      <DATA>
        <TABLEDATA>
          <TR>
            <TD>cube</TD>
            <TD>1</TD>
            <TD>IRAS-IRIS</TD>
            <TD>I422B2H0</TD>
            <TD>ivo://cds.u-strasbg.fr/IRAS-IRIS/25MU/I422B2H0</TD>
            <TD><![CDATA[http://aladix.u-strasbg.fr/cgi-bin/nph-Aladin++dev.cgi?out=image&position=0.000000+80.000000&field=I422B2H0&survey=IRAS-IRIS&color=25MU&mode=view]]></TD>
            <TD>image/fits</TD>
            <TD>1600</TD>
            <TD>I422B2H0</TD>
            <TD>0.000000</TD>
            <TD>80.000000</TD>
            <TD>0.5</TD>
            <TD>POLYGON 30.0 200.0 32.0 200.0 32.0 198.0 30.0 198.0</TD>
            <TD></TD>
            <TD></TD>
            <TD></TD>
            <TD>1000</TD>
            <TD>1.0</TD>
            <TD>0.21</TD>
            <TD>0.21</TD>
            <TD>5.0</TD>
            <TD></TD>
            <TD>Stokes</TD>
            <TD>IRAS-IRIS</TD>
            <TD></TD>
          </TR>
        </TABLEDATA>
      </DATA>
    </TABLE>
  </RESOURCE>
</VOTABLE>
```

## 9. References

IVOA
- SIAv2 IVOA recommendation: https://www.ivoa.net/documents/SIA/

Confluence
- SIAv2 over Butler - Confluence Design Page https://confluence.lsstcorp.org/display/DM/SIAv2+over+the+Butler

JIRA Issues:
- https://rubinobs.atlassian.net/browse/DM-45477
- https://rubinobs.atlassian.net/browse/DM-45906
- https://rubinobs.atlassian.net/browse/DM-45860

Other technotes for reference:
- https://dmtn-238.lsst.io/ {cite:p}`DMTN-238`
- https://dmtn-208.lsst.io/ {cite:p}`DMTN-208`

Github repos:
- https://github.com/lsst-dm/dax_obscore
- https://github.com/lsst-sqre/safir

```{bibliography}
  :style: lsst_aa
```
