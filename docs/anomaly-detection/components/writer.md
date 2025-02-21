---
# sort: 4
title: Writer
weight: 4
menu:
  docs:
    parent: "vmanomaly-components"
    weight: 4
    # sort: 4
aliases:
  - /anomaly-detection/components/writer.html
---

# Writer
<!--
There are 3 ways to export data from VictoriaMetrics Anomaly Detection: VictoriaMetrics, JSON file, or CSV file. Depending on the chosen option, different parameters should be specified in the config file in the `writer` section.
-->

For exporting data, VictoriaMetrics Anomaly Detection (`vmanomaly`) primarily employs the [VmWriter](#vm-writer), which writes produces anomaly scores (preserving initial labelset and optionally applying additional ones) back to VictoriaMetrics. This writer is tailored for smooth data export within the VictoriaMetrics ecosystem. 

Future updates will introduce additional export methods, offering users more flexibility in data handling and integration.

## VM writer

### Config parameters

<table>
    <thead>
        <tr>
            <th>Parameter</th>
            <th>Example</th>
            <th>Description</th>  
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><code>class</code></td>
            <td><code>"writer.vm.VmWriter"</code></td>
            <td>Name of the class needed to enable writing to VictoriaMetrics or Prometheus. VmWriter is the default option, if not specified.</td>
        </tr>
        <tr>
            <td><code>datasource_url</code></td>
            <td><code>"http://localhost:8481/"</code></td>
            <td>Datasource URL address</td>
        </tr>
        <tr>
            <td><code>tenant_id</code></td>
            <td><code>"0:0"</code></td>
            <td>For cluster version only, tenants are identified by accountID or accountID:projectID</td>
        </tr>
        <!-- Additional rows for metric_format -->
        <tr>
            <td rowspan="4"><code>metric_format</code></td>
            <td><code>__name__: "vmanomaly_$VAR"</code></td>
            <td rowspan="4">Metrics to save the output (in metric names or labels). Must have <code>__name__</code> key. Must have a value with <code>$VAR</code> placeholder in it to distinguish between resulting metrics. Supported placeholders:
                <ul>
                    <li><code>$VAR</code> -- Variables that model provides, all models provide the following set: {"anomaly_score", "y", "yhat", "yhat_lower", "yhat_upper"}. Description of standard output is <a href="/anomaly-detection/components/models/models.html#vmanomaly-output">here</a>. Depending on <a href="/anomaly-detection/components/models/models.html">model type</a> it can provide more metrics, like "trend", "seasonality" etc.</li>
                    <li><code>$QUERY_KEY</code> -- E.g. "ingestion_rate".</li>
                </ul>
                Other keys are supposed to be configured by the user to help identify generated metrics, e.g., specific config file name etc.
                More details on metric formatting are <a href="#metrics-formatting">here</a>.
            </td>
        </tr>
        <tr><td><code>for: "$QUERY_KEY"</code></td></tr>
        <tr><td><code>run: "test_metric_format"</code></td></tr>
        <tr><td><code>config: "io_vm_single.yaml"</code></td></tr>  
        <!-- End of additional rows -->
        <tr>
            <td><code>import_json_path</code></td>
            <td><code>"/api/v1/import"</code></td>
            <td>Optional, to override the default import path</td>
        </tr>
        <tr>
            <td><code>health_path</code></td>
            <td><code>"health"</code></td>
            <td>Absolute or relative URL address where to check the availability of the datasource. Optional, to override the default <code>"/health"</code> path.</td>
        </tr>
        <tr>
            <td><code>user</code></td>
            <td><code>"USERNAME"</code></td>
            <td>BasicAuth username</td>
        </tr>
        <tr>
            <td><code>password</code></td>
            <td><code>"PASSWORD"</code></td>
            <td>BasicAuth password</td>
        </tr>
        <tr>
            <td><code>timeout</code></td>
            <td><code>"5s"</code></td>
            <td>Timeout for the requests, passed as a string. Defaults to "5s"</td>
        </tr>
        <tr>
            <td><code>verify_tls</code></td>
            <td><code>"false"</code></td>
            <td>Allows disabling TLS verification of the remote certificate.</td>
        </tr>
        <tr>
            <td><code>bearer_token</code></td>
            <td><code>"token"</code></td>
            <td>Token is passed in the standard format with the header: "Authorization: bearer {token}"</td>
        </tr>
    </tbody>
</table>

Config example:
```yaml
writer:
  class: "writer.vm.VmWriter"
  datasource_url: "http://localhost:8428/"
  tenant_id: "0:0"
  metric_format:
    __name__: "vmanomaly_$VAR"
    for: "$QUERY_KEY"
    run: "test_metric_format"
    config: "io_vm_single.yaml"
  import_json_path: "/api/v1/import"
  health_path: "health"
  user: "foo"
  password: "bar"
```

### Healthcheck metrics

`VmWriter` exposes [several healthchecks metrics](./monitoring.html#writer-behaviour-metrics). 

### Metrics formatting
There should be 2 mandatory parameters set in `metric_format` - `__name__` and `for`. 
```yaml
__name__: PREFIX1_$VAR
for: PREFIX2_$QUERY_KEY
```
* for `__name__` parameter it will name metrics returned by models as `PREFIX1_anomaly_score`, `PREFIX1_yhat_lower`, etc. Vmanomaly output metrics names described [here](anomaly-detection/components/models/models.html#vmanomaly-output)

* for `for` parameter will add labels `PREFIX2_query_name_1`, `PREFIX2_query_name_2`, etc. Query names are set as aliases in config `reader` section in [`queries`](anomaly-detection/components/reader.html#config-parameters) parameter.

It is possible to specify other custom label names needed.
For example:
```yaml
custom_label_1: label_name_1
custom_label_2: label_name_2
```

Apart from specified labels, output metrics will return labels inherited from input metrics returned by [queries](/anomaly-detection/components/reader.html#config-parameters).
For example if input data contains labels such as `cpu=1, device=eth0, instance=node-exporter:9100` all these labels will be present in vmanomaly output metrics.

So if metric_format section was set up like this:
```yaml
metric_format:
    __name__: "PREFIX1_$VAR"
    for: "PREFIX2_$QUERY_KEY"
    custom_label_1: label_name_1
    custom_label_2: label_name_2
```

It will return metrics that will look like:
```yaml
{__name__="PREFIX1_anomaly_score", for="PREFIX2_query_name_1", custom_label_1="label_name_1", custom_label_2="label_name_2", cpu=1, device="eth0", instance="node-exporter:9100"}
{__name__="PREFIX1_yhat_lower", for="PREFIX2_query_name_1", custom_label_1="label_name_1", custom_label_2="label_name_2", cpu=1, device="eth0", instance="node-exporter:9100"}
{__name__="PREFIX1_anomaly_score", for="PREFIX2_query_name_2", custom_label_1="label_name_1", custom_label_2="label_name_2", cpu=1, device="eth0", instance="node-exporter:9100"}
{__name__="PREFIX1_yhat_lower", for="PREFIX2_query_name_2", custom_label_1="label_name_1", custom_label_2="label_name_2", cpu=1, device="eth0", instance="node-exporter:9100"}
```

<!--
# TODO: uncomment and maintain after multimodel config refactor, 2nd priority


## NDJSON writer
Generates data in the same format as <code>/export</code>. 

### Config parameters
<table>
    <thead>
        <tr>
            <th>Parameter</th>
            <th>Example</th>
            <th>Description</th>  
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><code>class</code></td>
            <td><code>"writer.ndjson.NdjsonWriter"</code></td>
            <td>Name of the class needed to enable writing into JSON line format file.</td>
        </tr>
        <tr>
            <td><code>path</code></td>
            <td><code>"data/output.ndjson"</code></td>
            <td>Path to file in JSON line format</td>
        </tr>
        <tr>
            <td><code>metric_format</code></td>
            <td><code>__name__: "vmanomaly_$VAR"</code></td>
            <td>Metrics to save the output (in metric names or labels). Must have <code>__name__</code> key. Must have a value with <code>$VAR</code> placeholder in it to distinguish between resulting metrics. Supported placeholders: <code>$VAR</code>, <code>$QUERY_KEY</code> and others as configured by the user.</td>
        </tr>
        <tr>
            <td><code>override</code></td>
            <td><code>True</code></td>
            <td>Override file flag. Default True</td>
        </tr>
    </tbody>
</table>

Config example:
```yaml
writer:
  class: "writer.ndjson.NdjsonWriter"
  path: 'data/output.ndjson'
  metric_format:
    __name__: "$VAR"
    for: "$QUERY_KEY"
    config: "io_ndjson.yaml"
```


## CSV writer

### Config parameters
<table>
    <thead>
        <tr>
            <th>Parameter</th>
            <th>Type</th>
            <th>Example</th>
            <th>Description</th>  
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><code>class</code></td>
            <td>str</td>
            <td><code>"writer.csv.CsvWriter"</code></td>
            <td>Name of the class enabling CSV writer</td>
        </tr>
        <tr>
            <td><code>header</code></td>
            <td>bool</td>
            <td><code>True</code></td>
            <td>Whether to write header (column names). Default True</td>
        </tr>
        <tr>
            <td><code>path</code></td>
            <td>str</td>
            <td><code>"data/jumpsup.csv"</code></td>
            <td>Where to save the results</td>
        </tr>
        <tr>
            <td><code>override</code></td>
            <td>bool</td>
            <td><code>True</code></td>
            <td>Override file flag. Default True</td>
        </tr>
        <tr>
            <td><code>tz</code></td>
            <td>str</td>
            <td><code>None</code></td>
            <td>Optional. Convert default timestamps in UTC to desired timezone, e.g. 'US/Pacific'. By default local timezone is used</td>
        </tr>
    </tbody>
</table>
Config example:

```yaml
writer:
  class: "writer.csv.CsvWriter"
  path: "data/io_csv_out.csv"
  header: True
  override: True
```
-->