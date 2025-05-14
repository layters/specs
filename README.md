# neroshop DHT specification

Authors: [@layter](https://github.com/layters)

## Overview
The neroshop DHT is based on the Kademlia DHT (Distributed Hash Table).



## Definitions

### Replication factor (`r`)
Represents the number of nodes closest to a key, in which `put` messages are sent to.
The default value for `r` is currently 3 and it is usually the same value as the maximum closest nodes (`k`) or lower.

###  Maximum closest nodes (`k`)
Represents the number of nodes closest to a key or node ID, in which `get`, `find_node`, and `get_providers` messages are sent to. The default value for `k` is currently 20.

### Distance

In all cases, the distance between two keys/node IDs is `XOR(sha3_256(key1),
sha3_256(key2))`.

`SHA-3-256` hexadecimal strings are first converted to an array (`std::array<uint8_t, 32>`) consisting of 32 bytes (`BUCKET_COUNT / 8`) which is then used to calculate the distance between two node IDs, using the XOR metric (`a ^ b`).

### Routing table
The routing table is a fixed size `std::array` consisting of **256** k-buckets (`std::vector<std::shared_ptr<Node>>`).

> _The number of k-buckets is equal to the number of bits in node IDs and keys. Since we use SHA-3-256, which produces 256-bits, the maximum k-buckets would be 256._

### Hash table
The hash table is an `std::unordered_map` that stores a `std::string` key representing a SHA-3-256 hash and an `std::string` value containing the JSON value.



## Protocol messages

* `ping`: This query type is often used to check the liveliness of a node.

* `find_node`: This query type is used to request a list of nodes with an id closest to a specific node ID or key.

    - The response is a `nodes` message containing information about nodes such as their IP or i2p addresses and ports.


* `put`: This query type is used to store key-value data in a receiving node's hash table. 

    - The querying node sends a `put` message to the `r` nodes with an ID closest to the key.
If any of the closest nodes fail to respond to the put request, they are replaced by other closest nodes within the routing table.

    - When the querying node makes a put request and it is the originator of the data, it also stores its client's own data in its hash table (both in-memory and on-disk) right after sending the `put` to the other nodes in its routing table.
On receiving a `put` request, a node will `store` the key-value pair data in its own hash table.

    - Each data contains a `metadata` field which specifies the type of data that is being stored. Metadata names include: `listing`, `user`, `message`, etc.

    - Data is validated (and verified using Monero digital signatures) before being inserted into a node's hash table.


* `get`: This query type is used to retrieve the value to a given key. 

    - The querying node typically sends a `get` message to the `k` nodes with an ID closest to the key and when any of those closests nodes receive the `get`, they respond with a message containing the value to the requested key. In the case that any of the closest nodes do not have the value in their hash table, they ~attempt to send `get` messages to the closest nodes in their routing table until the value is found or not~ respond with an error message indicating that the key was not found in their hash table (this is to speed up searches). 


* `get_providers`: This query type requests a list of peers that store the value for a given key.
    
    - This query is sent to the `k` closest nodes in the routing table. In addition to responding with a list of peers for the key, the closest nodes first check their own hash tables to see if they also hold the value for the given key. If the key is found then they add themselves to the list of peers.
    
    - The response contains a `values` field containing information about the providers' such as their IP or i2p addresses and ports.


* `map`: This query type takes the value then uses it to create search terms and map them to the corresponding key.
Unlike `put`, `map` stores data in the local database file rather than the in-memory hash table in order to make use of SQLite's FTS5 (Full-Text Search 5) feature and perform advanced and efficient search operations.

    - A "map" request is similar to Bittorent's "announce_peer" in a sense that you are announcing that you have the data that you want indexed (peers cannot send a map request containing data that they themselves do not possess).
    
    - On receiving a map request, a node would then add the querying node to its list of peers for the key specified in the map request.


## Features
- [x] Boostraping mechanism(s)
    - Hardcoded bootstrap nodes
- [x] Keys/Node IDs
    - represented as SHA-3-256 hashes
- [x] KBuckets
    - represented as an `std::array` of `std::vector<std::shared_ptr<Node>>`. A max of 256 kbuckets is set, each of them containing up to 20 elements.
- [x] XOR Distance between Keys
- [x] Basic protocol message types: `ping`, `find_node`, `put` (`store`), and `get` (`find_value`) as well as two additional message types specific to this implementation: `get_providers` and `map`.
- [x] Periodic node health checks
    - checks node liveliness every `NEROSHOP_DHT_NODE_HEALTH_CHECK_INTERVAL` seconds
- [x] Periodic data republishing
    - republishes data to the network every `NEROSHOP_DHT_DATA_REPUBLISH_INTERVAL` seconds
    - in addition to periodic republishing, node data is automatically republished upon termination of the GUI client
- [x] Periodic bucket refresh
    - node "refreshes" the routing table's kbuckets every `NEROSHOP_DHT_BUCKET_REFRESH_INTERVAL` seconds by sending a `find_node` query to the `k` closest nodes for even more nodes closest to its node ID.
- [x] Distributed indexing
    - the moment a node joins the network, it receives indexing data from the initial `k` closest nodes it contacts via the `map` protocol message type.
- [x] Data validation
    - validates data correctness
- [x] Data integrity verification
    - verifies data integrity using either **monero** or RSA digital signatures
- [ ] node i2p address blacklisting
- [x] Expiration dates on certain data

## Data serialization (examples)
**User:**
```json
{
    "avatar": {
        "name": "820498a0e45ce69840efda6d1c9f8ab27e43357ef43707d6e80fd818fc629799.jpg",
        "piece_size": 65536,
        "pieces": [
            "4901e8a761d3c7bb05ff31496553a37fc73d6b538712184a3740f5d6c112cf72",
            "bdf4070acec55b2b25d8a669d524d8fe46ad0d1e4056ff705d529448b286210e",
            "7b3db335757ae530061e8f24a437f43c3799a2517947d722622505d23942bbd2"
        ],
        "size": 189993
    },
    "created_at": "2024-07-08T02:42:00Z",
    "display_name": "dude",
    "metadata": "user",
    "monero_address": "52HpXJ54diSiHxjT91XdAUPsg8s1wrpDXNFkSCDpinN51Ph9N2c8AVue6bD9iKcRWwcrrk17Q72sQipMwtmECbTH43BMwyV",
    "public_key": "-----BEGIN PUBLIC KEY-----\nMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEApsartGobT3xOvgGHsJuY\nENx2BfGAsZeoZA+YaqbKPn4q+1gW1U6mKXvd1NUxum3kYPWy8PGqIcN3R3rw4roj\nv4F7qdRS3Tw5N5dL+lgQQXeBkNkUpZPLIJx+z7scc9PZHuLUZsVnMio/ZXuvo3ov\nGI7uWogqmziOEvkqNLB9PZ764PGFlb8jWNkayCR5QtCKjK68lk7hk7E5R8P7G1aJ\n5DBwZ2onR5DTN8As1NEa4B45+oqv+jI3O2JbKCFek01mAUnoGpqNAklJSQJPys2d\nVTdhCgEwDA2eB8J924UhJWPHyskzMFbSBa/K3oJjUZirEF2jygKZg27WwjeRqSmu\ngRIw6tEWXn9RgL6h0on/du/10rG+aCPKx2U9tiaciGcrPoDMoIAihxqXlj4tD/v7\noA97Z6ducK16CugAduY8PYrUjL3A9K7On0/LPS5PJZp68r9ZzLqHYyc80s9Wk5Nl\nR/c114srtjB33YMWrG4IItOXrz1dh/RcZEknzaMULYqOJz3WKYL7irVcMEwCEs7R\nML8/ic8Qhb/qD0l3JLrd5G9u3QOdv7Cp5G5LFeASH+ooJUNgesfBRUIKk6UL6VIU\nG0b8s+Ehefzml33JoO7z6T+O0UCw8Dq3+0sdj51ZtEe8A1mdJQiiFeTuUUU8D1Sv\n0Bz2umO0Fmc5RIifOh9tdnMCAwEAAQ==\n-----END PUBLIC KEY-----\n",
    "signature": "SigV2abdQosYGCTJcTLAVAmAH1bQQjsRtXV1ePCiWqktDtwPQVf7MJJNKiJCak1mBDqsfWPFkAQp3qrvCiEmtM1S9HcRD"
}
```

**Listing**:
```json
{
    "condition": "New",
    "currency": "USD",
    "custom_rates": {
        "XMR": 170
    },
    "date": "2024-08-14T02:02:42Z",
    "delivery_options": [
        "Digital"
    ],
    "id": "644a8a54-a3a1-4b00-87fb-bab7384a7415",
    "metadata": "listing",
    "payment_coins": [
        "XMR"
    ],
    "payment_methods": [
        "Crypto"
    ],
    "payment_options": [
        "Finalize"
    ],
    "price": 2,
    "product": {
        "category": "Miscellaneous",
        "description": "",
        "images": [
            {
                "id": 0,
                "name": "647c8228b70407c030ea50edfa1c50fe77a9865fe2b454eb827376a201dd245e.jpg",
                "piece_size": 131072,
                "pieces": [
                    "6df0a92043c17c8e2c7606123f0a020f68dde9793dd836582e63bbcc5f20bb46",
                    "9bd66158ebfce5957096b199aba70922843c955d8c0f612f600d05683c23030b",
                    "586e000bb24cfdfa5e95762528d5ca82ad585125e6755099bb6c519cf094e8bd"
                ],
                "size": 315793
            }
        ],
        "name": "some product"
    },
    "quantity": 1,
    "seller_id": "57yxBQXi7ztiooZKPqWTYy6HT9okm6QT26xtT14vTexgHFrKnxGRJV7LGaFwwFBAjkJ6NAdZBYDe1bxmQqR566cZCUYvaAb",
    "signature": "SigV2M4j2ZEfatz9j4L9LYe6gYnEzAbCaxynH1Xpp8vnitg1XN2M6edjbnqkFtAq5Wvk9rf4ifCjT26xzFjEgvUNJUEoe"
}
```
_Listings will be self-hosted and never published to the network so that they are easy to modify as decentralized data is hard to keep consistent across multiple nodes and also this helps a bit to prevent spam since whenever a node goes offline, its listings will not be visible to the rest of the network until it comes back online._

**Product rating**:
```json
{
    "comments": "This product is aiight",
    "metadata": "product_rating",
    "product_id": "2b715653-da61-4ea0-8b5a-ad2754d78ba1",
    "rater_id": "59HiXx4PNJDd4sWVWhWtqb8m9pB7Huyc9HgybAwjFGkj27HozxkaLXNDhFH4wtzfYCe8GLzhA5YUHPUGWnjfRhvdL5HHE1C",
    "signature": "SigV2BExHYCAKkhoTbC3L4tx2EeHTaz2ZAtLfu7YgWcu1Ex3JXVk4X8XssfPE3TLBSQGRNLFENo4iAMa3gFV3u3bKoEvY",
    "stars": 3,
    "timestamp": "2023-07-30T08:05:38Z"
}
```
**Seller rating**:
```json
{
    "comments": "I love you!!!",
    "metadata": "seller_rating",
    "rater_id": "59JKGvtkpPQ7UocLgK5UV79GTEYJqDsJMgF9CiKmmms858VKvVvx1uX8DfXKYLWtoK9LVL38WQgf1JVn8yGWXZuJJzVwWGt",
    "score": 1,
    "seller_id": "5AncSFWauoN8bfA68uYpWJRM8fFxEqztuhXSGkeQn5Xd9yU6XqJPW7cZmtYETUAjTK1fCfYQX1CP3Dnmy5a8eUSM5n3C6aL",
    "signature": "SigV2idt52jHJfAic6ccT5yQwkJ3iP7cmNZ8Y1DYFBoQme9C9foRBzM4b2XeZwCQyCwvoDPM4xbWQ3FBP9NvVCBMe6gkr",
    "timestamp": "2023-07-29T04:10:38Z"
}   
```

**Order**:
```json
{
	"coin": "Monero",
	"created_at": "2023-07-27T05:42:29Z",
	"customer_id": "52P6EwT4zhpEfFhDLc9WSWGkSaEudfF7t597EnFRnQ8sYv4SqdS7BwDPddiJtxeQUHSP7SYAfgycaPhwMvWLc8p8M26eDkB",
	"delivery_option": "Delivery",
	"discount": 0.0,
	"id": "43f5ab3f-7288-4ad5-8295-adaada079101",
	"items": [{
		"listing_key": "2e6b41de8e53b7cd8c143d4632d13fe3aafcc7ab0ce6c94a7fac232ea0e4634c",
		"quantity": 1,
		"seller_id": "53TTZUaFGVB9koKDF1tpxc3APGUVEBMqURpr1aMFZwhJPP8oJksaspq6XgHVaoZijwDnDHRYuMDFvbzNZUYcH2tt8NzMqtd"
	}, {
		"listing_key": "2e8c94bb4b3448478ab1cc7db22ee95f8c607a595d90f3284efb39c033c15c55",
		"quantity": 1,
		"seller_id": "53TTZUaFGVB9koKDF1tpxc3APGUVEBMqURpr1aMFZwhJPP8oJksaspq6XgHVaoZijwDnDHRYuMDFvbzNZUYcH2tt8NzMqtd"
	}],
	"metadata": "order",
	"notes": "12 Robot Dr. Boston MA 02115",
	"payment_option": "Escrow",
	"shipping_cost": 0.0,
	"status": "New",
	"subtotal": 0.036584724958608955,
	"total": 0.036584724958608955
}
```

**Message**:
```json
{
    "content": "nYknoIq5JyLzNQ7Tv/8HBBBdAAWK6JhFk1i1rajccOl6+6lAXgxYHuHJdADwX3CGp/sMi/LsPi3687Oy3lK44fCE6+OMCz9bpkqQ5pX/ORwxi9Lcmdu+5PDG9TaVLEMzOedU//R6/TVnXpedGOTGD+BFbumXYUGVcaGtZTp0EHnQJr+UNE1cBTo7jF/sqCQBjQ5QmkGPDLvumCCEXyt6QQRq+Q+W7sDeH17HpAI8iVMOtgzI9Oz0BdSHLIVpqTpGIoD5BYrHOv389Q6iBea2CciGQdEgRc9wwg8JWp4+KGan7rwkBzjAdu8Y5Ab2yfRkpYWab4Qqn9Pj8DBMl+Qw7Cri/uNt1Tf4rBkef68fzrfdVHtGWMUc5oiml7o47sE9cEh6G0Ll+j34VGD/WGBjsQ03sDN7/IsIjg92DlsSexGJj+mmTAseRA+ZC5WM8hS8rO9b7Q6EAUjFXnZf6ybdqki5U1K2COsqHPyHGUHsSNCs+UQjIVDqEBTRv32EaGmmOgtEgSUeMc9FnLNCXiZvFOACib03Ngvvt1df74z9WS+zML2FPtTX5dPSw9qbS2p9bwW5T7lnx1xOTbpO5fvnObq1kqUejrIMcWR1Ixkmo/PZP/nIwkWGGmnJcTU5oVowzjf/XrEWbGRw+pDAvAb9uwYIoPvrnkzBASI58PRni5Q=",
    "expiration_date": "2024-06-13T01:41:27Z",
    "metadata": "message",
    "recipient_id": "5AzTEG7CMZAPZYnUfvToHjK9884crCSPxLiKkTGXJcdpSH5nu3KqceuG8urjoChFAw16HfZb3XygshG2CMzBphrJJygvp43",
    "sender_id": "ZR/+hAR2AZCUiDzkTle4DRa+MDsK1zNbOBBsipX5DNRp1OYMiKDMYGx73I02V79rV6AoHr2D2YmDVd1sh+HYRwsJgyMmyxnBTLggudF/hIwsThiX9pzMdF7vPFnr3GRHWvlqkqvRPe1/PTQ2e+HL1hv5umSrfekrUblWQ/stTfwJwkNoAxzBQYfhz9StIX8piZbAGQS/k81QXDP/mBmBHYVeBjnYXa/4XJMhOIA7qq6+mwweyuu6YtERFJqT9by9ubW2XNCV1vA94ErLOF90ur/gJsb3TAtR0iRK2YElevc80lLwwvyc/DABmLHNdaSypM1cdcJigzo7eoRm2ZniK3NM8DE3urETgi5N45io6qIG97I6Ng3GhL6oSpCdLAWq1t3+eBDU5kS9GOLIW2JJWrLlzFb0SWhQjNyNLm45K66ryb3uVgLE0yM2Ixftzh1X4hIVBWLQ8nNZhSVoSRsZtj/JEhKYGlVQZSh3W5dpaUTfxRroLAZlDFHT4Pi9mdhUNOKFw+yoDT9lJrJq91SqhCer712+0LsHoDnByeuj6+aMMqyxOaWSmi47yhElLzJ0hH2OrQ0thgYT+dx1GU8obj19tKik/t/7C5gHmAquaswnqp5s/K9ISqCZ0cDCciO/Iv1WWJG0C/GFcecpq3vZMvJhED+Q8xXmMEe/fyN0hVI=",
    "signature": "SigV2UrRgknKtdcwCdLPX4YnYtdF8Tf2F4oyPqVwCuQNgD1yeE5JFkecWmg9P8FjjofpLMTZ1DRJD8HPj4TGHcBptPYWW",
    "timestamp": "2024-05-14T01:41:27Z"
}
```
_Using a tool like [base64decode](https://www.base64decode.org/), you can decode the base64 encoded `content` and `sender_id` to confirm that these fields have been encrypted._


> Note: `user`, `seller_rating`, `product_rating`, and `message` datas are already feature complete while `listing` and `order` datas still need more work.


## RPC messages
[`MessagePack`](https://msgpack.org/) is the binary serialization format used to transmit data throughout the neroshop network. JSON data is first converted to msgpack before it is transmitted. This is because msgpack reduces the JSON data size by 32%, making it smaller, faster and more efficient to transfer data across the network.

There are three types of messages that are sent in the neroshop network: `query`, `response`, and `error`. A query is basically a request made by a peer. The primary query types used are `ping`, `find_node`, `put`, `get`, `get_providers` and `map`.
When a query is sent, normally a response should be expected. Responses are typically a sign of the successful execution of a query. Sometimes the result of a query may be an error if a function could not be executed or no value is returned from a `get` request, or perhaps something else.

Each query consists a `query` field containing the query type, an `args` field containing the arguments for the specific query type. A response consists of any field/value representing the result of the response. All messages must have an `id` field containing the node's ID and a `tid` (transaction ID) to identify which message matches with which response, with the exception of messages sent when in IPC mode where the local GUI client sends queries to the local daemon server. A `version` field is also included in each message to specify which version of the neroshop DHT is currently being used by a peer.


## References

[BitTorrent](http://bittorrent.org/beps/bep_0005.html)
