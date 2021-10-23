# Logstash to Splunk
writeup about sending Logstash data to Splunk using the HTTP Event Collector

## 1. (Splunk) Create a HTTP Event Collector token

[Official Documentation: Configure HTTP Event Collector on Splunk Enterprise](https://docs.splunk.com/Documentation/Splunk/8.2.2/Data/UsetheHTTPEventCollector)

Settings --> Add Data --> Monitor --> HTTP Event Collector

	Name: "webtoken" (or any desired name)
	UNCHECK: Enable indexer acknowledgement

click **Next**

*this is where you will decide which index this token is written to upon being received*

	Select Allowed Indexes --> click the desired index (or create a new one, "logstash", for demo purposes)

click **Review** --> **Submit**

	copy the token and save it for later
	# note this token was used for this writeup, then deleted
		c6012558-7817-45e0-a3a5-7dfc876e1bf3

## 2. (Splunk) Enable Tokens Globally

Settings --> Data inputs --> HTTP Event Collector --> Global Settings (top right)

*this is where you will decide which source type this token is written to at index time*

	# note HTTP Event Collector is where the token can be retrieved again if needed
	All Tokens: Enabled
	Default Source Type: Structured --> _json
	Default Index: logstash
	UNCHECK: Use Deployment Server
	UNCHECK: Enable SSL (will be added later in the instructions)
	HTTP Port Number: 8088 (default)

## 3. (Logstash) Output Statement

*note the usage of the /services/collector/raw endpoint; the event (vs raw) endpoint is unreliable when sending POST data from Logstash*

	output {
		http {
			content_type => "application/json"
			http_method => "post"
			url => "http://your-splunk-server:8088/services/collector/raw"
			headers => ["Authorization", "Splunk c6012558-7817-45e0-a3a5-7dfc876e1bf3"]
		}
	}

## TO DO
- generate SSL/TLS certificates
- configure the Logstash truststore and keystore

## Considerations
- use one token per index+sourcetype, and name/describe them accordingly
- harden and firewall your assets; do not allow the entire world, or even the whole corporate network, to reach the ingest ports
- certificates don't last forever; include the Splunk keys in your key management processes (and don't forget the Logstash stores)
- Logstash --> Kafka <-- Splunk: this setup is more durable and reduces data loss if the Splunk cluster goes down for some reason
