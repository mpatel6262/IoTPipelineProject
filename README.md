# IoT Pipeline Setup (Skip to Step 6 if first-time setup is complete)  

## Step 1: Create AstraDB Database
You will need to create an AstraDB databse and fill out its connection information in kafkaToCassandra.py and webpage.py. You can use default settings for creating this database.  

### Step 1.1: Create your table using CQL  
```
CREATE TABLE keyspace_name.new_table (
    system_id int PRIMARY KEY,
    model_number text,
    timestamp text,
    ambient_temperature float,
    battery_level int,
    humidity float,
    light float,
    pressure float
);
```
#### To Manually Add Data:

```
INSERT INTO iotdatabase.sensortaginfo (system_id, model_number, timestamp, ambient_temperature, battery_level, humidity, light, pressure) VALUES (1000, 'Manual Input', '01/22/2024 11:14:15', 20.0, 95, 20.1, 20.2, 20.3);
```

#### To Delete Table Data:

```
TRUNCATE iotdatabase.sensortaginfo;
```

### Step 1.2: Copy AstraDB connection information to kafkaToCassandra.py and webpage.py  
ASTRA_DB_ID - Unique identifer for your DB - "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"  
ASTRA_DB_REGION - The region of your DB - "us-east-1"  
ASTRA_DB_APPLICATION_TOKEN - Another identifier for your DB - "AstraCS:XXXXXXXXXXX"  
ASTRA_DB_KEYSPACE - Name of your cassandra keysapce - "iotdatabase"  
COLLECTION_NAME - Name of your table - "sensortaginfo"  


## Step 2: Create AWS Instances  
You will need three instances, with at least one large instance to host the zookeeper and kafka servers. I opted for Ubuntu on these instances and I would recommend adding a security group to allow all incoming traffic and create a .pem key to connect to the instance.  

## Step 3: Download MobaXterm  
MobaXterm is my recommended method of connecting to your AWS instances. Download Here: https://mobaxterm.mobatek.net/download-home-edition.html  

## Step 4: Download Apache Kafka
You will need to download Apache Kafka to upload to the instance which will run the Kafka server. Current testing has been done with version 2.13-3.6.0, Download Here: https://kafka.apache.org/downloads  

## Step 5: Connect to Each Instance and Complete Setup

### Step 5.1: Setup Kafka Instance

```
ssh -i "IoTProjectKey.pem" ubuntu@<PUBLIC IP OF INSTANCE>
```
```
sudo apt-get update
```

#### Upload kafka tgz  
```
tar -zxvf kafka_xxxxx.tgz
```

#### Rename folder to kafka  
```
mv old_kafka_folder kafka
```

#### Delete kafka tgz  
```
rm kafka_xxxxx.tgz
```

#### Install Java Runtime Environment  
```
sudo apt install openjdk-11-jdk
```

#### Python Setup: 

```
git clone https://github.com/mpatel6262/IoTPipelineProject.git
```
```
sudo apt install python3-pip
```
```
pip3 install pykafka
```
```
pip3 install paho-mqtt==1.6.1
```

### Step 5.2: Setup Cassandra Instance:  

```
ssh -i "IoTProjectKey.pem" ubuntu@<PUBLIC IP OF INSTANCE>
```
```
sudo apt-get update
```
```
git clone https://github.com/mpatel6262/IoTPipelineProject.git
```
```
sudo apt install python3-pip
```
```
pip3 install kafka-python
```
```
pip3 install astrapy==0.3.3
```

### Step 5.3: Setup Virtual Sensor Simulator Instance:  

```
ssh -i "IoTProjectKey.pem" ubuntu@<PUBLIC IP OF INSTANCE>
```
```
sudo apt-get update
```
```
git clone https://github.com/mpatel6262/IoTPipelineProject.git
```
```
sudo apt install python3-pip
```
```
pip3 install paho-mqtt==1.6.1
```
```
pip3 install flask
```
```
pip3 install astrapy==0.3.3
```
```
chmod +x IoTPipelineProject/run_simulators.sh
```
```
chmod +x IoTPipelineProject/stop_simulators.sh
```

## Step 6: Start the Pipeline: 

### In Kafka Instance:  

#### To Run Multiple Terminals:  
You will need to run multiple terminals for this instance (one for zookeeper server, kafka server, python script), I recommend tmux for this:  
Start Tmux: tmux  
Enable Mouse: Ctrl + B : <Enter> set -g mouse <Enter>  
Split Horizontally: Ctrl + B %  
Split Vertically: Ctrl + B "  

#### To Start Kafka:  
##### Terminal 1:  
```
kafka/bin/zookeeper-server-start.sh kafka/config/zookeeper.properties
```

##### Terminal 2:  
```
kafka/bin/kafka-server-start.sh kafka/config/server.properties
```

##### Terminal 3:  
```
kafka/bin/kafka-topics.sh --create --bootstrap-server <PUBLIC IP OF KAFKA INSTANCE>:9092 --replication-factor 1 --partitions <NUM_PARTITIONS> --topic TISENSORTAGDATA
```
```
python3 IoTPipelineProject/BrokerToKafka/brokerToKafka.py <PUBLIC IP OF KAFKA INSTANCE>
```

### In Cassandra Instance: 
```
python3 IoTPipelineProject/KafkaToCassandra/kafkaToCassandra.py <PUBLIC IP OF KAFKA INSTANCE>
```
OR
```
cd IoTPipelineProject/KafkaToCassandra
```
```
./run_consumers.sh <PUBLIC IP OF KAFKA INSTANCE> <NUMBER OF CONSUMERS>
```
```
./stop_consumers.sh
```

### In Virtual Sensor Instance:  
```
python3 IoTPipelineProject/WorkloadGenerator/simulator.py <SENSOR ID>  
```
OR
#### Configure simulators.conf with the number of each simulator you would like to start
```
cd IoTPipelineProject/WorkloadGenerator
```
```
./run_simulators.sh
```
```
./stop_simulators.sh
```

## Step 7: Monitor Data Using Webpage:  
You can start the webpage from any system, I would recommend just running the python script from your personal device. Make sure to fill out the AstraDB connection data as referenced in Step 1.2.  
```
python3 IoTPipelineProject/Webpage/webpage.py
```
You can utilize the search bar to filter by sensor_id, or press the delete button to remove a stale sensor's data from the Cassandra Database.  
