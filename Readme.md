# Unlocking Flexibility: A Practical Guide to Elastic Agent in Hybrid Mode

##  1. <a name='Tableofcontent'></a>Table of content
<!-- vscode-markdown-toc -->
* 1. [Table of content](#Tableofcontent)
* 2. [Overview](#Overview)
* 3. [Prerequisites](#Prerequisites)
* 4. [Test Scenario: Setting Up NGINX](#TestScenario:SettingUpNGINX)
	* 4.1. [Install NGINX](#InstallNGINX)
	* 4.2. [Activate NGINX Metrics](#ActivateNGINXMetrics)
* 5. [The Elastic Agent and EDOT](#TheElasticAgentandEDOT)
* 6. [Part 1: Collecting NGINX Logs with Fleet](#Part1:CollectingNGINXLogswithFleet)
	* 6.1. [Create an Agent Policy in Kibana](#CreateanAgentPolicyinKibana)
	* 6.2. [Enroll Your Agent](#EnrollYourAgent)
	* 6.3. [Add the NGINX Integration (Logs Only)](#AddtheNGINXIntegrationLogsOnly)
	* 6.4. [Validate Log Collection](#ValidateLogCollection)
* 7. [Part 2: Collecting NGINX Metrics with EDOT](#Part2:CollectingNGINXMetricswithEDOT)
	* 7.1. [Install the OTel Content Integration](#InstalltheOTelContentIntegration)
	* 7.2. [Test the OTel Collector (Standalone)](#TesttheOTelCollectorStandalone)
* 8. [ Part 3: Bringing all together in Elastic-agent Hybrid](#Part3:BringingalltogetherinElastic-agentHybrid)
	* 8.1. [Export Your Fleet Policy Configuration](#ExportYourFleetPolicyConfiguration)
	* 8.2. [Create the Standalone Hybrid `elastic-agent.yml`](#CreatetheStandaloneHybridelastic-agent.yml)
	* 8.3. [Monitoring](#Monitoring)
	* 8.4. [Restart the Agent](#RestarttheAgent)
	* 8.5. [Final Validation](#FinalValidation)
* 9. [Conclusion](#Conclusion)
* 10. [References](#References)

<!-- vscode-markdown-toc-config
	numbering=true
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->

##  2. <a name='Overview'></a>Overview

The Elastic Agent is Elastic single, unified collector for logs, metrics, traces, and security data. Its power lies in its flexibility. While many users love the simplicity of managing agents centrally with **Fleet**, there are times when you need more—perhaps to integrate custom OpenTelemetry (OTel) configurations or to gradually migrate existing OTel workloads without losing central management for other data sources.

This is where **hybrid mode** comes in.

Hybrid mode allows a single Elastic Agent to *simultaneously* run integrations managed by Fleet *and* a standalone Elastic Distribution of OpenTelemetry collector (EDOT) configuration.

In this guide, we'll walk through a practical example: collecting NGINX data from a single server using two different methods at the same time:

1.  **NGINX Logs:** Collected using the standard, Fleet-managed **NGINX Integration**.
2.  **NGINX Metrics:** Collected using the **Elastic Distribution of OpenTelemetry (EDOT)** collector, configured locally.

This approach gives you the best of both worlds: the rich, out-of-the-box experience of our standard integrations and the infinite flexibility of OpenTelemetry.

---

##  3. <a name='Prerequisites'></a>Prerequisites

This tutorial was tested using 
* Elastic Stack version 9.2.1
* Elastic Agent version 9.2.1

* Last Update: 13/11/2025

Before we begin, you'll need an Elastic Stack deployment (Elastic Cloud is the easiest way to start) and a Linux host where you can install NGINX and the Elastic Agent.

---

##  4. <a name='TestScenario:SettingUpNGINX'></a>Test Scenario: Setting Up NGINX

First, let's get our NGINX server running and configure it to expose the necessary log and metric data.

###  4.1. <a name='InstallNGINX'></a>Install NGINX

On your Linux host, run the following commands to install and start NGINX:

```bash
sudo apt -y install nginx
sudo systemctl enable nginx --now
```

Verify it's working by making a web request. You should see the default "Welcome to nginx!" page.

```bash
curl http://localhost
```
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```

###  4.2. <a name='ActivateNGINXMetrics'></a>Activate NGINX Metrics

We need to enable the `stub_status` module to expose basic server metrics.

1.  Edit your NGINX configuration file (e.g., `/etc/nginx/sites-available/default`).
2.  Inside the `server { ... }` block, add the following `location` snippet:

```nginx
    location = /status {
            stub_status;
    }
```

3.  Save the file and restart NGINX to apply the changes:

```bash
sudo systemctl restart nginx
```

4.  Verify the metrics endpoint is now active:

```bash
curl http://localhost/status

Active connections: 1 
server accepts handled requests
 2 2 2 
Reading: 0 Writing: 1 Waiting: 0 
```

With NGINX ready, let's set up the Elastic Agent.

---

##  5. <a name='TheElasticAgentandEDOT'></a>The Elastic Agent and EDOT

When you download the Elastic Agent, our OpenTelemetry distribution (EDOT) is already bundled inside. If you look at the unpacked agent directory, you'll see the OTel components:

```bash
$ ls ~/elastic-agent-9.2.1-linux-x86_64/
+-- elastic-agent
+-- elastic-agent.yml
+-- otel.yml
+-- otelcol
+-- otel_samples
...
```

* **otelcol:** The EDOT (Elastic Distribution of OpenTelemetry Collector) binary.
* **otel.yml:** A default, standalone OTel configuration file.
* **otel_samples:** A folder containing various example OTel configurations.

Our hybrid configuration will leverage this bundled `otelcol` binary, running it as part of the main `elastic-agent` service.

---

##  6. <a name='Part1:CollectingNGINXLogswithFleet'></a>Part 1: Collecting NGINX Logs with Fleet

First, we'll set up the "standard" collection path using a Fleet-managed policy.

###  6.1. <a name='CreateanAgentPolicyinKibana'></a>Create an Agent Policy in Kibana

1.  In Kibana, navigate to **Management > Fleet > Agent policies**.
2.  Click **Create agent policy**.
3.  Give it a name, such as **"NGINX-Log-Collection"**, and click **Create agent policy**.

###  6.2. <a name='EnrollYourAgent'></a>Enroll Your Agent

Now, let's install the agent on our NGINX server and enroll it in this new policy.

1.  From your new "NGINX-Log-Collection" policy page, click the **Add agent** button.
2.  Follow the on-screen instructions:
    * Select **Enroll in Fleet**.
    * Choose your OS (e.g., Linux) and architecture.
    * Copy the generated `sudo elastic-agent install ...` command.
3.  Run this command on your NGINX server. The agent will install as a service, enroll in Fleet, and start.
4.  Back in Kibana (under **Management > Fleet > Agents**), you'll see your new agent check in with a "Healthy" status.

###  6.3. <a name='AddtheNGINXIntegrationLogsOnly'></a>Add the NGINX Integration (Logs Only)

1.  In Kibana, go to **Management > Integrations**.
2.  Search for "NGINX" and click the main **NGINX** integration.
3.  Click **Add NGINX**.
4.  On the configuration page:
    * Give the integration a name (e.g., "nginx-logs").
    * Select your "NGINX-Log-Collection" policy.
    * **Crucially:** Scroll down and **uncheck the box for "Collect NGINX metrics"**. We'll handle metrics separately with OTel.
    * Leave the log paths as their defaults.
5.  Click **Save and continue**, then **Save and deploy changes**.

###  6.4. <a name='ValidateLogCollection'></a>Validate Log Collection

Wait a minute or two for the agent to receive the new policy. Generate some traffic (`curl http://localhost`) and then:

1.  In Kibana, go to **Analytics > Dashboards**.
2.  Search for and open the **[Logs NGINX] Webserver** dashboard.
3.  You should see your NGINX access logs populating the dashboard.



Great! Part one is complete. The Elastic Agent is now shipping logs, all managed via Fleet.

---

##  7. <a name='Part2:CollectingNGINXMetricswithEDOT'></a>Part 2: Collecting NGINX Metrics with EDOT

Now, let's add the second collection path—metrics—using the bundled OTel collector.

###  7.1. <a name='InstalltheOTelContentIntegration'></a>Install the OTel Content Integration

First, we need to install the Kibana assets that know how to visualize NGINX OTel metrics.

1.  In Kibana, go to **Management > Integrations**.
2.  Search for "NGINX OpenTelemetry" and click the **NGINX OpenTelemetry** integration.
3.  Click **Add NGINX OpenTelemetry**.
4.  You don't need to configure anything. Just click **Save and continue** to install the assets.

###  7.2. <a name='TesttheOTelCollectorStandalone'></a>Test the OTel Collector (Standalone)

Before we create the hybrid config, let's test the OTel collection by itself.

1.  **Create an OTel configuration file** on your NGINX server. Let's call it `~/nginx-otel-test.yml`:

    ```yaml
    receivers:
    nginx:
        endpoint: "http://localhost:80/status"
        collection_interval: 10s

    processors:
    resourcedetection:
        detectors: ["system", "ec2"]

    exporters:
    # Exporter to send logs and metrics to Elasticsearch
    elasticsearch/otel:
        endpoints: [ "{env:ELASTIC_URL}" ]
        api_key: {env:ELASTIC_API_KEY}
        mapping:
        mode: otel
    debug:
        verbosity: detailed

    service:
    pipelines:
        metrics:
        receivers: [nginx]
        processors: [resourcedetection]
        exporters: [debug, elasticsearch/otel]
    ```

    * **env:ELASTIC_URL:** Find this in your Elastic Cloud portal (APM & Tracing). It looks like `...apm.elastic-cloud.com`.
    * **env:ELASTIC_API_KEY:** Create one in Kibana under **Management > Dev Tools > API Keys**.

2.  **Run the collector in the foreground.** The `elastic-agent` binary has a special `otel` subcommand for this:

    ```bash
    # Run this from the agent's install directory
    sudo /opt/Elastic/Agent/elastic-agent otel --config ~/nginx-otel-test.yml
    ```

3.  You should see console output showing successful data batches being sent.

4.  **Validate Metrics Collection:**
    * In Kibana, go to **Analytics > Dashboards**.
    * Search for and open the **[Metrics NGINX OpenTelemetry] Overview** dashboard.
    * You should see metric data appearing (active connections, requests, etc.).



Once you've confirmed data is flowing, stop the foreground process (`Ctrl+C`).

---

##  8. <a name='Part3:BringingalltogetherinElastic-agentHybrid'></a> Part 3: Bringing all together in Elastic-agent Hybrid

> **Important:** When running the Elastic Agent in hybrid mode, you have to switch the managed agent to standalone. You will have to provide both the Agent configuration (which you can get from your Fleet policy) and the EDOT configuration we created earlier. This agent will **no longer be managed by Fleet**.

###  8.1. <a name='ExportYourFleetPolicyConfiguration'></a>Export Your Fleet Policy Configuration

First, we need the configuration for the NGINX *logs* that Fleet was managing.

1.  In Kibana, go to **Management > Fleet > Agent policies**.
2.  Find your **"NGINX-Log-Collection"** policy and click on it.
3.  Click the **"Actions"** button (top right) and select **"Copy policy"**. This will show you the raw YAML configuration.
4.  Copy the YAML. It will look something like this:

    ```yaml
      - id: 671146b6-fb31-4981-b60e-12a44606d200
        name: nginx-logs
        type: nginx
        use_output: default
        data_stream:
          namespace: default
        streams:
          - id: nginx-access-abc...
            data_stream:
              dataset: nginx.access
            # ... other log settings ...
          - id: nginx-error-abc...
            data_stream:
              dataset: nginx.error
            # ... other log settings ...
    ```

###  8.2. <a name='CreatetheStandaloneHybridelastic-agent.yml'></a>Create the Standalone Hybrid `elastic-agent.yml`

Now, we will completely replace the `elastic-agent.yml` file with our new standalone config.

1.  Open the main agent configuration file:

    ```bash
    sudo vim /opt/Elastic/Agent/elastic-agent.yml
    ```

2.  **Delete all content** in this file and replace it with the following structure. This combines your Elasticsearch output, the `inputs` (for logs) from your policy, and the `otel` block (for metrics).

    ```yaml
    # 1. ADD YOUR OUTPUTS (e.g., Elasticsearch)
    # This tells the agent where to send ALL data
    outputs:
      default:
        type: elasticsearch
        hosts:
          - 'env:ELASTIC_URL'
        api_key: 'env:ELASTIC_API_KEY'
    
    # 2. ADD THE 'INPUTS' FROM YOUR FLEET POLICY
    # This is for NGINX logs
    inputs:
      - id: nginx-logs-nginx-0123abc
        name: nginx-logs
        type: nginx
        use_output: default
        data_stream:
          namespace: default
        streams:
          - id: nginx-access-abc...
            data_stream:
              dataset: nginx.access
            paths:
              - /var/log/nginx/access.log*
            vars:
              - name: paths
                value:
                  - /var/log/nginx/access.log*
          - id: nginx-error-abc...
            data_stream:
              dataset: nginx.error
            paths:
              - /var/log/nginx/error.log*
            vars:
              - name: paths
                value:
                  - /var/log/nginx/error.log*
    
    # 3. ADD THE 'OTEL' BLOCK FOR METRICS
    # This is for NGINX metrics
    receivers:
    nginx:
        endpoint: "http://localhost:80/status"
        collection_interval: 10s

    processors:
    resourcedetection:
        detectors: ["system", "ec2"]

    exporters:
    # Exporter to send logs and metrics to Elasticsearch
    elasticsearch/otel:
        endpoints: [ "{env:ELASTIC_URL}" ]
        api_key: {env:ELASTIC_API_KEY}
        mapping:
        mode: otel
    debug:
        verbosity: detailed

    service:
    pipelines:
        metrics:
        receivers: [nginx]
        processors: [resourcedetection]
        exporters: [debug, elasticsearch/otel]
    ```

    * **env:ELASTIC_URL:** This is your Elasticsearch connection info. You can create a new API key in Kibana with agent privileges.
    * **env:ELASTIC_API_KEY:** This is your Elasticsearch API Key


###  8.3. <a name='Monitoring'></a>Monitoring

Optionally, you can configure the monitoring for both Elastic-Agent and EDOT by adding to the previously generated `elastic-agent.yml` file the following content just below the NGINX Metrics data collection from EDOT.

    ```yaml
    # 3. ADD BELOW THE 'OTEL' BLOCK FOR METRICS THE CONTENT
    ## Monitoring

    ### Elastic-agent 
    agent.monitoring:
      # enabled turns on monitoring of running processes
      enabled: true
      # enables log monitoring
      logs: true
      # enables metrics monitoring
      metrics: true
      # exposes /debug/pprof/ endpoints for Elastic Agent and Beats
      # enable these endpoints if the monitoring endpoint is set to localhost
      pprof.enabled: false
      # specifies output to be used
      use_output: default
      http:
        # exposes a /buffer endpoint that holds a history of recent metrics
        buffer.enabled: false

    ### EDOT
    #### configuration for the otel collector managed by elastic-agent
    agent.collector:
      telemetry:
        # Endpoint on which the otel collector will expose its Prometheus metrics. Default is localhost with a random port. (curl http://localhost:8181/metrics)
        endpoint: http://127.0.0.1:8181
      healthcheck:
        # Endpoint on which the otel collector will expose its status. Default is localhost with a random port. (curl http://localhost:8182/health/status)
        endpoint: http://127.0.0.1:8182
    ```

###  8.4. <a name='RestarttheAgent'></a>Restart the Agent

Save the file and restart the `elastic-agent` service to apply your new standalone configuration.

```bash
sudo systemctl restart elastic-agent
```

###  8.5. <a name='FinalValidation'></a>Final Validation

You've done it! The single `elastic-agent` service is now running in standalone hybrid mode.

Go back to your two Kibana dashboards:
* **[Logs NGINX] Webserver:** You'll see new logs arriving, now collected by the `inputs:` section of your local `elastic-agent.yml`.
* **[Metrics NGINX OpenTelemetry] Overview:** You'll see new metrics arriving, collected by the `otel:` section of your local `elastic-agent.yml`.

In Fleet, this agent will now show as "Offline" or "Unenrolled," which is expected, as it is no longer managed by Fleet.

---

##  9. <a name='Conclusion'></a>Conclusion

You now have a single Elastic Agent efficiently collecting NGINX logs via a standalone `inputs` configuration and NGINX metrics via a local `otel:` configuration.

This standalone hybrid pattern is incredibly powerful. You can use it to:
* Take full, local control over agents that have special requirements.
* Combine standard Elastic integrations with custom OpenTelemetry receivers, all in one configuration file.
* Gradually migrate existing OTel workloads to Elastic.

##  10. <a name='References'></a>References
 
1. [Github Documentation](https://github.com/elastic/elastic-agent/blob/main/docs/hybrid-agent-beats-receivers.md)
2. [EDOT Collector](https://www.elastic.co/docs/reference/edot-collector)
3. [Elastic-agent managed demo configuration sample](./elastic-agent-managed.yml)
4. [EDOT demo configuration sample](./otel.yml)
5. [Elastic-agent standalone hybrid demo configuration sample](./elastic-agent-hybrid.yml)