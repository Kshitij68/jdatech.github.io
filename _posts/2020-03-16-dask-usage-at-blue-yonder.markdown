  ---
  layout: single
  title:  "Dask Usage at Blue Yonder"
  date:   2019-10-25 10:00:00 +0100
  tags: technology python data-engineering
  header:
    overlay_image: assets/images/tech_gear_banner.jpg
    overlay_filter: 0.2
    show_overlay_excerpt: false
  author: Andreas Merkel
  author_profile: true
  ---
  # Dask Usage at Blue Yonder

  Back in 2017, Blue Yonder started to look into 
  [Dask/Distributed](https://distributed.dask.org) and how we can leverage it to power
  our machine learning and data pipelines.
  In 2019, we started to use it heavily in production for providing demand predicions,
  recommended order quantities, and price proposals to our customers.
  Time to take a a look at the way we are using Dask!

  ## Use cases

  First, let's have a look at the use cases we have for Dask. The use of Dask at Blue Yonder
  is strongly coupled to the concept of [Datasets](https://tech.jda.com/introducing-kartothek/),
  tabular data stored in blob storage. We use these datasets for storing everything that is needed
  for generating predictions, recommended order quantities, and recommended prices for our customers,
  as well as the predictions, order quantities, and prices themselves.

  ### Downloading data from a database to a dataset

  One typical use case that we have is downloading large amounts of data from a database into a
  dataset. We use Dask/Distributed here as a means to parallelize the download and the graph is
  an extremely simple: a lot of parallel nodes downloading pieces of data into a blob and one
  reduction node writing the dataset metadata (which is basically a dictionary what data can be found in which blob).

  ### Doing computations on a dataset

  A lot of our algorithms for machine learning, forecasting, and optimization work on datasets.
  A source dataset is read, computations are performed on the data, and the result is written to
  a target dataset.
  In many cases, the layout of the source dataset (the partitioning, i.e., what data resides in which blob)
  is used for parallelization. This means the algorithms work independently on the individual
  blobs of the source dataset. Therefore, our respective Dask graphs again look very simple, with parallel nodes each performing
  the sequential operations of reading in the data from a source dataset blob, doing some computation on it,
  and writing it out to a target dataset blob. Again, there is a final reduction node writing the target
  dataset metadata.

  ### Dataset maintenance

  We also use Dask/Distributed for performing maintenance operations like garbage collection on datasets.
  Garbage collection is necessary since we use a lazy approach when updating datasets and only delete or update
  references to blobs in the dataset metadata, deferring the deletion of no longer used blobs.
  In this case, Dask is used to parallelize the delete requests to the storage provider (Azure blob service).
  In a similar fashion, we use Dask to parallelize copy operations for backing up datasets.

  ### Repartitioning a dataset

  Our algorithms rely on source datasets that are partitioned in a particular way. It is not always
  the case that we have the data available in a suitable partitioning, for instance, if the data results
  from a computation that used a different partitioning. In this case, we have to repartition the data.
  This works by reading the dataset as a Dask distributed dataframe (ddf), repartitioning the ddf using
  network shuffle, and writing it out again to a dataset.

  ## Dask cluster setup at Blue Yonder

  We run a somewhat unique setup of Dask clusters that is partly driven by the specific requirements
  of our domain and partly historically grown. We are running a large number of individual clusters
  that are homogeneous in size with many of the clusters dynamically scaled. The peculiarities of this
  setup have in some cases triggered edge cases and uncovered bugs, which lead us to submit a number
  of upstream pull requests.

  ### Cluster separation

  We are running separate Dask clusters for separate customers in order to isolate the customers from each other.
  For the same reason, we also run separate clusters for production, staging, and development environments. 
  But it does not stop there. For performance reasons, we install the Python packages holding the code needed
  for the computations on each worker. Since the different components of our compute pipeline are implemented
  and maintained as completely separate software packages whose requirements are not synchronized, this also
  means we have to run a separate Dask cluster for each of the components. This leads to us operating more than
  ten Dask clusters per customer and environment, with most of the time, only one of the clusters being active
  and computing something. While this leads to overhead in terms of administration and hardware resouces,
  it also gives us a lot of flexibility. For instance, we can update the software on the cluster of one part of the compute pipeline
  while another part of the pipeline is computing something on a different cluster.

  ### Some numbers

  The number and size of the workers varies from cluster to cluster depending on the degree of parallelism
  of the computation being performed, its resource requirements, and the available timeframe for the computation.
  At the time of writing this, we are running more than 500 distinct clusters.
  They have between one and 225 workers, with worker size varying between 1GiB and 64GiB of memory.

  ## Cluster scaling and resilience

  To improve manageability, resilience, and resource utilization, we run the Dask clusters on top
  of [Apache Mesos](http://mesos.apache.org/)/[Aurora](http://aurora.apache.org/). This
  means it is really easy to commission or decommission Dask clusters and to add or remove workers to/from
  an existing Dask cluster. At the moment, we are starting a migration from Mesos/Aurora towards 
  [Kubernetes](https://kubernetes.io/)

  Running on top of a system like Mesos or Kubernetes provides us with resilience since a failing worker 
  (for instance, as result of a failing hardware node)
  can simply be restarted on another node of the system. 

  Running 500 Dask clusters requires a lot of hardware. We have put two measures in place to improve
  the utilization of hardware resources: oversubscription and autoscaling.

  ### Oversubscription

  [Oversubscription](http://mesos.apache.org/documentation/latest/oversubscription/) is a feature of Mesos
  that allows to allocate more resources than physically present to services running on the system.
  This is based on the assumption that not all services exhaust all of their allocated resources at the same time.
  If the assumption is violated, classifying services into tiers allows to preempt less important services
  in favor of critical ones.
  We use this to re-purpose the resources allocated for production clusters but not utilized the whole time
  and use them for development and staging systems.

  ### Autoscaling

  Autoscaling is a mechanism we implemented to dynamically adapt the number of workers in a Dask cluster
  to the load on the cluster. For this, we added the ``desired_workers`` metric to Distributed, which exposes
  the degree of parallelism that a computation has and allows us to infer how much workers a cluster should
  ideally have. Based on this metric, as well as on the overall resources available and on fairness criteria
  (remember, we run a lot of Dask/Distributed clusters), we add or remove workers to our clusters.

  Restarting failed workers for resilience and adding/removing workers for autoscaling requires that the Distributed
  cluster can cope with workers being removed and/or new instances being added. While this is a supported use case, 
  it appeared to us that this seemed not a widely used feature. By making heavy use of it, we were able to uncover
  some bugs and to submit the corresponding fixes.

