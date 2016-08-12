
**NOTE:** I'm not sure if distribution is the best name for this concept or it should be called with other names like "image reference". So I'd like to hear your thoughts.

A distribution represents the method to fetch an image starting from an input string. It doesn't mandates the image types. Usually a distribution can only provide one image type but some can provide different image types (for example, see below, the docker distribution can provide a native docker image or an ACI generated by docker2aci). 

Before this concept was introduced, rkt used the ImageType concept, the ImageType was mapped to the input image string format and internally covered multiple concepts like distribution, transport and image type (hidden since all are now appc ACIs).

The distribution concept is used as the primary information in the different layers of rkt (fetching but also for references in a CAS/ref store).

## Distribution types

The distribution can be logically split in two types:

* Name based distribution
* Location based distribution

Name based distribution can be considered the first class distribution type as it'll be the primarly used distribution type.
Location based distribution can be thought as a sub-distribution for Name based distribution or a fast alternative (but not covering the Name based distribution feature) to directly fetch an image from a location.

A distribution is represented as an URI with the uri scheme as "cimd" and the opaque data and query parts as the distribution information (I started trying to use just URLs being an URI subset for location but distributions IMHO don't cleanly map to a resource locator but better to a name, [rfc3986](https://tools.ietf.org/html/rfc3986)) I'm calling them URIs instead of URNs because it's the suggested name from the rfc (and URNs are defined as having an urn scheme by rfc2141).

Every distribution starts with a common part: `cimd:DISTTYPE:V=uint32(VERSION):` where `cimd` is the container image distribution scheme, DISTTYPE is the distribution type, V=uint32(VERSION) is the distribution type format version.

### Current rkt distributions

In rkt there are currently three kind of distribution: Appc, Archive and Docker.

**Appc**
This is a Name based distribution

Appc defines a distribution using appc image discovery
The format is: `cimd:appc:v=0:name?label01=....&label02=....`
The scheme is "appc"
the labels values must be Query escaped
Example: `cimd:appc:v=0:coreos.com/etcd?version=v3.0.3&os=linux&arch=amd64`

**ACIArchive**
This is a Location based distribution

ACIArchive defines a distribution using an archive file

The format is: `cimd:aci-archive:v=0:ArchiveURL?query...`
The distribution type is "aci-archive"
FORMAT describes the archive format (for example "aci", "ociimagelayout")
ArchiveURL must be query escaped

Examples:
`cimd:aci-archive:v=0:file%3A%2F%2Fabsolute%2Fpath%2Fto%2Ffile`
`cimd:aci-archive:v=0:https%3A%2F%2Fexample.com%2Fapp.aci`
`cimd:aci-archive:v=0:file%3A%2F%2Fabsolute%2Fpath%2Fto%2Ffile?ref=refname`

**Docker**
This is a Name based distribution

Docker defines a distribution using a docker registry
The format is the same as the docker image string format (man docker-pull) with the "docker" scheme:
`cimd:docker:v=0:[REGISTRY_HOST[:REGISTRY_PORT]/]NAME[:TAG|@DIGEST]`
Examples:
`cimd:docker:v=0:busybox`
`cimd:docker:v=0:busybox:latest`
`cimd:docker:v=0:registry-1.docker.io/library/busybox@sha256:a59906e33509d14c036c8678d687bd4eec81ed7c4b8ce907b888c607f6a1e0e6`

### Future distributions

**OCI Image distributions**
This is a Name based distribution

Now a docker distribution can also provide oci image but in future oci image will define one or more own kind of distribution starting from an image name (and additional tags, labels)

**OCI Image layout**
This is a Location based distribution

It can fetch an image starting from a [OCI image layout](https://github.com/opencontainers/image-spec/blob/master/image-layout.md) format. The location can point to a single file archive, to a local/remote directory based layout or other kind of locations

Probably it will be the sub-distribution used by the OCI image distribution described aboce) (like ACIArchive is the sub-distribution for the appc distribution).

Since the OCI image layout can provide multiple images choosable by a ref there's the need to specify which ref to use in the archive distribution URI (see the above rev query parameter). Since distribution just covers one image it's not possible to import all the refs with just a distribution URI.

**Note** considering the OCI image spec README, probably the final distribution format will be similar to the app distribution. So there's a need to distinguish their User Friendly string (prepending an appc: or oci: ?).

To sum it up:

| Distribution   | Type     | URI Format                                                                | Sub-distribution |
|----------------|----------|---------------------------------------------------------------------------|------------------|
| Appc           | Name     | `cimd:appc:v=0:name?label01=....&label02=...`                             | ACIArchive       |
| Docker         | Name     | `cimd:docker:v=0:[REGISTRY_HOST[:REGISTRY_PORT]/]NAME[:TAG&#124;@DIGEST]` | <none>           |
| ACIArchive     | Location | `cimd:aci-archive:v=0:ArchiveURL?query...`                                |                  |
| OCI            | Name     | `cimd:oci:v=0:TODO`                                                       | OCIImageLayout   |
| OCIImageLayout | Location | `cimd:oci-image-layout:v=0:URL?ref=...`                                   |                  |

## User friendly distribution strings
Since the distribution URI can be complex there's a need to help the user to request an image via some user friendly string. rkt already has these kind of available input image strings (now mapped to an AppImageType):

* appc discovery string: example.com/app01:v1.0.0,label01=value01,... or example.com/app01,version=v1.0.0,label01=value01,... etc...
* file path: absolute /full/path/to/file or relative
The above two also may overlap so some heuristic is currently needed to distinguish them.
* file URL: file:///full/path/to/file
* http(s) URL: http(s)://host:port/path
* docker URL: this is a strange URL since it the schemeful (docker://) version of the docker image string

So, also the maintain backward compatibility these image string will be converted to a distribution URI:

| Current ImageType                     | Distribution URI                                                          |
|---------------------------------------|---------------------------------------------------------------------------|
| appc string                           | `cimd:appc:v=0:name?label01=....&label02=...`                             |
| file path                             | `cimd:aciarchive:v=0:ArchiveURL`                                          |
| file URL                              | `cimd:aciarchive:v=0:ArchiveURL`                                          |
| https URL                             | `cimd:aciarchive:v=0:ArchiveURL`                                          |
| docker URI/URL (docker: and docker:// | `cimd:docker:v=0:[REGISTRY_HOST[:REGISTRY_PORT]/]NAME[:TAG&#124;@DIGEST]` |

The above table also adds docker URI (docker:) as a user friendly string and its clearer than the URL version (docker://)

The parsing and generation of user friendly string is done outside the distribution package.

rkt uses two function to:
* parse a user friendly string to a distribution URI.
* generate a user friendly string from a distribution URI. This is useful for example when showing the refs from a refs store (so they'll be easily understandable and if copy/pasted they'll continue to work).

A user can provide as an input image both the "user friendly" strings or directly a distribution URI.

## Comparing distributions URIs
A Distribution will also provide:

* a function to compare if two Distribution URIs are the same (for example ordering the query parameters)

## Fetching logic with distribution

Distribution will be the base for a future refactor of the fetchers logic:

* Every distribution will have one or more dedicated fetchers:

* archive distribution -> base archive fetcher (may choose a sub fetcher based on the archive format) -> transport logic to fetch the ArchiveURL
* docker distribution -> docker fetcher ->  The docker fetcher may choose to use docker2aci and import an ACI in the store or natively handle v2/oci blobs and import them in the store.
* appc distribution -> appc fetcher -> The appc fetcher will call appc discovery and fetch the discovered URLs. An interesting thing is that this fetcher should internally convert the discovered ACI URL to a child distribution of type Archive without rewriting all the fetching logic

This will also creates a better separation between the distribution and the transport layers.

For example there may exist multiple transport plugins (file, http, s3, bittorrent etc...) to be called by an archive distribution.
