# Performance analysis of Prometheus on k8s

## Questions require answering

1. How many samples can I manage with 2 cores and 6 GBs or ram?
2. How much metric size and cardinality affect the required resources?
3. Is it better to let Prometheus scrape or use a collector to remote write to Prometheus?
4. How the performance of thanos is affected when we have multiple kubernetes deployments?

## Experiment setup

We'll use an application called avalance to generate random prometheus metrics.  
`~/go/bin/avalanche --gauge-metric-count=100 --counter-metric-count=100 --histogram-metric-count=100 --port=9001`

Prometheus will be setup with the following config.  
```
resources:
    `limits:
    cpu: 3000m
    memory: 7Gi
    requests:
    cpu: 2000m
    memory: 6Gi
args:
    - --config.file=/etc/config/prometheus.yml
    - --storage.tsdb.path=/data
    - --web.console.libraries=/etc/prometheus/console_libraries
    - --web.console.templates=/etc/prometheus/consoles
    - --web.enable-lifecycle
    - --web.enable-remote-write-receiver
    - --storage.tsdb.retention.time=15m`
```

The experiment should be reproducible. Using docker deskop with k8s enalbed the reader should be able to apply the following yaml files and validate the numbers.  
```
kubectl apply -f avalance.yaml -f prom.yaml -f alloy.yaml
```

## The experiment

### Attempt 1 - Raw k8s config with all the basic relabels 

1. Deploy prom.yaml and avalance.yaml
2. Wait for 10mins to have data
3. Use baton to performance test the queries

![Alt text](images/plain-prom.png)

#### Results

With 10 avalance pods we gathered ~400k samples  
RAM was about to be maxed out at 5.2GBs   
Response times are good!  
```
argy@Argyrioss-MacBook-Pro ~ % ~/go/bin/baton -c 100 -r 50 -u "http://localhost:30000/api/v1/query?query=sum(scrape_samples_scraped)"
Configuring to send GET requests to: http://localhost:30000/api/v1/query?query=sum(scrape_samples_scraped)
Generating the requests...
Finished generating the requests
Sending the requests to the server...
Finished sending the requests
Processing the results...

=========================== Results ========================================

Total requests:                                    50
Time taken to complete requests:          20.032375ms
Requests per second:                             2496
Max response time (ms):                             0
Min response time (ms):                    9223372036854775807
Avg response time (ms):                           NaN

========= Percentage of responses by status code ==========================

Number of connection errors:                        0
Number of 1xx responses:                            0
Number of 2xx responses:                           50
Number of 3xx responses:                            0
Number of 4xx responses:                            0
Number of 5xx responses:                            0
```

### Attempt 2 - Raw k8s config with all the basic relabels 

1. Update Deploy prom.yaml with label drop
```
metric_relabel_configs:
- action: labeldrop
  regex: (kubernetes_io.*|beta_kubernetes_io.*|app_kubernetes_io.*|helm_sh_chart|instance|node|uid|pod_template_hash|series_id|cycle_id)
```
2. Deploy prom.yaml and avalance.yaml
3. Wait for 10mins to have data
4. Use baton to performance test the queries

![Alt text](images/labeldrop-prom.png)

#### Results

With 10 avalance pods we gathered ~400k samples  
RAM was reasonable at 1.2GBs  
Response times are still good!    
```
argy@Argyrioss-MacBook-Pro ~ % ~/go/bin/baton -c 100 -r 50 -u "http://localhost:30000/api/v1/query?query=sum(scrape_samples_scraped)"
Configuring to send GET requests to: http://localhost:30000/api/v1/query?query=sum(scrape_samples_scraped)
Generating the requests...
Finished generating the requests
Sending the requests to the server...
Finished sending the requests
Processing the results...

=========================== Results ========================================

Total requests:                                    50
Time taken to complete requests:          15.469292ms
Requests per second:                             3232
Max response time (ms):                             0
Min response time (ms):                    9223372036854775807
Avg response time (ms):                           NaN

========= Percentage of responses by status code ==========================

Number of connection errors:                        0
Number of 1xx responses:                            0
Number of 2xx responses:                           50
Number of 3xx responses:                            0
Number of 4xx responses:                            0
Number of 5xx responses:                            0
```

### Attempt 3 - Use grafana alloy to scrape and push metrics to the prometheus remote write

1. Configure config.alloy to the exact prom configs used previously
2. Comment out all scrape configs of prometheus
3. Deploy the avalance.yaml, prom.yaml and alloy.yaml
4. Wait for 10mins to have data
5. Use baton to performance test the queries

#### Results

With 10 avalance pods we gathered ~400k samples  
RAM was exploding up to 7GBs cause OOM errors on Prometheus! Why?  
Query performance when prometheus was active was still good  
```
argy@Argyrioss-MacBook-Pro ~ % ~/go/bin/baton -c 100 -r 50 -u "http://localhost:30000/api/v1/query?query=sum(scrape_samples_scraped)"
Configuring to send GET requests to: http://localhost:30000/api/v1/query?query=sum(scrape_samples_scraped)
Generating the requests...
Finished generating the requests
Sending the requests to the server...
Finished sending the requests
Processing the results...

=========================== Results ========================================

Total requests:                                    50
Time taken to complete requests:          30.475167ms
Requests per second:                             1641
Max response time (ms):                             0
Min response time (ms):                    9223372036854775807
Avg response time (ms):                           NaN

========= Percentage of responses by status code ==========================

Number of connection errors:                        0
Number of 1xx responses:                            0
Number of 2xx responses:                           50
Number of 3xx responses:                            0
Number of 4xx responses:                            0
Number of 5xx responses:                            0
```

### High level observations

- Remote write is causing higher RAM utilisation on the prometheus side
- Grafana alloy wont give us any performance improvement
- Query response times when data is in memory is still good 
- Cardinality matters a lot!

## In depth search