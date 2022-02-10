# Logstash to Splunk
This is a writeup about sending Logstash data to Splunk using the HTTP Event Collector. Logstash provides an immense filtering capability which can be used to screen and reduce data before it hits the Splunk indexers. Additionally, Logstash can accept a vast array of inputs, and provides a middleware capability when translating Elastic Beats into Splunk logs. Using the Elastic Common Schema inside Splunk works perfectly well, as dotted fields (ex. source.ip) function without issue in the pipeline, and standard field renames (ex. turn periods into underscores, etc) can be applied either in Splunk or Logstash. Further decoupling of services, and increased data durability, can be achieved by using a Kafka cluster between Logstash and Splunk, but that is beyond the scope of this writeup.

# Considerations
| Issue | Logstash | Splunk |
| -- | -- | -- |
| Transformations | LS permanently alters data before reaching Splunk | SP can update values dynamically due to search-time extraction |
| Infrastructure | LS is not cluster-aware and requires orchestration management | SP can deploy policies directly to the ingest and indexing tiers |
| Migration | Existing Beats+LS infrastructure still works up until it reaches Splunk | SP cannot manage Logstash, Beats, or other non-Splunk agents directly* |

\* = Splunk Deployment Servers can manage endpoint software in some circumstances

## 1. (Splunk) Create a new HTTP Event Collector token

[Official Documentation: Configure HTTP Event Collector on Splunk Enterprise](https://docs.splunk.com/Documentation/Splunk/8.2.2/Data/UsetheHTTPEventCollector)

Settings --> Add Data --> Monitor --> HTTP Event Collector

	Name: "webtoken" (or any desired name)
	UNCHECK: Enable indexer acknowledgement

click **Next**

*this is where you will decide which index this token is written to upon being received*

	Create a new index
		Index Name: logstash
		(leave rest defaults for this demo)
		Save
	Select Allowed Indexes --> click logstash to "whitelist" the index for this token

click **Review** --> **Submit**

	copy the token and save it for later
	# note this token was used for this writeup, then deleted
		c6012558-7817-45e0-a3a5-7dfc876e1bf3

## 2. (Splunk) Enable Tokens Globally

Settings --> Data inputs --> HTTP Event Collector --> Global Settings (top right)

*these default source type settings will not affect which actual source type this token is written to, as we will change that in the following step (note you cannot create a new source type from this panel)*

	# note HTTP Event Collector settings screen is where the token can be retrieved again if needed
	All Tokens: Enabled
	Default Source Type: Structured --> _json
	Default Index: logstash
	UNCHECK: Use Deployment Server
	UNCHECK: Enable SSL (will be added later in the instructions)
	HTTP Port Number: 8088 (default)

## 3. (Splunk) Create Source Type

Settings --> Source types --> New Source Type (top right)

*it is very important to set Indexed Extractions to "none", otherwise Splunk will display duplicated data as it double-parses the JSON*

	Name: winlogbeat
	Description: Windows endpoint data
	Destination app: Search & Reporting
	Category: Operating System
	Indexed Extractions: none
	# additional configurations should be made to set "@timestamp" as the document time field with the appropriate time zone, but that is not critical to this demo as both _time and @timestamp can be utilized for the same purpose

## 4. (Splunk) Set Source Type for Token

Settings --> Data inputs --> HTTP Event Collector --> Edit (on appropriate source type name)

	Source Type: winlogbeat

## 5. (Logstash) Output Statement

*note the usage of the /services/collector/raw endpoint; the event (vs raw) endpoint is unreliable when sending POST data from Logstash*

	output {
		http {
			content_type => "application/json"
			http_method => "post"
			url => "http://your-splunk-server:8088/services/collector/raw"
			headers => ["Authorization", "Splunk c6012558-7817-45e0-a3a5-7dfc876e1bf3"]
		}
	}

## Considerations
- use one token per index+sourcetype, and name/describe them accordingly
- harden and firewall your assets; do not allow the entire world, or even the whole corporate network, to reach the ingest ports
- certificates don't last forever; include the Splunk keys in your key management processes (and don't forget the Logstash stores)
- Logstash --> Kafka <-- Splunk: this setup is more durable and reduces data loss if the Splunk cluster goes down for some reason

## TO DO
- generate SSL/TLS certificates
- configure the Logstash truststore and keystore
