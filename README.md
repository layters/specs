# neroshop DHT specification

Authors: [@larteyoh](https://github.com/larteyoh)

## Overview
The neroshop DHT is based on the Kademlia Distributed Hash Table (DHT) with some additional protocol message types.



## Definitions

### Replication factor (`r`)
Represents the number of nodes closest to a key, in which `put` messages are sent to.
The default value for `r` is currently 10 and it is usually the same value as the maximum closest nodes (`k`) or lower.

###  Maximum closest nodes (`k`)
Represents the number of nodes closest to a key or id, in which `get` and `find_node` messages are sent to. The default value for `k` is currently 20.

### Distance

In all cases, the distance between two keys/node ids is `XOR(sha3_256(key1),
sha3_256(key2))`.

### Routing table
...

### Hash table
The hash table is an `std::unordered_map` that stores a `std::string` key representing a SHA-3-256 hash and an `std::string` value containing the JSON value.



## Protocol messages

* `ping`: This query type is often used to check the liveliness of a node.

* `find_node`: This query type is used to request a list of nodes with an id closest to a specific id or key.

    - The response is a `nodes` message containing information about nodes such as their IP addresses and ports.


* `put`: This query type is used to store key-value data in a receiving node's hash table. 

    - The querying node sends a `put` message to the `r` nodes with an id closest to the key.
If any of the closest nodes fail to respond to the put request, they are replaced by other closest nodes within the routing table.

    - When the querying node makes a put request and it is the originator of the data, it also stores its own data in its hash table right after sending the `put` to the other nodes in its routing table.
On receiving a `put` request, a node will `store` the key-value pair data in its own hash table.

    - The kind of data stored in the hash table is typically related to user account, listings, orders, and ratings/reviews. 
Each data contains metadata which specifies the type of data that is being stored.


* `get`: This query type is used to find a value to a specific key. 

    - The querying node typically sends a `get` message to the `k` nodes with an id closest to the key and when any of those closests nodes receive the `get`, they respond with a message containing the value to the requested key. In the case that any of the closest nodes do not have the value in their hash table, they attempt to send `get` messages to the closest nodes in their routing table until the value is found or not.


* `map`: This query type takes the value then uses it to create search terms and map them to the corresponding key.
Unlike `put`, `map` stores data in the local database file rather than the in-memory hash table in order to make use of SQLite's FTS5 (Full-Text Search 5) feature and perform advanced and efficient search operations.

    - A "map" request is similar to Bittorent's "announce_peer" in a sense that you are announcing that you have the data that you want indexed (peers cannot send a map request containing data that they themselves do not possess).
    
    - The process of mapping search terms to DHT keys is known as inverted indexing or simply, indexing.


* `set`: Same as put, except it updates the value to a key without changing the key.

    - This query also checks and validates the signature beforehand, using the user's RSA or Monero keys to ensure that the data can only be modified by its originator.

    - Set messages can only be processed while in IPC mode so that outside nodes cannot modify other peers' data anyhow.


## Features
- [x] Boostraping mechanism(s)
    - Hardcoded bootstrap nodes
- [x] Keys/Node IDs
    - represented as SHA-3-256 hashes
- [x] KBuckets
    - represented as an unordered_map of <int, std::vector<std::unique_ptr<Node>>>. A max of 256 kbuckets is set, each of them containing up to 25 elements.
- [x] XOR Distance between Keys
- [x] Basic protocol message types: `ping`, `find_node`, `put`, and `get` as well as two additional message types specific to this implementation: `map` and `set`.
- [x] Periodic health checking
    - checks node liveliness every `NEROSHOP_DHT_PERIODIC_CHECK_INTERVAL` seconds
- [x] Periodic republishing (refresh)
    - republishes data to the network every `NEROSHOP_DHT_REPUBLISH_INTERVAL` hours
    - in addition to periodic republishing, node data is automatically republished upon termination of the GUI client
- [x] Distributed indexing
    - the moment a node joins the network, it receives indexing data from the initial `k` closest nodes it contacts via the `map` protocol message type.
- [x] Data validation
    - validates data correctness
- [x] Data integrity verification
    - verifies data integrity using either **monero** or RSA digital signatures
- [ ] node IP address blacklisting
- [ ] Keyword/search term blacklisting
- [x] Expiration dates on certain data such as `orders`

## Data serialization (examples)
**User:**
```json
{
    "avatar": {
        "name": "32c34b46588b53c4212557cf91d5bca391a83c8b45cd403227d4bf73666a49e0.jpg",
        "size": 72163
    },
    "created_at": "2023-08-01T15:00:23Z",
    "display_name": "dude",
    "metadata": "user",
    "monero_address": "5ARm5QSDW3uWT8EEeoQ5mHghcw82SqYrUHzS99CS1TaxY3XDMGCyeDUN59veZwYfLCNH6mvvzKwJYiLv7Bag1Swn7sSEF9e",
    "public_key": "-----BEGIN PUBLIC KEY-----\nMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAoGKa3cN7JNj6xKgvG49f\nkg4Izod2oqNqoraYHMLAhI29MWXUIc8FizhhJUlO4Ts3i2qFaq1ng5jM15K2SQUo\na1ve4rATAbL0j+AnFMPQ/i883m6UJRdO5rUdtjLiKELuXckaQ3pbOmJ06lXI3qQ+\nzrPtgHp7hK2zfJSI+lgqHfU5AoQThOo5+QhuVgu1eJrvTMphc+XSPQ/8/7aPAdsB\nWI3FFy/QRtKr7RsjrsC2rTzYkqUo+MBXocN6d/1qCEPOgLuKJZvsKHt0VtWEHcQ2\ndTIqn4MhCh7Vz7psS1rBlfFW/j+PHXsQS7IRtU8RIy9yAoCKRO80aBhUe1ktvLaP\n2zXnW87vrRABYCc9fy9e3ZzCbPbAE/avozF/qR+XusBEKuvNUmMKW7CBet4CsGXc\nCAbJh5HIwrp+lgMZlgGLTjQ5cfegqtYtXWlJz09BbaY1+VNKhhhd0X/CN7apmqpu\njq94C9Db3dQGqTowHYflnwB3G3iXEcllmwECgFAIy4AG1cx3p/xkyXKe7r8QKcZF\nywE/2IQg8xmqtNDyQvcrIA9y6nyARskYZuqio62eQTaLKiYPTELmg5XmXyyVarJw\nIVXeWjqFz7WtS3RnM7RxoN8cwnd4JPIFTusGkucLAd/TBYKytPrsV68dvUmyrm1F\no63Sv7T7kBpeGIn5VOFY0wUCAwEAAQ==\n-----END PUBLIC KEY-----\n",
    "signature": "SigV2PagGh1SMm7KVrahMXeb7177Mzox7RHr1ciS1juk5WLfWatZiEods4ougHZKzgujp1oYhtG2vDguYcH3ac5zMYtjY"
}
```
```json
{
    "avatar": {
        "name": "5d2c5842ba2341d5be8a579b2db50cf5db491d3ae8c8adbd9bcdaccf6403ffb8.png",
        "size": 3348
    },
    "created_at": "2023-08-01T15:40:23Z",
    "display_name": "lupita",
    "metadata": "user",
    "monero_address": "59PFsC9wHepdrz5oseoDW3auk8a1HtUZbDHmPWwbBFjsa3kcue9N4GbK2hMRoFGgyLGNv14Z2QjDGjT1TrsC1UerPK3igm4",
    "public_key": "-----BEGIN PUBLIC KEY-----\nMIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAwtXNtHoyaiMGbtB2R1m+\nVMI15xjzERCnUsqL2qrxCo/QCcCqtKjEhdknPQcMO5taAmaUJnovDfJcmIA5M0kQ\n4ZKjKtzREN/YSanvgAemkX75I5rQPXIMQAz4Cs5Ie+cheOrl1WYo8DoAzlNRi8z0\nT2MDNBisMaWhqGfwEUgNtXQjxZpcYs9GmiM4qgIdCSniA6ixuPAsLNCI/LXhEBxX\ng1tyYJW4nOm9Sb0vlDr/ZiQRPDz23TdvlF6C1OXotsdd2dhNsbSEtXE7wEUYPX6O\n2UXbBF5PtzrmTuB6zoC9U4qfXrpkFJ9uPxpjER2fa0RSdcRhmywh/WTgVOUjR8iT\nRqSEs+8ByyZ2qV6TUSzKo2dWwCQIO6NHcX6ITbSrUeGFddIWuUmWVRGtG9d62yZD\n5o2895Qd+0J51L7l7xUNcFHmjxSm35AiGsvHJ4u583pz4DUodB8vASODuz/85X3x\nFdmLHV0IpeKZvpaSmYIRUsizg6SzXHNTYHEmOIlYCCJ16Zz2WO9P3OkzNms0ScKq\nfnL+OSBC47zquWpI905bB625SVJJaUZJvlIg6/wU2fdfVteeuboEiMiPyted9aId\nS8Vq8Ks8yFTZJzvhyOqpzSLx6/v5OuA+ABQH+AHOax36QRB+i6Fe3Vnc/YJLjmTp\n/8+KD7atMZb0UOpGCy4INeUCAwEAAQ==\n-----END PUBLIC KEY-----\n",
    "signature": "SigV2gASDvXcQkeW5LLzt7mMxZu5NvBb3s9rawYzf9wDVoWqeUtxtKTtWhvLVM9pn3rJ6aBYvKNX47tuhv57BPRP6jTyL"
}
```

**Listing**:
```json
{
    "condition": "New",
    "currency": "USD",
    "date": "2023-07-20T06:52:01Z",
    "id": "44d7f6f9-9a94-4fd9-84ad-7dc22a182ada",
    "location": "Germany",
    "metadata": "listing",
    "price": 10.0,
    "product": {
        "category": "Sports & Outdoors",
        "description": "",
        "images": [
            {
                "id": 0,
                "name": "2d2480374904c515684793a6ce174da1452c0ce87aabebfb8a11a30ebf648d51.png",
                "size": 6768
            }
        ],
        "name": "Monero ball"
    },
    "quantity": 1,
    "seller_id": "5ARm5QSDW3uWT8EEeoQ5mHghcw82SqYrUHzS99CS1TaxY3XDMGCyeDUN59veZwYfLCNH6mvvzKwJYiLv7Bag1Swn7sSEF9e",
    "signature": "SigV2bo1rMpaK9xeWU6CEuSnSCJ7buT8qANS8gUsjRx4wVpqpdxwj7LbLitkjYfD1NJeYmvJndYM9bUe2qWGrn7W35ebz"
}
```
```json
{
    "condition": "New",
    "currency": "USD",
    "date": "2023-07-20T07:14:26Z",
    "id": "2cd40b7b-a142-425f-b17a-afe9e6064a78",
    "location": "Thailand",
    "metadata": "listing",
    "price": 5.0,
    "product": {
        "category": "Food & Beverages",
        "description": "",
        "images": [
            {
                "id": 0,
                "name": "6ccce4863b70f258d691f59609d31b4502e1ba5199942d3bc5d35d17a4ce771d.png",
                "size": 26874
            }
        ],
        "name": "Green Tea"
    },
    "quantity": 100,
    "seller_id": "59PFsC9wHepdrz5oseoDW3auk8a1HtUZbDHmPWwbBFjsa3kcue9N4GbK2hMRoFGgyLGNv14Z2QjDGjT1TrsC1UerPK3igm4",
    "signature": "SigV2ZqBnw1sass4BrpWeq5HPacdf1zgdfkikYEW356kJw4XrazHuRPrirYydSeGVcm2wcRTWWLu5g1oX2PfxogjnnYNQ"
}

```

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
    "content": "covNjIfeDiZQOzJwZ9AkLLVWk3xyzuTYSoe6yv7sdyS4ZrplsODEw6E9SV/0oAWyYcoTslImuSlzf7IqfROXOYE6J4nobyUQf+rMz8wf6YKk9yGM9gMIhruszsydfUmOWuCtFLGUe48K1sCl3KllXXixTEhspX0PUX3Y52R19gSDteACJSaIHUgpYAMiqUH0rBQIYZ2KiWEHGmgq08fKkRbNpHUfIsyaoXKNIai44F0fu2zRjbbVo3lBpUzYPDml7nI6ZzJYLONFAml9pCi3LVZt9u1CLya9x2vQkmuZShSo71gMDvBWNI0eWT28/rD8caZ0LzKcVY2v/QlACylbYRw+Y5givC20yQxVuVBBdUz8n2Nnghbv67hq7xO/skX5D/KPBs7WDEmU5acZ+EG1LA8WEAvp+r+Swopq0rBYZyZZIxrTC0NTZexQltqpbzju0VokwMiRj/17/4e0MP/XYFwpYPrxlHoOddtE17FM/7RSsjmK1Qw7YrElNHMKpXW3bSOsa4FglVCv9uC3kfwJvdGoxjI64bVQPHfJM7MG9sz42rG8HRC4rJ3h2tsZ2SYRGecTyhe6e1RLc7kjoqHvuv7Z0x8dGQ+/49KfDhmWaAcfMgH8B2rzWXL7fUh2zQKo+A8sv35cMK7YKUgvtSAO8SbgRQTeiI8xt2CadJXV7Jc=",
    "expiration_date": "2025-09-28T10:12:02Z",
    "metadata": "message",
    "recipient_id": "59MemvccRxiCHiKaZzQTNaY7AQvTJ7oMsPR6WVg6S1dUBcr53WXRUxtU7YsVcrdsXFEYmGi6DgoXcWUQf9hPwLspPYZksAS",
    "sender_id": "v1r9QM1vDMH2Me0SzSSC0oHQ6P7pcA632fxCZkHKga9glXoDh/NErzXbonuDc/YAim/aHZoNhN8apE3h4esMFHLeaTs988b7Bl1xCKQ3Kv5A1tFMAyKNgsuk1zQpq02kgZ5nVUgKGiDtsGWbA0kYWSVskFUH29KbDvTTIFl/m6NjmpsDwkCdP4qMj83KaBY1el8KXdlyYENymtp08bPHqUuRA4EDtTlCcx/AAtaf1a4ZebuLiCZ/UBvn6UkyC7fgGoDTcE7BC4o7hw5G79zsRzok2VqdLzU4jZWCczBwhCsaO+Z/ayUVonR8u1Vpo6Wfb0oIS4Lb9uwtRvtbdQYhiSPMBFTs7V3jFNwJMm7EldwH0R9VogTFRrMg838aZs2TINN5TOxWF8nLhK+0QteKHE9FBCOucuyTv1L0JA+nJTy+bXyAeOi3xw2G6WsPwWAUDKAuKeQlV2Znpfruf7Cw4HCFtofTKbgtC8bINJ/M4pjZ5Z082/O9xBjWMYH6vKWWWQTbf4EXxt3rGcnyc1SY454zP27lqHbbclHUWZs84DWdL30UTMorj9Bzolxr8gWuEIYf9/UmRkDDpVT5ERMiv94IfRQK2RFBphJNhXxsIpsYbog2x4Th3W2GAFOoVOAH15deR+l6KTX4MkLyrGRWvbQeMO2iRoTL+V9grchtvW8=",
    "timestamp": "2023-09-28T10:12:02Z"
}
```

## RPC messages
[`MessagePack`](https://msgpack.org/) is the binary serialization format used to transmit data throughout the neroshop network. JSON data is first converted to msgpack before it is transmitted. This is because msgpack reduces the JSON data size by 32%, making it smaller, faster and more efficient to transfer data across the network.

There are three types of messages that are sent in the neroshop network: `query`, `response`, and `error`. A query is basically a request made by a peer. The primary query types used are `ping`, `find_node`, `put`, `get`, `map`, and `set`.
When a query is sent, normally a response should be expected. Responses are typically a sign of the successful execution of a query. Sometimes the result of a query may be an error if a function could not be executed or no value is returned from a `get` request, or perhaps something else.

Each query consists a `query` field containing the query type, an `args` field containing the arguments for the specific query type. A response consists of any field/value representing the result of the response. All messages must have an `id` field containing the node's id and a `tid` to identify which message matches with which response. With the exception of IPC mode where the local GUI client sends queries to the local daemon server. A `version` field is also included in each message to specify which version of the neroshop DHT is currently being used by a peer.


## References

[BitTorrent](http://bittorrent.org/beps/bep_0005.html)

[anarkiocrypto](https://twitter.com/AnarkioC) - for helping me come up with the design for the data structure for users, listings, orders etc.
