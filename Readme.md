
# with helm 3

$ helm install elasticsearch stable/elasticsearch 
wait for few minutes..

$ kubectl apply -f .\fluentd-daemonset-elasticsearch.yaml

$ helm install kibana stable/kibana -f kibana-values.yaml

$ kubectl apply -f .\counter.yaml

# Other
kubernetes.pod_name: counter

# old with helm 2
# Note: for some reason 7.6.0 doesn't work !!
$ helm install --name elasticsearch stable/elasticsearch --set master.persistence.enabled=false --set data.persistence.enabled=false --set image.tag=7.6.0

$ helm install --name fluentd stable/fluentd --set output.host=elasticsearch-client

$ helm install --name kibana stable/kibana -f kibana-values.yaml


# within logging namespace

$ kubectl create namespace logging

$ helm install --name elasticsearch stable/elasticsearch --namespace logging
--set master.persistence.enabled=false --set data.persistence.enabled=false

$ helm install --name fluentd stable/fluentd --namespace logging --set output.host=elasticsearch-client.logging.svc.cluster.local


$ helm install --name kibana stable/kibana -f kibana-values.yaml --namespace logging




$ kubectl exec elasticsearch-client-6b5b7b45d7-64zv6 -i -t curl "127.0.0.1:9200/_search?q=*&pretty"



  "data": {
    "containers.input.conf": "<source>\n   @id fluentd-containers.log\n   @type tail\n   path /var/log/containers/*.log\n   pos_file /var/log/es-containers.log.pos\n   tag raw.kubernetes.*\n   read_from_head true\n   <parse>\n     @type multi_format\n     <pattern>\n       format json\n       time_key time\n       time_format %Y-%m-%dT%H:%M:%S.%NZ\n     </pattern>\n     <pattern>\n       format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/\n       time_format %Y-%m-%dT%H:%M:%S.%N%:z\n     </pattern>\n   </parse>\n </source>\n # Detect exceptions in the log output and forward them as one log entry.\n <match raw.kubernetes.**>\n   @id raw.kubernetes\n   @type detect_exceptions\n   remove_tag_prefix raw\n   message log\n   stream stream\n   multiline_flush_interval 5\n   max_bytes 500000\n   max_lines 1000\n </match>\n # Concatenate multi-line logs\n <filter **>\n   @id filter_concat\n   @type concat\n   key message\n   multiline_end_regexp /\n$/\n   separator \"\"\n </filter>\n # Enriches records with Kubernetes metadata\n <filter kubernetes.**>\n   @id filter_kubernetes_metadata\n   @type kubernetes_metadata\n </filter>\n # Fixes json fields in Elasticsearch\n <filter kubernetes.**>\n   @id filter_parser\n   @type parser\n   key_name log\n   reserve_data true\n   remove_key_name_field true\n   <parse>\n     @type multi_format\n     <pattern>\n       format json\n     </pattern>\n     <pattern>\n       format none\n     </pattern>\n   </parse>\n </filter>",
    "forward-input.conf": "<source>\n  @type forward\n  port 24224\n  bind 0.0.0.0\n</source>",
    "general.conf": "# Prevent fluentd from handling records containing its own logs. Otherwise\n# it can lead to an infinite loop, when error in sending one message generates\n# another message which also fails to be sent and so on.\n#<match fluentd.**>\n  #@type null\n#</match>\n\n# Used for health checking\n<source>\n  @type http\n  port 9880\n  bind 0.0.0.0\n</source>\n\n# Emits internal metrics to every minute, and also exposes them on port\n# 24220. Useful for determining if an output plugin is retryring/erroring,\n# or determining the buffer queue length.\n<source>\n  @type monitor_agent\n  bind 0.0.0.0\n  port 24220\n  tag fluentd.monitor.metrics\n</source>",
    "output.conf": "<match **>\n  @id elasticsearch\n  @type elasticsearch\n  @log_level info\n  include_tag_key true\n  # Replace with the host/port to your Elasticsearch cluster.\n  host \"#{ENV['OUTPUT_HOST']}\"\n  port \"#{ENV['OUTPUT_PORT']}\"\n  scheme \"#{ENV['OUTPUT_SCHEME']}\"\n  ssl_version \"#{ENV['OUTPUT_SSL_VERSION']}\"\n  logstash_format true\n  <buffer>\n    @type file\n    path /var/log/fluentd-buffers/kubernetes.system.buffer\n    flush_mode interval\n    retry_type exponential_backoff\n    flush_thread_count 2\n    flush_interval 5s\n    retry_forever\n    retry_max_interval 30\n    chunk_limit_size \"#{ENV['OUTPUT_BUFFER_CHUNK_LIMIT']}\"\n    queue_limit_length \"#{ENV['OUTPUT_BUFFER_QUEUE_LIMIT']}\"\n    overflow_action block\n  </buffer>\n</match>",
    "system.conf": "<system>\n  root_dir /tmp/fluentd-buffers/\n</system>"
  }
