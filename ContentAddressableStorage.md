A Content Addressable Storage (CAS) is a collection of service endpoints which provide read and creation access to immutable binary large objects (blobs). The core service is declared in the [Remote Execution API](https://github.com/bazelbuild/remote-apis), and also requires presentation of the [ByteStream API](https://github.com/googleapis/googleapis/blob/master/google/bytestream/bytestream.proto) with specializations for resource names and behaviors.

An entry in the CAS is a sequence of bytes whose computed digest via a hashing function constitutes its address. The address is specified as either a [Digest] message, or by the makeup of a resource name in ByteStream requests.

A CAS supports several methods of retrieving and inserting entries, as well as some utility methods for determining presence and iterating through linked hierarchies of directories as Merkle Trees.

Functionally within the REAPI, the CAS is the communication plane for Action inputs and outputs, and is used to retain other contents by existing clients, including bazel, like build event information.

# Actions

## Reads

A read of content from a CAS is a relatively simple procedure, whether accessed through BatchReadBlobs, or the ByteStream Read method. The semantics associated with these requests require the support of content availability translation (NOT_FOUND for missing), and seeking to a particular offset to start reading. Clients are expected to behave progressively, since no size limitation nor bandwidth availability is mandated, meaning that they should advance an offset along the length of the content until complete with successive requests, assuming DEADLINE_EXCEEDED or other transient errors occur during the download. `resource_name` for reads within ByteStream Read must be `"{instance_name}/blobs/{hash}/{size}"`

## Writes

Writes of content into a CAS require a prior computation of an address with a digest method of choice for all content. A write can be initiated with BatchUpdateBlobs or the ByteStream Write method. A ByteStream Write `resource_name` must begin with `{instance_name}/uploads/{uuid}/blobs/{hash}/{size}`, and may have any trailing filename after the size, separated by '/'. The trailing content is ignored. The `uuid` is a client generated identifier for a given write, and may be shared among many digests, but should be strictly client-local. Writes should respect a WriteResponse received at any time after initiating the request of the size of the blob, to indicate that no further WriteRequests are necessary. Writes which fail prior to the receipt of content should be progressive, checking for the offset to resume an upload via ByteStream QueryWriteStatus.

Buildfarm implements the CAS in a variety of ways, including an in-memory storage for the reference implementation, as a proxy for an external CAS, an HTTP/1 proxy based on the remote-cache implementation in bazel, and as a persistent on-disk storage for workers, supplementing an execution filesystem for actions as well as participating in a sparsely-sharded distributed store.

# Buildfarm Implementations

Since these implementations vary in complexity and storage semantics, a common interface was declared within Buildfarm to accommodate substitutions of a CAS, as well as standardize its use. The specifics of these CAS implementations are detailed here.

## Memory

The memory CAS implementation is extremely simple in design, constituting a maximum size with LRU eviction policy. Entry eviction is a registrable event for use as a storage for the delegated ActionCache, and Writes may be completed asynchronously by concurrent independent upload completion of an entry.

## GRPC

This is a CAS which completely mirrors a target CAS for all requests, useful as a proxy to be embedded in a full Instance declaration.

## HTTP/1

The HTTP/1 CAS proxy hosts a GRPC service definition for a configurable target HTTP/1 service that it communicates with using an identical implementation to the [bazel http remote cache protocol](https://github.com/bazelbuild/bazel/tree/master/src/main/java/com/google/devtools/build/lib/remote/http)

## Shard

A sharded CAS leverages multiple Worker CAS retention and proxies requests to hosts with isolated CAS shards. These shards register their participation and entry presentation on a ShardBackplane. The backplane maintains a mapping of addresses to the nodes which host them. The sharded CAS is an aggregated proxy for its members, performing each function with fallback as appropriate; FindMissingBlobs requests are cycled through the shards, reducing a list of missing entries, Writes select a target node at random, Reads attempt a request on each advertised shard for an entry with failover on NOT_FOUND or transient grpc error. Reads are optimistic, given that a blob would not be requested that was not expected to be found, the sharded CAS will failover on complete absence of a blob to a whole cluster search for an entry.

## Worker CAS

Working hand in hand with the Shard CAS implementation, the Worker CAS leverages a requisite on-disk store to provide a CAS from its CASFileCache. Since the worker maintains a large cache of inputs for use with actions, this CAS is routinely populated from downloads due to operation input fetches in addition to uploads from the Shard frontend.