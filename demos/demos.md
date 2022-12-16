# Demos

To learn and use the ESP StateDB Windows, we will work on a real-world usecase from a Bank. All the banks today offer credit cards to account holders with a certain limit on monthly usage. Credit cards have made the payment experience very convenient and it widely used for shopping, online purchases and payments, ATM withdrawals, etc. However, misuse of credit cards, if lost or stolen is also very common. 

Through demos we will present how SAS ESP can be used for fraudulent transaction/(s) detection with aggregation using the StateDB Windows, and how SAS ESP can perform fast lookups to fetch the credit card user’s account balance. You will also see how all the ESP models leverages Kafka Message Bus for auto-scaling. 

Following are the demos we will be covering in this section:

* [Demo 1: Load Reference/Dimension data into Redis using ESP StateDB Writer Window](#demo-1-load-referencedimension-data-into-redis-using-esp-statedb-writer-window)
* [Demo 2: Verify credit card account information and update the balance amount](#demo-2-verify-credit-card-account-information-and-update-the-balance-amount)
* [Demo 3: Multiple Data Fetch with the StateDB Reader Window](#demo-3-multiple-data-fetch-with-the-statedb-reader-window)
* [Demo 4: Detect fraudulent transactions with aggregation using StateDB Reader Window](#demo-4-detect-fraudulent-transactions-with-aggregation-using-statedb-reader-window)
  * [Auto-scaling ESP Server Pods with Kafka to Detect Fraudulent transactions](#auto-scaling-esp-server-pods-with-kafka-to-detect-fraudulent-transactions)

##  Setup
To run the demos, you will need the following:

1.	ESP deployment (either [lightweight ESP (standalone)](https://github.com/sassoftware/esp-kubernetes) or [Viya 4](https://github.com/sassoftware/viya4-deployment)) running in K8s.
2. Access to SAS Event Stream Processing Studio and SAS Event Stream Manager.
3.	Here we are using [managed Redis Cluster running in Azure](https://azure.microsoft.com/en-us/products/cache/#:~:text=Azure%20Cache%20for%20Redis%20is,benefits%20of%20a%20managed%20service.). You can use any deployment of Redis. Additionally, the same models and data can be used for Singlestore, with some minor changes in the StateDB windows configurations. **OR** You can also [deploy a non-managed Redis Cluster in Kubernetes](https://docs.redis.com/latest/kubernetes/deployment/quick-start/).  
4. [Redisinsight](https://redis.com/fr/redis-enterprise/redisinsight/) for web interface to Redis database.
4. [Download all the ESP models, input dimension data, transaction events](demos/demo_examples.zip) to replicate the demo in your environments.

## Demo 1: Load Reference/Dimension data into Redis using ESP StateDB Writer Window

Figure 1 demonstrates the flow of the demo. In this demo, we load 1 million records  (dimension/reference/whitelist data)of user information with their account balances. Here we load the dimension data from a CSV file into Redis using an ESP model. 

<figure align="center">
  <img src="demos/images/Demo1_LoadReferenceData.jpg" width="80%" height="80%">
  <figcaption><i>Figure 1. Demo to Load Reference/Dimension data into Redis using ESP StateDB Writer Window</i></figcaption>
</figure>

Reference data can be loaded to Redis using data sources, pipelines, or any other mechanism as well. However, in our case, we are using ESP to do that. 

Figure 2, shows the model where the source window gets the dimension data (records) from the CSV, and the StateDB writer window write them to the Redis.

<figure align="center">
  <img src="demos/images/Demo1_ESPModel.jpg" width="20%" height="20%">
  <figcaption><i>Figure 2. ESP XML Model to load reference/Dimension data into Redis</i></figcaption>
</figure>

Below is the configuration for the StateDB writer window. Change the values for the hostname and password with your Redis credentials.

```sh
        <window-statedb-writer name="StateDBWriter">
          <statedb type="redis" cluster="false" hostname=" dgespkuberedis" port="6379" password="JAhWt5FlCeEq”/>
          <write prefix="userbalance" del-dead-sec-keys="true">
            <output>
              <field-selection name="userid" db-name="userid"/>
              <field-selection name="startTimestamp" db-name="startTimestamp"/>
              <field-selection name="endTimestamp" db-name="endTimestamp"/>
              <field-selection name="balance" db-name="balance"/>
            </output>
          </write>
        </window-statedb-writer>
````
When the above model is executed, you will see the output like in the Figure 3. 

<figure align="center">
  <img src="demos/images/Demo1_ESPModelRun.jpg" width="80%" height="80%">
  <figcaption><i>Figure 3. ESP XML Model Execution in ESP Studio</i></figcaption>
</figure>

Figure 4 shows how the reference data records are then seen in Redis. In our test sample, all the users start at the same balance of $10000. 

<figure align="center">
  <img src="demos/images/Demo1_Redis.jpg" width="80%" height="80%">
  <figcaption><i>Figure 4. Reference data in Redis</i></figcaption>
</figure>

## Demo 2: Verify credit card account information and update the balance amount

In demo 2, credit card users make transactions and at every transaction, a new event is generated which must be processed in real-time. So when a user uses the card, it is checked if the transaction amount is possible or not. 

Figure 5 demonstrates the flow of this demo. The transaction events are sent to the ESP where the StateDB Reader window fetches the required information from the Redis.  In Redis, we verify if the balance is higher than the transaction amount and if it is true, a new transaction event is written in the Redis and the amount balance is updated. Additionally, if the balance is lower than the desired transaction amount, an alert message is sent to the credit card holder and the transaction is canceled. 

<figure align="center">
  <img src="demos/images/Demo2_VerifyandUpdateAmount.jpg" width="80%" height="80%">
  <figcaption><i>Figure 5. Demo 2 to Verify Credit Card Account Information and Update the Balance Amount</i></figcaption>
</figure>

Figure 6 presents the ESP model corresponding to this demo. Here, ESP receives a transaction event at the `source window`. Then, ESP will fetch the balance amount of this particular user which made the transaction using the `StateDB Reader window`. We will get user details, user ID, and the balance amount. If the balance is less than the transaction amount, it will give an alert message in the `compute window`. But if it is not, then two actions will happen. First, this new transaction will be added to the Redis and the current balance will be modified to reflect the new transaction. So basically, the transaction amount will be subtracted from the balance and written back to the Redis. 

<figure align="center">
  <img src="demos/images/Demo2_ESPModel.jpg" width="50%" height="50%">
  <figcaption><i>Figure 6. ESP XML Model to Verify Credit Card Account Information and Update the Balance Amount</i></figcaption>
</figure>

Now you can run the model using the historic transaction test data ` historicaltransaction100k.csv`. Figure 7 shows how the `newbalance` is created by subtracting the `amount` from the `balance`. It is important to note that we are first reading the record from the Redis and then modifying it. Not only that, two separate write operations are happening in parallel, i.e., writing the new transaction to Redis and writing the newly computed balance.

<figure align="center">
  <img src="demos/images/Demo2_ESPModelRun.jpg" width="80%" height="80%">
  <figcaption><i>Figure 7. ESP XML Model Execution in ESP Studio</i></figcaption>
</figure>

**NOTE:** The time difference between the two transactions from the same user must be greater than 1 millisecond. This is usually the case in general as no user makes two transactions within a period of 1 millisecond. If that happens, then the read before write (second transaction) operation can give inconsistent results. We encounter this because ESP is processing way faster than the Redis (read and write to Redis). However, this is an edge case and never happens in real-world scenarios.

## Demo 3: Multiple Data Fetch with the StateDB Reader Window

In this demo, we will demonstrate fetching all the matching transactions for a credit card user and count the number of transactions above $1000. Figure 8 presents the simple flow of this demo. If the number of transactions above $1000 is greater than or equal to 2 (including the historic transactions in Redis and the current transaction), an alert will be generated. It is not a simple aggregation as we need to filter out the transactions based on a condition, i.e., `#Tx > $1000`. So, here we need to perform multiple fetches for each of the incoming events with filtering to get only those transactions that fulfill the condition (including the current transaction).

**NOTE:** In the demo, we will not implement the alert part.

<figure align="center">
  <img src="demos/images/Demo3_MultipleFetchBeforeWrite.jpg" width="80%" height="80%">
  <figcaption><i>Figure 8. Demo to Demonstrate Multiple Data Fetch with the StateDB Reader Window</i></figcaption>
</figure>

Figure 9, shows the ESP XML model flow of this demo. We get the transaction events in the `Source Window`. In the `StateDB Reader Window` we fetch all the records of the current user. This is where we perform multiple fetches from Redis. `Filter Window` then filters out all the transactions above $1000. Note that we are using a stateful model with an `Aggregate Window` which counts the number of transaction events above $1000 followed by another  `Filter Window` that counts these transactions and if the number of transactions exceeds or equals 2, an alert is generated. 

<figure align="center">
  <img src="demos/images/Demo3_ESPModel.jpg" width="20%" height="20%">
  <figcaption><i>Figure 9. ESP XML Model for Multiple Data Fetch with the StateDB Reader Window</i></figcaption>
</figure>

Figure 10, shows all the input transactions. To keep it simple and to understand the functionality of the demo, we are sending two transaction events generated by `userid_20` and `userid_21`. 

<figure align="center">
  <img src="demos/images/Demo3_sourceWindowInputs.jpg" width="80%" height="80%">
  <figcaption><i>Figure 10. Input Events to Source Window in the ESP Model</i></figcaption>
</figure>

In Figure 11, we see the fetched events for the `useri_20`. These are all `insert` events. 

<figure align="center">
  <img src="demos/images/Demo3_inserts.jpg" width="80%" height="80%">
  <figcaption><i>Figure 11. Fetched "insert" events in ESP StateDB Reader Window</i></figcaption>
</figure>

However, we also see corresponding `delete` events for those `insert` events in Figure 12. These delete events are generated in the `StateDB Reader Window` via the property **generate-deletes="true"**. 
This is required to delete the temporary state created by the model in the internal memory of the ESP server pod. Here we use internal memory as a transient storage to keep the matching records during the processing of that event. Once the event is processed, the matching records from the internal memory are deleted. Do not confuse this `generate-deletes` property with Time-To-Die (TTD) in Redis. They are different.

<figure align="center">
  <img src="demos/images/Demo3_deletes.jpg" width="80%" height="80%">
  <figcaption><i>Figure 12. Fetched "delete" events in ESP StateDB Reader Window</i></figcaption>
</figure>

Figure 13, `Filter Window` shows the final results with the count of transactions higher than $1000. 

<figure align="center">
  <img src="demos/images/Demo3_filterResults.jpg" width="80%" height="80%">
  <figcaption><i>Figure 13. Final results in the Filter Window</i></figcaption>
</figure>

## Demo 4: Detect fraudulent transactions with aggregation using StateDB Reader Window

In this demo, we demonstrate the detection of fraudulent transactions based on the average aggregation of the transactions. Figure 14 presents a Credit Card (CC) holder doing several transactions over a period. Now, maliciously, a person obtains the credit card information of the CC holder. He/She performs a transaction of a large amount which is much higher than the average amount of transactions done by the CC holder. In this case, a bank would suspect this transaction to be a fraudulent one and would want to inform the CC holder about it by an alert SMS. 
Many banks now send such alert messages that if this transaction is not by you, please report the bank. 

Here, the average of transactions can be computed over a month, months, or even a year. And this is the retention policy for the transactions stored in Redis. 

<figure align="center">
  <img src="demos/images/Demo4_SimpleAggregation.jpg" width="80%" height="80%">
  <figcaption><i>Figure 14. Demo to demonstrate Detection of fraudulent transactions with aggregation using StateDB Reader Window</i></figcaption>
</figure>

Figure 15 shows the model which receives the transaction events in the `Source Window`. In the `StateDB Reader Window`, it fetches all the records of the user (user is obtained from the current transaction event) and calculates the average amount of all the transactions done in a given period. The `Filter Window` filters out the events where the new transaction amount is greater than the average amount of the previous historic transactions. `Compute Window` shows the alert message with the amount of high value. 

<figure align="center">
  <img src="demos/images/Demo4_ESPModel.jpg" width="20%" height="20%">
  <figcaption><i>Figure 15. ESP XML Model for Simple Aggregation</i></figcaption>
</figure>

Figure 16 shows the results of the `StateDB Reader Window` which displays the `avgamount` of the previous historic transactions along with the `amount` of the new *malicious* transaction amount for that user to compare with. 

<figure align="center">
  <img src="demos/images/Demo4_StateDBReaderOutput.jpg" width="90%" height="90%">
  <figcaption><i>Figure 16. Results of StateDB Reader Window</i></figcaption>
</figure>

In below Figure 17, the `Filter Window` shows all the results where the *malicious* transaction amount exceeds the average amount (of all the transactions) for that user. 

<figure align="center">
  <img src="demos/images/Demo4_FilterOutput.jpg" width="90%" height="90%">
  <figcaption><i>Figure 17. Results of Filter Window</i></figcaption>
</figure>

Finally, Figure 18 displays the result from the `Compute Window` which shows the alert message for all the transactions where the amount was higher than the average amount.

<figure align="center">
  <img src="demos/images/Demo4_ComputeOutput.jpg" width="90%" height="90%">
  <figcaption><i>Figure 17. Results of Compute Window</i></figcaption>
</figure>

### Auto-scaling ESP Server Pods with Kafka to Detect Fraudulent transactions

In this demo, we will extend the ESP XML model from Demo 4 to auto-scale to handle the increasing incoming events. We will use Kafka to stream the incoming events to the `ESP Source Window` in the model. You can use any deployment of Kafka, be it [managed Kafka by Confluent]( https://www.confluent.io/confluent-cloud/?utm_medium=sem&utm_source=google&utm_campaign=ch.sem_br.nonbrand_tp.prs_tgt.kafka_mt.xct_rgn.emea_lng.eng_dv.all_con.kafka-azure&utm_term=apache%20kafka%20azure&creative=&device=c&placement=&gclid=CjwKCAiAv9ucBhBXEiwA6N8nYO2EqHMGF5Wt_1pJp5MkznVtxDVdu0jv_QQRZ-ZTjYxMLaH6VV8YRRoC0_gQAvD_BwE) or [non-managed Kafka from Strimzi in Kubernetes]( https://strimzi.io/). 

In the `Source Window`, we have configured the Kafka connector as shown below. It reads the input events from the input topic `intopic` which is streamed with the incoming events. We create 10 partitions in the `intopic`.

**NOTE** We do not define any partition number so that auto-scaling ESP server pods can read from any partition selected by Kafka to its consumers in the same consumer group after rebalancing every time an ESP server pod joins. 

```sh
<window-source index="pi_EMPTY" name="Source">
          <schema>
            <fields>
              <field name="id" type="double" key="true"/>
              <field name="userid" type="string" key="true"/>
              <field name="timestamp" type="stamp" key="true"/>
              <field name="amount" type="double"/>
            </fields>
          </schema>
          <connectors>
            <connector class="kafka" name="New_Connector_1">
              <properties>
                <property name="type"><![CDATA[pub]]></property>
                <property name="kafkahostport"><![CDATA[10.0.29.73:9092]]></property>
                <property name="kafkatopic"><![CDATA[intopic]]></property>
                <property name="kafkaconsumergroupid"><![CDATA[group0]]></property>
                <property name="urlhostport"><![CDATA[unused:33333]]></property>
                <property name="kafkatype"><![CDATA[csv]]></property>
                <property name="dateformat"><![CDATA[%Y-%m-%d %H:%M:%S]]></property>
                <property name="kafkaglobalconfig"><![CDATA[auto.commit.interval.ms=3000]]></property>
              </properties>
            </connector>
          </connectors>
        </window-source>
````

Figure 18 demonstrates the high-level architecture of the auto-scaling ESP server pods with Kafka to detect fraudulent transactions. 

For this demo, we have 1 million user information with a starting balance in the Redis. We have used another ESP model to stream user transactions to Kafka input topic `intopic`. 

<figure align="center">
  <img src="demos/images/Demo4_ScalableSimpleAggregation.jpg" width="80%" height="80%">
  <figcaption><i>Figure 18. High-level Architecture of Auto-scaling ESP Server Pods with Kafka to Detect Fraudulent transactions</i></figcaption>
</figure>
 
We run both the models,`Scalable_AggregationTransaction_Redis.xml` and `inputtransactions.xml` from SAS Event Stream Manager.  

Figure 19 shows how we load and start the `Scalable_AggregationTransaction_Redis.xml` model. 

<figure align="center">
  <img src="demos/images/Demo4_LoadProjectinESM.jpg" width="40%" height="40%">
  <figcaption><i>Figure 19. Load Scalable Model Scalable_AggregationTransaction_Redis.xml in SAS ESM</i></figcaption>
</figure>

We also provide the deployment settings in the SAS ESM. We configure CPU, memory, and number of min and max replicas for scaling. Figure 20, demonstrates how to provide these settings. 

<figure align="center">
  <img src="demos/images/Demo4_DeploymentSettingsESM.jpg" width="40%" height="40%">
  <figcaption><i>Figure 20. Deployment Settings for Model Scalable_AggregationTransaction_Redis.xml in SAS ESM</i></figcaption>
</figure>

Once we start the model, you can see and review it as shown in Figure 21. 

<figure align="center">
  <img src="demos/images/Demo4_ScalableModelStarts.jpg" width="80%" height="80%">
  <figcaption><i>Figure 21. Model Scalable_AggregationTransaction_Redis.xml Starts in SAS ESM</i></figcaption>
</figure>

Now, we can start streaming the incoming events to Kafka `intopic` so that the scalable model can start processing the events. Figure 22, shows how we load and configure the `inputtransactions.xml` model. Note this model does not scale. 

<figure align="center">
  <img src="demos/images/Demo4_LoadWriteTransAndPropESM.jpg">
  <figcaption><i>Figure 22. Load and Configure inputtransactions.xml model in SAS ESM</i></figcaption>
</figure>

Once, the user transaction events start to flow in the Kafka, then with time, the scalable model starts to scale to handle the increasing load as shown in Figure 23. 

<figure align="center">
  <img src="demos/images/Demo4_scalingServers.jpg" width="80%" height="80%">
  <figcaption><i>Figure 23. Scalable Model in ESM</i></figcaption>
</figure>



