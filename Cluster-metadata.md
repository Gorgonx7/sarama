## Current implementation

The current implementation is ugly and not optimal, but works reasonably well in practice.

#### Initializing metadata

1. We initialize the cluster with a list of seed brokers. The order of this list is randomized so that every running client will contact a different broker initially. This list of brokers are stored in the `seedBrokers` slice of the client. They have no ID assigned to them.
2. Once we get metadata back, any broker that is included in the response will be added to the `brokers` map, indexed by ID.
3. All the operations that require a broker will look it up in the `brokers` map, based on ID.

#### Refreshing metadata with error handling

1. We contact the first broker on the seed broker list.
2. If it does not respond, we remove it from the `seedBrokers` slice, and add it to the `deadSeeds` slice.
3. If the `seedBrokers` list is empty, we grab a broker from the `brokers` map instead. If this one fails to respond as well, we remove it from the `brokers` map.
4. If the `brokers` map is empty, we wait for a bit so the cluster can recover, and resurrect all the `deadBrokers` by moving them back to the `seedBrokers` slice. This costs one retry attempt.
5. If we get a successful response, we add all brokers included in the response back to the `brokers` map.
5. If we get a response, but it is a failure, we wait for a bit and try again. This costs a retry as well.
6. By default we retry 3 times, after which we return an error.

### Problems with this implementation

- The meaning of a retry is unclear. Sometimes it means "try all brokers", but sometimes we go through a retry after talking to only broker.
- We keep 3 sets of brokers: the `brokers` map, `seedBrokers`, and `deadSeeds`.
- The logic is hard to understand.

### Potential improvements

- Only use the `brokers` map to store brokers that have an ID. Only use the list of seed brokers in `NewClient` to initialize the `brokers` map, throw them away afterwards.
- Contact brokers that are already connected before connecting to brokers that are not used, to minimize connections.
- Contact brokers that haven't processed a request recently before contacting brokers that just handled one, to better load balance requests over the active brokers.
