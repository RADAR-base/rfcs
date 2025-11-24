---
RFC: 0002
Title: Just In Time Interventions Platform
Author(s): Heet Sankesara (@hsankesata)
Status: Draft
Created: <2025-11-24>
Updated:
Discussion:
---
Summary
-------

Now that we have a stable data collection platform and a few application using real time streaming (using Ksql and NiFi), the next step is to develop a platform that can run just in time adapative interventions (JITAI). This platform needs to be generic so that we could add and run multiple interventions application ranging from regular data checks to machine learning based prediction algorithms. In the RFC, I am proposing an initial JITAI architecture that can used to plug in various interventions and execute them.

Motivation
----------

Past work that used JITAI have been focused on specific use case and is not generalisable. In this work, we’ll focus on developing a platform that can be used for different use cases with minimal effort. It’ll be a novel addition to the platform which I do not believe exist in any other mHealth platform.

Non-Goals
---------

it doesn't cover the development of specific intervention applications, but rather the platform to run those applications.

Guide-level explanation
-----------------------

Explain the feature/change as if to a new contributor or user. Include UX flows, examples, and diagrams as needed.

The first step towards building a just-in-time intervention architecture is to design a generalisable and scalable architecture. This is an essential first step that could help us understand what we could achieve and what the constraints and limitations are. As can be seen in the figure 1, we aim to drive the JITAI system using configs, as shown in the config below.

```yaml
model_name: Sample  # No model needed
model_description: Sample
data:
  table: sample_table
  project: Sample project
tasks:
  - task_name: inference
    type: inference
    depends_on: missing_data_checks
    schedule: "* * * * *"
    path: /inference/sample_project/inference_me.py
    config:
      model_version: best
      mlflow_experiment_name: sample experiment
      server: airflow (will run it in the airflow operator)
    params:
      threshold: 0.80
    actions:
      - type: "send_notification"
          output_key: "is_anomaly"
          value_condition: "eq"
          value: "true"
        config: []
      - type: "alert"
        config:
          method: "email"
      - type: "alert"
        config:
          method: "api_request"

  - task_name: training
    type: traininig
    depends_on: missing_data_checks
    enabled: false
    schedule: "0 8 * * *"
    git_repo: github.com/radar-base/radar-ml
    path: /projects/sample_project/train_me.py
    config:
      server: "localhost:9000"
      mlflow_experiment_name: sample experiment
    params:
      batch_size: 256
      num_epochs: 100

  - task_name: missing_data_checks
    type: data_checks
    enabled: true
    schedule: "0 8 * * *"
    checks:
      - type: "missing_data"
        tables: sample_table
        window_minutes: 15
        min_records: 10
    actions:
      - type: "send_notification"
        output_key: "is_anomaly"
        value_condition: "eq"
        value: "true"
        config: []
      - type: "alert"
        config:
          method: "email"
      - type: "alert"
        config:
          method: "api_request"

```

