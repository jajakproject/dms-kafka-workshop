# Setup MSK
## Prepare security group
Create a security group in DMSworkshop VPC called `sg-MSK`
Configure inbound rules like this: to allow 9092,9094 and 2181 from default and DMS EC2

![Pasted image 20230411212713](https://user-images.githubusercontent.com/108851851/231217435-e02ed370-3ade-4a88-ab10-3999eca9c462.png)

Enable SSH on default sg, enable 80 port on default sg as well

![image](https://user-images.githubusercontent.com/108851851/231217592-34a399a4-9988-4185-bd8a-ae4902434964.png)

Create EC2 Linux, use the default sg in DMS VPC, enable assign **Public IP**.

## Create MSK Cluster
Search MSK to open MSK Console, then click **Cluster** on the left menu.
<img width="790" alt="image" src="https://user-images.githubusercontent.com/108851851/231217969-6833eb32-65db-406f-a22b-fd47801c497d.png">

Click **Create cluster**

<img width="627" alt="image" src="https://user-images.githubusercontent.com/108851851/231218445-b52d713d-24b1-4bc4-a7ea-b49ada4223ba.png">

Choose **Custom Create**

<img width="858" alt="image" src="https://user-images.githubusercontent.com/108851851/231218681-f7dc9ab5-d6c8-4d5a-b937-429acbee73f8.png">

1. Give it a customer name `MSK-DMS-Workshop`.
2. Cluster type choose **Provisioned**.
3. Kafka version choose **2.8.1**.
4. Broker type choose **kafka.m5.large**.
5. Number of zones choose **3**.
6. Brokers per zone choose **1**.
7. Scroll down to `Configuration`.
8. Choose **Customised configuration**.
9. Click **Create Configuration**.
10. Give it a name `MSK-Configuration`
11. Scroll down and modify the configuration, we only change one setting:
**auto.create.topics.enable=true**
12. Click **Create**
13. Back to the MSK creation tab, click **Next**.
14. Choose VPC **Dms-Vpc**.
15. Choose the 3 zone with different AZ, pick the corresponding subnet.

<img width="848" alt="image" src="https://user-images.githubusercontent.com/108851851/231220811-491838e9-b8b5-4a6d-9aa1-575d5d3b9677.png">

16. Remove default security group, add the one we created `sg-MSK`
17. Click **Next**
18. In security page, tick **Unauthenticated access**, untick **IAM role-based authentication**.
19. Scroll down to encryption, tick **Plaintext**.
20. Click **Next**
21. Click **Next**
22. Click **Create cluster**

This step will take you 10 minutes. Feel free to take a break.

## Setup Kafka Client and Kafdrop
SSH into the EC2 we created.
```bash
ssh -i DMSKeyPair.pem ec2-user@<Public IP for the EC2>
```
DL Kafka
```bash
mkdir kafka
cd kafka
wget https://dlcdn.apache.org/kafka/3.4.0/kafka_2.13-3.4.0.tgz
tar -xzf kafka_2.13-3.4.0.tgz
cd kafka_2.13-3.4.0

```
Install Java
```bash
sudo yum install java-17
```

Use the consumer CLI to read record
```bash
bin/kafka-console-consumer.sh --topic kafka-default-topic --from-beginning --bootstrap-server b-1.mskworkshopcluster.ncqogm.c14.kafka.us-east-1.amazonaws.com:9092
```
Install kafdrop
```bash
cd /home/ec2-user
mkdir kafdrop
cd kafdrop
wget https://github.com/obsidiandynamics/kafdrop/releases/download/3.31.0/kafdrop-3.31.0.jar

```
Run kafdrop
```bash
sudo java --add-opens=java.base/sun.nio.ch=ALL-UNNAMED \
    -jar kafdrop-3.31.0.jar \
    --kafka.brokerConnect=<boostrap server IP1:port, IP2:port, IP3:port...> \
    --server.port=80
    
```

Access thru the public IP of that EC2

![Pasted image 20230411214146](https://user-images.githubusercontent.com/108851851/231222423-3ff80e51-5be8-49e1-bc2e-fbf31228849d.png)

## Setup DMS Target Endpoint
1. Go to DMS Console, Click **Create Endpoint**.
2. Endpoint Type choose **Target endpoint**.
3. Endpoint identifier input `msk-target`.
4. Target engine choose **Kafka**.
5. Go to MSK console, click the cluster we created. Then click **View client information**

<img width="1088" alt="image" src="https://user-images.githubusercontent.com/108851851/231224057-b7d77311-f8e0-47a0-a68f-fa53b5cedfbf.png">

6. Copy the Plaintext Private endpoint

<img width="1091" alt="image" src="https://user-images.githubusercontent.com/108851851/231224175-c81c39b9-1f1d-4b6b-82f4-5a34a5fa2e36.png">

7. Paste the value into Broker field

<img width="858" alt="image" src="https://user-images.githubusercontent.com/108851851/231224344-b6e3abea-76ff-4055-9807-37a0ddfd74c9.png">

8. Click **Save**.

## Setup DMS Task
1. Go to DMS Console, click **Database migration tasks**, click **Create database migration task**.
2. Name it `MSK-DMS-Task`
3. Replication instance choose the instance we provisioned.
4. Source database endpoint choose **sqlserver-source**.
5. Target database endpoint choose **msk-target**.
6. Migration type choose **Migrate existing data and replicate ongoing changes**.
7. In Task settings, Target table preparation mode choose **Do nothing**.
8. Scroll down and tick **Turn on CloudWatch logs**.
9. Scroll down to Table mappings, click **Add new selection rule**.
10. Schema choose **Enter a schema**.
11. Source name input `dbo`.
12. Source table name input `person_dump`.
13. Click **Add new selection rule** again.
14. Schema choose **Enter a schema**.
15. Source name input `dbo`.
16. Source table name input `player`. (don't use person table, it is too large)
17. Click **Create task**.
18. Wait for the task starts, visit kafdrop, check to see if any message streamed into Kafka topic.

## Map data into different topics
1. Click **Action** then choose **Stop**
2. Wait till the task is stoped, click **Action** then choose **Modify**.

<img width="1128" alt="image" src="https://user-images.githubusercontent.com/108851851/231227234-4147557e-b5d4-4962-8c3f-063368319df6.png">

3. Scroll down to Table mappings, choose **JSON editor**. Use the following JSON which appends two object mappings to our previous settings.
We want to deliver table changes into different topics.

```JSON
{
    "rules": [
        {
            "rule-type": "selection",
            "rule-id": "221047323",
            "rule-name": "221047323",
            "object-locator": {
                "schema-name": "dbo",
                "table-name": "player"
            },
            "rule-action": "include",
            "filters": []
        },
        {
            "rule-type": "selection",
            "rule-id": "204703346",
            "rule-name": "204703346",
            "object-locator": {
                "schema-name": "dbo",
                "table-name": "person_dump"
            },
            "rule-action": "include",
            "filters": []
        },
        {
            "rule-type": "object-mapping",
            "rule-id": "3",
            "rule-name": "MapToKafka1",
            "rule-action": "map-record-to-record",
            "kafka-target-topic": "person_dump_topic",
            "object-locator": {
                "schema-name": "dbo",
                "table-name": "person_dump"
            }
        },
        {
            "rule-type": "object-mapping",
            "rule-id": "4",
            "rule-name": "MapToKafka2",
            "rule-action": "map-record-to-record",
            "kafka-target-topic": "player_topic",
            "object-locator": {
                "schema-name": "dbo",
                "table-name": "player"
            }
        }
    ]
}
```

4. Resume the task
5. Try to update person_dump and player table
```SQL
update person_dump set FirstName = 'B' where lastName = 'A'
update player set full_name = 'Shead, DeShawnmm2' where ID = 5157
```
Observe the new messages from kafdrop

## Enable before image
1. Stop the task and modify.
2. Under Task settings, choose JSON editor
change line 147:
```JSON
  "BeforeImageSettings": null,
  "ControlTablesSettings": {
    "historyTimeslotInMinutes": 5,
    "HistoryTimeslotInMinutes": 5,...
```
To this:
```JSON
"BeforeImageSettings": { 
	"EnableBeforeImage": true, 
	"FieldName": "before-image", 
	"ColumnFilter": "all" 
	},
  "ControlTablesSettings": {
    "historyTimeslotInMinutes": 5,
    "HistoryTimeslotInMinutes": 5,...
```
2. Again, try to run some updates
```SQL
update person_dump set FirstName = 'B' where lastName = 'A'
update player set full_name = 'Shead, DeShawnmm2' where ID = 5157
```
Observe the new messages from kafdrop



