
## GSoC 2020 Proposal
## Organisation: CNCF(coredns)

## Student Info:

- **NAME:** Sumera Priyadarsini
- **GITHUB USERNAME:** [@Sylfrena](https://github.com/Sylfrena)
- **EMAIL:** sumerapriyadarsini@gmail.com, sylphrenadin@gmail.com
- **LOCATION:** Bangalore, India
- **TIME ZONE:** UTC+5:30

## PROJECT INFO

| `TITLE` | [**Anomaly detection of CoreDNS server through machine learning**](https://github.com/coredns/rfc/issues/2)| 
|---|---|
| `MENTORS` | [Yong Tang](https://github.com/yongtang) (@yongtang), [Paul Greenberg](https://github.com/greenpau) (@greenpau) |


## PROBLEM STATEMENT

- DNS servers are critical to devops infrastructure. Any anomalous/unexpected behaviour may lead to outage of an entire system. 
- Presently, CoreDNS only has a monitoring system where most alerting rules are crafted manually leading to slow response times.
- The goal of this project is to develop a machine learning based anomaly detection system for CoreDNS so as to provide an automated monitoring system, possibly with a pre-alert mechanism for pre-emptive action and post-alert corrective mechanism for mitigating the failure.

 
## IMPLEMENTATION

### 1. DNS DATA EXTRACTION 

- CoreDNS metrics can be easily exposed by using `Prometheus`. Querying these metrics on the prometheus server gives us various information about the requests currently being processed.

- Any anomalous behaviour in a system relying on CoreDNS will reflect in these metrics.

- Relevant metrics exposed to prometheus mostly fall in the following categories:

	1. **Network Connection Tracking queries**:
	
		- `net_conntrack_dialer_conn_attempted_total`
		- `net_conntrack_dialer_conn_closed_total`
		- `net_conntrack_dialer_conn_established_total`
		- `net_conntrack_dialer_conn_failed_total`
		- `net_conntrack_listener_conn_accepted_total`
		- `net_conntrack_listener_conn_closed_total`
	

	These queries are very significant as they can point us towards number of calls that were made,established or failed to the system. Usually an anomaly in calls will affect almost all the other kind of metrics as well.
	
	2. **System Metrics**
	
		- `process_cpu_seconds_total` 
		- `process_open_fds` 
		- `process_max_fds`
		- `process_resident_memory_bytes`
		- `process_start_time_seconds` 
		- `process_virtual_memory_max_bytes`
		- `process_virtual_memory_bytes`
		
	 - These metrics keep track of the server processes running on system resources such as number of open file descriptors, length of processes, memory consumption. Most outage incidents happen as a result of processes exceeding system's computation limits. 
	 - Therefore incident alerts must be designed so as to be sent based on behaviour of system metrics- especially if they are spiking collectively with other types of metrics.
	 
	3. **CoreDNS Metrics**
		
		- `coredns_build_info{version, revision, goversion}`: info about CoreDNS itself.
		- `coredns_panic_count_total{}`: total number of panics.
		- `coredns_dns_request_count_total{server, zone, proto, family}`: total query count.
		- `coredns_dns_request_duration_seconds{server, zone, type}`: duration to process each query.
		- `coredns_dns_request_size_bytes{server, zone, proto}`: size of the request in bytes.
		- `coredns_dns_request_do_count_total{server, zone}`: queries that have the DO bit set
		- `coredns_dns_request_type_count_total{server, zone, type}`: counter of queries per zone and type.
		- `coredns_dns_response_size_bytes{server, zone, proto}`: response size in bytes.
		- `coredns_dns_response_rcode_count_total{server, zone, rcode}`: response per zone and rcode.
		- `coredns_plugin_enabled{server, zone, name}`: indicates whether a plugin is enabled on per server and zone basis.

	- These metrics return the status of CoreDNS server itself as regards to requests and responses that are routed/resolved through the server.
	- They take input parameters regarding the server responsible for the request, the protocol of the response(tcp/udp), IP address family version number, type of query, and rcode of the response to return duration and size of response and requests.
	- Response duration and Request duration usually tend to gradually decrease before an anomaly, hence combined with other anomaly types, they are very useful metrics for pre-incident alerting.
	
- There are several other metrics that give information about `Prometheus API` itself and background `Go` routines. These metrics however are not listed here as they may not be extensively relevant to our use case.

- After extracting CoreDNS data from Prometheus API, once an anomalous event has been artificially simulated, or obtained otherwise, preprocessing needs to be done on the dataset. Dataset must be multivariate timeseries data collected a certain duration including the time before and after anomaly occured. 

- Correlation analysis must be done to evaluate intermetric relationships. Data Smoothening should be done to reduce noise as much as possible. Given that we are using multivariate data and trying to retain both seasonality and trends, triple exponentiation smoothening seems apt.

- Once preprocessing and statistical analysis of dataset is completed, we can move on to anomaly detection models.


### 2. PROPOSED ALGORITHM MODELS FOR ANOMALY DETECTION

The performance of machine learning algorithms is are highly sensitive to the nature of the input dataset- primary parameters here pertaining to the CoreDNS use-case include the following factors:

	- density/sparsity of dataset
	- univariate/multivariate dataset
	- timeseries aspect of dataset
	- non-labelled/labelled(structured) nature of dataset
	
A myriad number of contemporary and traditional algorithmic approaches are compared and proposed, taking into account the fact that the input dataset in not finalised, given that the data gathering phase has just begun. 

| Algorithms | Timeseries Input | Capture Seasonality/Trend | Unsupervised Data |
| --- | --- | --- | --- | 
| `XGBoost` |  | Not great |  |
| `Vanilla RNNs` | :heavy_check_mark: | Not so great |  |  
| `Autoencoder LSTM` | :heavy_check_mark: | Captures patterns | :heavy_check_mark: |
| `GRU` | :heavy_check_mark: | Captures patterns | :heavy_check_mark: | 
| `NuPIC HTM` | :heavy_check_mark: | Great | :heavy_check_mark: | 

However, some algorithms such as Autoencoder LSTMs, HTM stand out- both in terms of efficacy and their industrial usecases. We will explore these algorithms in depth.


### A. LSTM AUTOENCODERS

Recurrent Neural Networks, aka RNNs have major advantages over traditional algorithms- a key property of recurrent neeural networks is their ability to persist information, or their hidden cell state for later use in the model, giving rise to the concept of memory. These deep learning based neural networks are have the ability to _capture complex underlying patterns_, thus being able to incorporate significant time series characteristcs of **trend, seasonality** and autocorrelation within presentations.

However, one major caveat of RNNs is their limited capacities when it comes to **unsupervised data, and long term dependencies in the data**- both of which might be key characteristics of CoreDNS server metrics data.

A subtype of RNNs, Long Short Term Memory (LSTM) networks succesfully detect and store long term dependencies. They do this by using ***forget, input, and, output gates*** which control information flow by manipulating memory states.  LSTM models work in an unsupervised, semi-supervised manner.


	1. Autoencoder architecture essentially learns an identity function, takes input data, create representation of core driving features of data and then learn to develop it again.
	2. A repeat vector layer distributes the compressed representational vector across time steps of decoder.
	3. Final output layer of decoder provides us the reconstructed input data.
	4. Model is compiled using a neural network optimiser and mean square error is used for computing loss function.
	5. Model is trained and fitted with the multivariate distribution, anomaly scores are calculated based on the loss function distribution and a predefined confidence interval.
	6. If anomaly score for a point is above a certain predetermined score threshold, then the point is classified as an anomaly.
	

We explore two variants- both Variational Autoencoder LSTMS and Gated Recurrent Unit LSTMS.



#### VARIATIONAL AUTO ENCODER LSTMs
	- VAE LSTMs creates compessed probabilities as opposed to compressed features in simple autoencoder LSTMS
	- This is acheived by putting constraints on the encoding network which forces generation of latent vectors that follow a unit gaussian distribution.
	- In these models, there is a tight tradeoff between acuracy of the network and similarity of latent vectors with actual gaussian distribution.
	- Two kinds of losses- latent loss(aka KL divergence) and generative losses are taken into account and used in calculation of confidence intervals.
	
#### GATED RECURRENT UNITS

	- They are basically LSTMs(even though they are technically RNNs) with different gating.
	- GRUs have only two gates- reset and update as opppsed to three gates in LSTMs.
	- There is no memory unit- full hidden content is exposed any control. hence less computationally less expensive.
		
	- Major advanatage :  Less resource heavy and near equal efficiency as LSTM Autoencoders
	- Perform faster on less training data- which might add a definite advantage to our use case.
	- simpler, easier to modify as lesser number of gates are present.

#### DISADVANTAGES

- LSTM and GRUs work better in a contextual setting mainly with supervised data.

#### AVAILABILITY:

Autoencoder LSTMs, Variational Autoencoder LSTMs, and Gated Recurrent Units can be implemented directly from the `tf.keras` [Library](https://www.tensorflow.org/guide/keras).

### B. HIERARCHICAL TEMPORAL MEMORY 

- The hierarchical temporal memory algorithm has been developed by Numenta to mimic the human brain neocortex in order to facilitate anomaly detection.
- HTMs are well suited to work with unsupervised, multivariate, noisy data.
- HTMs do this by modeling spatial and temporal patterns in streaming data.
- Unlike most other methods, HTM continously learns temporal patterns, concurrently handlling noisy, unsupervised data.
- A typical HTM network focuses on sequence learning, continual learning, and even sparse representations.
- They emply a simple Hebbian model as opposed to traditional backpropagation, and are based on binary weights and units, rendering them much more efficient.
- A typical HTM network is a tree shaped hierarchy of levels- each comprising of several nodes. Higher the hierarchy, lesser the nodes. Higher hierarchy levels can reuse patterns learned at lower levels by combinig them to meorize more complex patterns.
- Each HTM node has same functions of:
	- `Learning and Inference Mode:` Sensory data comes into bottom level regions.
	- `Generation Mode:` Bottom level regions output generated pattern of a given category
- Top levels generally have a single region that stores most general(spatial) and permanent(temporal) categories/concepts.
- When set to inference mode, a region in each level interprets information coming up from its child regions as probabilities of categories it has in its memory.
- Each HTM region learns by identifying and memorizing spatial patterns—combinations of input bits that often occur at the same time. It then identifies temporal sequences of spatial patterns that are likely to occur one after another.

#### ADVANTAGES

	- higher computational efficiency due to lack of backpropagation
	- great at handling noise- useful feature for streaming data.
	- retain both spatial and temporal memory over a long time.

#### AVAILABILTY: 

HTM is available as part of the `nupic` package on [`PyPi`](https://pypi.org/project/nupic/)	
## END DELIVERABLES

- The model should be able to detect anomalies- especially focusing on reducing false postives as well as false negatives.
- The model needs to efficient at capturing trends and seasonalities and retaining the information in its memory as feedback so that false positives are reduced. 
- Model should be able to take in automatic feedback for contextual anomalies.
- Model must be able to take post-corrective action to mitigate an anomaly- like increase of requests in the zalando use case could have been followed by self alerting and autoincreasing the memory limit.

## POSSIBLE OBSTACLES

 - Sample dataset:
 	- No industry level sample dataset is available on open source channels for model validation purposes. Simulating a proper anomalous dataset will be tough as biases might creep in as a result of artificial contsruction of a dataset. 

	
## TIMELINE

### Community Bonding Phase (4th May - 1st June)

|Week #                                    | Tasks |
|---|---|
|Community Bonding Period | - Read up on DNS server anomalies and viable anomaly simulation techniques. Understand best practices for generating relevant input datsets.   |

### Coding Phase 1 (Starts 1st June)
|Week #                                    | Tasks |
|---|---|
|Community Bonding Period | - Read up on DNS server anomalies and viable anomaly simulation techniques. Understand best practices for generating relevant input datsets.   |
|Week  1 | - Start simluation of anomalous instances to generate relevant dataset   | 
|Week  2 | - Preprocess dataset- perform EDA, Perform clustering to see how metric groups relate to each other | 
|Week  3 | - Develop criteria and threshold for anomaly scores. Characterise anomalies mathematically  |  
|Week  4 | - Build initial model. Follow up with model tuning. Decide which model is more suitable. |  

### Coding Phase 2
|Week #                                    | Tasks |
|---|---|
|Week  5 | - Improve confidence interval and anomaly score functionalities.|  
|Week  6 | - Write tests. Package into a class. Debug|  
|Week  7 | - Develop workflow for Alerting mechanism |  
|Week  8 | - Write and incorporate alert modules into the class. This includes incident management and automated tooling |  

### Coding Phase 3 (Ends 24th August)
|Week #                                    | Tasks |
|---|---|
|Week  9 | - Write tests for alert modules. Debug. |  
|Week  10 | - Implement entire package. |  
|Week  11 | - Debug  |  
|Week  12 | - Finish documentation |  


## Why should I be selected to work on this project?

- I have significant past experience in developing machine learning models. So far, I have worked in Anomaly Detection involving alerts, root cause inference, collective anomalies, feedback modules - mostly(but not restricted to) using Unsupervised Clustering models, but only for closed source software.
- I am also quite proficient at pre-processing large amounts of raw data
- I have previously contributed to open source projects and I am well versed with software development best practices such as version control, unit testing, writing idiomatic code.
- Besides Python I also like writing code in other languages like Ruby, C, etc and love learning new languages so if I ever have to make upstream contributions to coredns I would love to learn Go for it.
- I want to take the oppurtunity to apply my knowledge to developing open source anomaly detection software.