![Airflow UI](https://external-content.duckduckgo.com/iu/?u=http%3A%2F%2Fdrive.google.com/uc?id=1_VzuJr76wYgGTw-0LqIARIZre309suCq)

Reference-level design
----------------------

Precise, technical details: architecture, data models, APIs, configuration, migrations, security/privacy considerations, performance characteristics, and failure modes.

### Technical details

Those configs can be stored in a GitHub Repo, which we can sync frequently using a GitHub sync in the Kubernetes cluster as described in Figure 1 below.

![Airflow UI](https://docs.google.com/drawings/d/e/2PACX-1vRFoXLS5zKhosYTjzFOb9ivlnrnwPnBoliy9-AFSKfVsUAMXJGU5aMfZO0Vi6dz05Sr2yXV2-9PwUIe/pub?w=1218&h=825)

**Figure 1: JITAI System Architecture** - The figure shows the working of the just-in-time intervention system in detail. We intend to have a GitHub repository where we can store the configs for just-in-time interventions. Utilising the GitHub sync, which would regularly sync the content of the repo to Airflow, Airflow will read the configs and generate the tasks stated in the config files. It can then perform the operations according to the given schedule and send the intervention actions to NiFi. Then NiFi can perform the specific action based on the intervention action condition.

Then an orchestration tool (Airflow) would read those configs and generate multiple data processing pipelines. The data processing and prediction flow can be scheduled from the config file initially, but later can be modified from the UI. The same applies to any parameters concerning the process.

Once the flow pipeline is in place, it can read data directly from any Kafka topics, from a database or from data warehouses and datalakes. For example in the case of training the model, we would probably need the whole historical dataset so it would make sense to read the data from the database or data warehouses. However for inference, it would be more efficient to read the data from the Kafka topics themselves in order to reduce the latency.

Another factor that we have considered is that we could possibly need GPUs for training or even for inference if we are using large deep learning models. This could add substantive cost if we do it in AWS, where radar-base is deployed. Hence, we decided it would be good to have a feature to offload the training/inference process to external servers, where we have already secured GPUs. We can deploy the model training module there, and it can run the training when requested. This could save the cost of renting GPUs on AWS. We’ll make the training module run as a REST client so it can be deployed to any external servers.  Additionally, we’ll deploy a machine learning lifecycle tool to monitor the training process and hyperparameters. We have used MLflow in the previous system and will use it in this iteration as well.

Lastly, we’ll deploy an ETL tool to process the streaming data in real time. This can accelerate the feature generation process especially during the just in time interventions. We can further plan to create an ETL extension that can run the pipelines in RADAR-base analytics using radarpipeline. Since ETL frameworks like Apache Flink already use Spark in the backend, it would work well with radarpipeline. Moreover we can create a Flink connector in the radarpipeline using PyFlink making it easy to run pipelines.

Moreover, NiFi provides several easy-to-create, customisable pipelines that can send requests to the RADAR-App server for sending participants' notifications and to external servers such as clinician dashboards or emails to research time about participant disengagement. Nifi supports a wide range of processors, which makes it an ideal tool to schedule different types of interventions and makes it easier to integrate newer requirements.


Compatibility and migration
---------------------------

Not applicable for this RFC.

Alternatives considered
-----------------------

For the orchestration tool, we considered using other tools like Prefect and Gagstart. However, we decided to go with Airflow because of its wide adoption, extensive community support, and rich ecosystem of plugins and operators. Airflow's flexibility in defining complex workflows using Python code also made it a suitable choice for our requirements. Moreover, Airflow allow us to add authentication and role based access control which is essential for a clinical platform. This is a paid service in other orchestration tools. This is the primary reason we chose Airflow over other orchestration tools.


Operational considerations
--------------------------

Rollout, monitoring, observability, feature flags, and rollback strategy.

We can monitor the JITAI platform using Airflow's built-in monitoring tools. Airflow provides a web-based UI that allows us to monitor the status of our workflows, view logs, and set up alerts for failures. We can also integrate Airflow with external monitoring tools like Prometheus and Grafana for more advanced monitoring and alerting capabilities.

Security and privacy
--------------------

The biggest security and privacy concern with the JITAI platform is that it allows to run custom code (for inference and training) which could potentially access sensitive participant data. To mitigate this risk, we have to implement strict access controls and code review processes for the configs file to ensure that only trusted code is executed on the platform. The initial version of the platform will only allow code from trusted repositories to be executed. In the future, we can explore using sandboxing techniques to further isolate the execution environment and prevent unauthorized access to sensitive data.

Testing strategy
----------------

Unit/integration/e2e tests, validation plans, test data, and success metrics.

We're planning to implement unit tests for individual components of the JITAI platform, including the config parser, data processing modules, and action handlers. Integration tests will be conducted to ensure that the different components work together seamlessly. End-to-end tests will simulate real-world scenarios to validate the entire workflow from config ingestion to intervention delivery.

Open questions
--------------


1. How to best manage and version control the intervention configurations?
2. What are the latency requirements for different types of interventions?
3. How to ensure the security and privacy of sensitive participant data during real-time processing?
4. What are the scalability considerations for handling a large number of participants and interventions simultaneously?
5. How to effectively monitor and log the performance of the JITAI platform?
6. What are the best practices for integrating machine learning models into the JITAI platform?
7. How to handle failures and retries in the intervention delivery process?


References
----------

Links to prior art, related issues/PRs, and documentation.
