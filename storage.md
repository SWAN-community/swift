![Shared Web InFormaTion](images/swift.128.pxls.100.dpi.png)

# Shared Web InFormaTion (SWIFT) - Storage Manager

## Abstract

SWIFT requires a storage solution to persist node details between application 
recycling and between scaling the number of instances up or down. And also support the addition of new nodes using a api endpoint or a command line application.

For the majority of implementations, local storage is recommended. This reduces 
operational complexity and reliance on API calls to storage solutions external 
to the application.

However, storage interfaces for AWS DynamoDB, Azure Table Storage and Google 
Firebase are provided in the 
[go reference implementation](https://github.com/SWAN-community/swift-go).

SWIFT is not prescriptive when choosing a storage implementation as organizations
will have different operational requirements.

## Node Sharing

SWIFT node sharing allows operators to connect networks together by sharing details about nodes. This includes the node domain, start and end date and the node's shared secrets. As no central list of nodes needs to be maintained, this allows multiple networks to operate in a decentralized manner.

There is a node whose sole role is to share node details. It responds to requests to the well known sharing endpoint:

`/swift/api/v1/share`

The node will respond with a JSON document containing all known good nodes. The 
response is encrypted with the requested sharing node's shared secret. If the calling 
node knows the shared secret then it can decrypt the response and add the node 
details to it's own internal storage.

## Storage Manager

The storage manager is an interface which allows a SWIFT node to maintain a reference to multiple storage solutions and provide a simple API to query those stores.

On initialization, the storage manager will check each store it has been configured with.If, in any of the stores, it finds nodes with the sharing role, it will poll these nodes to retrieve the known good nodes that the sharing node is aware of.

For each sharing node polled, a new volatile in-memory store instance is created to record the node details. These additional stores are checked in-turn for sharing nodes and the process repeats until all known sharing nodes have been polled or until the max configurable number of stores has been reached.

After init, storage manager maintains a map of nodes keyed on domain that is readonly. The state of the storage manager does not change for the lifetime of the instance. 

While the storage manager interface contains methods to set or update nodes, these nodes will not be available in read operations until a new instance of storage manager has been initialized.

### Storage manager Interface

Methods provided by a storage manager implementation. All methods are private unless specified otherwise.

#### getNode

getNode returns the node instance associated with the domain. 

|Parameter|Type|Description|
|-|-|-|
|`domain`|`string`|**Required.** The node domain|

Returns: `node`

#### getNodes

getNodes returns all the node instances associated with a network.

|Parameter|Type|Description|
|-|-|-|
|`network`|`string`|**Required**. The node network|

Returns: `[]node`

#### getAllActiveNodes

getAllActiveNodes returns all the nodes for all networks which have the alive
flag set to true and have a start date that is before the current time.

Returns: `[]node`

#### getAllNodes

getAllNodes returns all the nodes for all networks.

Returns: `[]node`

#### getStores

Get an array of all the store instances.

Returns: `[]store`

#### GetAccessNode

GetAccessNode is a public method which returns an access node for the requested network, or null if there is no access node available. This would be used by an operator to get a SWIFT access node for an operation.

|Parameter|Type|Description|
|-|-|-|
|`network`|`string`|The network to get an access node from|

Returns: `node`

#### setNodes

setNodes adds or updates a node in the specified store. setNodes will also
succeed if no store name is provided and only one writeable store exists
in the storage manager.

|Parameter|Type|Description|
|-|-|-|
|`store`|`string`|The name of the store instance|
|`nodes`|`[]node`|**Required.** The node instances to add to the store|

## Storage Service

The SWIFT service is designed to handle high volumes of traffic. To avoid contention the storage manager that maintains a list of nodes is read-only once initialized. When setting new nodes, these new nodes will only become available when the storage manager is re-created. 

The storage service maintains a reference to a storage manager and continually refreshes the storage manager in the background.

On initialization, the store instances passed to the storage service are persisted. These are then used to create a new storage manager.

The storage service also maintains a background thread which will periodically refresh the active storage manager. A new storage manager is built using the same store instances used at start up. These store instances are used each time there is a refresh.

To avoid contention and the refresh process updates the reference to the storage manager in-place:
 * A new instance of storage manager is created. 
 * When the storage manager has initialized, the reference to the storage manager is locked and then updated to the new instance.
 * The old dereferenced storage manager can then be cleaned up or left for garbage collection.

On each refresh, all the known sharing nodes will be polled to recreate the storage manager. 

The refresh interval is a configurable number of minutes. Around 30 minutes strikes a good balance between reducing start up traffic and being frequent enough to handle changes to the network.

### Storage Service Interface

Methods provided by a storage service implementation. All methods are private unless specified otherwise.

#### getNode

getNode returns the node instance associated with the domain. 

|Parameter|Type|Description|
|-|-|-|
|`domain`|`string`|**Required.** The node domain|

Returns: `node`

#### getNodes

getNodes returns all the node instances associated with a network.

|Parameter|Type|Description|
|-|-|-|
|`network`|`string`|**Required**. The node network|

Returns: `[]node`


#### getAllNodes

getAllNodes returns all the nodes for all networks.

Returns: `[]node`

#### GetStoreNames

GetStoreNames returns an array of names of all the writeable stores.

Returns: `[]string`

#### SetNode

SetNode is a public method which takes a `register` instance and creates a new node. It returns a boolean value for if the set operations was successful or not and another boolean for if this is an update operation (if a node with the same domain already exists).

|Parameter|Type|Description|
|-|-|-|
|`register`|`Register`|Contains data used to register the node|

#### setNodes

setNodes adds or updates a node in the specified store. setNodes will also
succeed if no store name is provided and only one writeable store exists
in the storage manager.

|Parameter|Type|Description|
|-|-|-|
|`store`|`string`|The name of the store instance|
|`nodes`|`[]node`|**Required.** The node instances to add to the store|

## Alive Service 

In SWIFT operations, nodes will poll and redirect to one another very frequently. To maintain a good level of service and avoid situations where a bad node causes an operation to fail, nodes should be aware of which other nodes are operational.

Only nodes that have a start date that has elapsed and are flagged as alive can be selected for an operation. Each time a node is successfully accessed, the time of the operation is recorded against the node and the alive flag is set to true. This is what is called a passive alive verification.

SWIFT nodes must periodically poll each other to verify they are alive when a passive alive verification has not been performed for a configurable number of seconds. This is known as an active alive verification.

The process of an active alive verification is as follows:
1. Create a nonce value and encrypt it with the shared secret of the polled node.
2. Pass the encrypted data to the polled node;
3. The polled node decrypts the data and;
4. passes the decrypted data back as the response.
5. The polling node confirms that the response matches the original nonce value.
6. Update the node's last accessed time and set alive flag as indicated by the response.

If there is a response with a success status code and the value returned by the polled node matches the nonce value, then the alive flag is set to true and the last accessed time value is updated to the current time. Otherwise, if these conditions are not met then the alive flag is set to false. 

If the node is considered to be not alive then the alive service will check the node again after the polling interval has elapsed. Until the node alive flag is set to true then it will not be selected for use in SWIFT operations.
