# Header-Based Avro Subject Name Routing for Kafka Connect

## What This Does
Allows a Kafka Connect application to resolve Avro schema subject names from Kafka records headers. Specifically, if the `cogent_extraction_avro_subject_name` is set and not empty, the value of the header will be used as the schema subject name to deserialize the record value. If the `cogent_extraction_avro_subject_name` header is empty or does not exist, the schema name will be set to `__cogent_extraction_avro_subject_name_unavailable__`, which represents a schema that does not exist. The record will be routed to the connector's DLQ topic.

## Quick Start

### Setup
```
brew install openjdk@17
sudo ln -sfn /opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk-17.jdk
brew install maven
```

For this session, set Java 17 as the default Java version

```
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
export PATH=$JAVA_HOME/bin:$PATH
java -version
```

### Build
```bash
cd schema-registry
mvn clean package -pl avro-converter -DskipTests -Dcheckstyle.skip=true -Dspotbugs.skip=true
```

### Test
```bash
cd schema-registry
mvn test -pl avro-converter -Dtest=HeaderBasedAvroConverterTest#testOriginalCogentScenario -Dcheckstyle.skip=true -Dspotbugs.skip=true
```

### Deploy
```bash
git add .
git commit -m "..."
git tag v7.6.0-cogent-{release}
git push origin {branch}
git push origin v7.6.0-cogent-{release}
```

## Create Release

### Build the JAR
```bash
mvn clean package -pl avro-converter -DskipTests -Dcheckstyle.skip=true -Dspotbugs.skip=true
```

### Create GitHub Release
- Go to https://github.com/cogent-security/schema-registry/releases
- Click "Draft a New Release"
- Select Tag: `v7.6.0-cogent-{release}`
- Add a Title and Description
- Attach the newly created binary `avro-converter/target/components/packages/confluentinc-kafka-connect-avro-converter-7.6.0.zip`
- Publish release

### 2. Test Release
```bash
# Download and verify
wget https://github.com/cogent-security/schema-registry/releases/download/v7.6.0-cogent-{release}/confluentinc-kafka-connect-avro-converter-7.6.0.zip
unzip confluentinc-kafka-connect-avro-converter-7.6.0.zip
jar -tf confluentinc-kafka-connect-avro-converter-7.6.0/lib/kafka-connect-avro-converter-7.6.0.jar | grep HeaderBasedAvroConverter
```

## Usage in MSK Connect

In `cogent-java`, run the following to create a new kafka-iceberg connector with the custom HeaderBasedAvroConverter.
```
cd flink-pipelines/ingestion/src/main/iceberg-connector-plugin
al core-cicd
./create_kafka_iceberg_connector.sh
```

This is how the jars are actually extracted in `create_kafka_iceberg_connector.sh`
```
...
COGENT_TAG=v7.6.0-cogent-{release}
COGENT_BASE=https://github.com/cogent-security/schema-registry/releases/download/$COGENT_TAG
wget -O cogent-converter.zip $COGENT_BASE/confluentinc-kafka-connect-avro-converter-$VER.zip

# Extract the converter and its dependencies to lib/
echo "Extracting Cogent converter to lib/..."
unzip -j cogent-converter.zip "*/lib/*" -d lib/
rm cogent-converter.zip
...
```

Update your connector configuration to use a new `value.converter`

```properties
value.converter=com.cogent.kafka.connect.HeaderBasedAvroConverter
value.converter.schema.registry.url=https://psrc-0kywq.us-east-2.aws.confluent.cloud
value.converter.schema.registry.basic.auth.credentials.source=USER_INFO
value.converter.schema.registry.basic.auth.user.info=YOUR_CREDENTIALS
value.converter.schemas.enable=true
```

Messages now route to schema subject names based on their `cogent_extraction_avro_subject_name` header.

## Example Scenario

**Topic:** `pacific-m365-user`  
**Header:** `cogent_extraction_avro_subject_name=m365-user-value`  
**Result:** Message uses schema from subject `m365-user-value`

**Topic:** `pacific-m365-user`  
**Header:** *(missing)*  
**Result:** Message routed to DLQ with subject `__cogent_extraction_avro_subject_name_unavailable__`