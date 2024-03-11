# Recommendation engine

## Diagram

![diagram](./Recommender.drawio.svg "Architecture")

## Components

### User-facing API

- Implementation: REST API on GKE/GCE
- Purpose: Interface with the users; wrap the raw model serving API 
- Reason:
    - REST API is an industry standard for exposing a functionality to the end user
    - GKE/GCE are both good choices for hosting a REST app
- Pros (over directly exposing Model Serving API):
    - provides an additional security level
    - handles gathering additional data for each request so that model serving can be a "pure function"
    - can autoscale separately from model serving
- Cons (over directly exposing Model Serving API):
    - additional cost (should be negligible compared to model serving)
    - additional layer of complexity

### Model Serving API

- Implementation: [Vertex AI Endpoints](https://cloud.google.com/vertex-ai/docs/predictions/overview#model_deployment)
- Purpose: produce predictions
- Reason: Vertex AI is native to GCP, and have many capabilities we're interested in
- Pros:
    - native to GCP
    - supports many model types out-of-the-box
    - can autoscale
- Cons:
    - "vendor-locked" to GCP
    - costs for batch inference workloads can be suboptimal

### Model Registry

- Implementation: [Vertex AI Model Registry](https://cloud.google.com/vertex-ai/docs/model-registry/introduction)
- Purpose: track model versions for easier rollback and general bookkeeping/analysis
- Reason: provides a streamlined interface to manage models, experiment tracking
- Pros:
    - native to GCP
    - eliminates the need to track models by hand/implement our own model registry
- Cons:
    - "vendor-locked" to GCP

### Model Artifact Storage

- Implementation: GCS
- Purpose: store model artifacts
- Reason: GCS is durable, scalable and usually cost-efficient.
- Pros:
    - simple
    - unused models can still be kept in a low-cost storage tier
- Cons:
    - data access speed (need to test)

### Model re-training pipeline

- Implementation: [Vertex AI pipelines](https://cloud.google.com/vertex-ai/docs/pipelines/introduction)
- Purpose: orchestrate model re-training process
- Reason: Vertex AI pipelines offer an orchestration framework for automated training
- Pros:
    - native to GCP
- Cons:
    - "vendor-locked" to GCP
    - less mature than many other orchestration solutions (e.g. Airflow)

### Model Monitoring

- Implementation: [Cloud Monitoring](https://cloud.google.com/monitoring/docs)
- Purpose: monitor system/model performance
- Reason: Cloud monitoring has many integrated metrics it can track, and also supports custom ones.
- Pros:
    - simple
    - native to GCP
    - supports open protocols (e.g. Prometheus)
    - supports custom metrics
- Cons:
    - idk

### Session data storage

- Implementation [Memorystore](https://cloud.google.com/memorystore/docs/redis)
- Purpose: store user session data so it can be accessed in a performant way
- Reason: we need this data to be readily available to produce tailored recommendations
- Pros:
    - really fast
    - managed
    - redis is open-source
- Cons:
    - cost? (need to estimate)

### Prediction log storage

- Implementation: GCS
- Purpose: store request-response logs from our serving layer for further analysis or retraining
- Reason: GCS can store large amounts of data efficiently
- Pros:
    - simple
    - unused data can still be kept in a low-cost storage tier
- Cons:
    - idk

## Overall solution

### Pros
- scalable
    - both serving layers and model re-training can be autoscaled
    - session storage is suitable for high load
- extendable, all of the following can be added easily:
    - new model types
    - new monitoring metrics
    - new features
    - new re-training triggers
- native to GCP
- uses modern tech stack

### Cons
- vendor-locked to GCP
- not suited for batch inference (this might not be a concern)

### Cost

It may be possible to implement a more cost-efficient solution tailored to the specific use-case (e.g. batch prediction), but this will increase complexity.

### Points of failure/further investigation:
- automatic retraining triggers should be engineered with care
- auto-scaling on both serving layers should be set up carefully to be able to accomodate spikes (e.g. Black Friday) and other usage patterns (e.g. scale down quickly in times of lower traffic)
- the user-facing REST api should be behind a load balancer
- might be a good idea to cache requests
