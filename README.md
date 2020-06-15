# thanos_prom_POC
thanos_prom_POC

3.2 Thanos Sidecar
The main component that runs along Prometheus
Reads and archives data on the object store
Manages Prometheus’s configuration and lifecycle
Injects external labels into the Prometheus configuration to distinguish each Prometheus instance
Can run queries on Prometheus servers’ PromQL interfaces 
Listens in on Thanos gRPC protocol and translates queries between gRPC and REST


3.3 Thanos Store
Implements the Store API on top of historical data in an object storage bucket 
Acts primarily as an API gateway and therefore does not need significant amounts of local disk space
Joins a Thanos cluster on startup and advertises the data it can access 
Keeps a small amount of information about all remote blocks on a local disk in sync with the bucket
This data is generally safe to delete across restarts at the cost of increased startup times


3.4 Thanos Query 
Listens in on HTTP and translates queries to Thanos gRPC format
Aggregates the query result from different sources, and can read data from Sidecar and Store
In HA setup, Thanos Query even deduplicates the result


A note on run-time duplication of HA groups: Prometheus is stateful and does not allow for replication of its database. Therefore, it is not easy to increase high availability by running multiple Prometheus replicas. 

Simple load balancing will also not work -- say your app crashes. The replica might be up, but querying it will result in a small time gap for the period during which it was down. This isn’t fixed by having a second replica because it could be down at any moment, for example, during a rolling restart. These instances show how load balancing can fail. 

Thanos Query pulls the data from both replicas and deduplicates those signals, filling the gaps, if any, to the Querier consumer.

‍

3.5 Thanos Compact 
Applies the compaction procedure of the Prometheus 2.0 storage engine to block data in object storage
Generally not concurrent with safe semantics and must be deployed as a singleton against a bucket
Responsible for downsampling data: 5 minute downsampling after 40 hours and 1 hour downsampling after 10 days
‍

3.6 Thanos Ruler
Thanos Ruler basically does the same thing as the querier but for Prometheus’ rules. The only difference is that it can communicate with Thanos components.

‍

4. Thanos Implementation
Prerequisites: In order to completely understand this tutorial, the following are needed:

1. Working knowledge of Kubernetes and kubectl

2. A running Kubernetes cluster with at least 3 nodes (We will use a GKE)

3. Implementing Ingress Controller and Ingress objects (We will use Nginx Ingress Controller); although this is not mandatory, it is highly recommended in order to reduce external endpoints.

4. Creating credentials to be used by Thanos components to access object store (in this case, GCS bucket) 

          a. Create 2 GCS buckets and name them as prometheus-long-term and thanos-ruler

          b. Create a service account with the role as Storage Object Admin

          c. Download the key file as json credentials and name it thanos-gcs-credentials.json

          d. Create a Kubernetes secret using the credentials, as you can see in the following snippet:
