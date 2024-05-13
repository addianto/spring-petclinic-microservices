# Live Demo

To begin, run all containers:

```shell
docker compose up --detach
# In some Docker Compose installation, the command is: docker-compose up --detach
```

## Prometheus & Grafana

1. Open the Grafana dashboard at [`http://localhost:3000/dashboard`](http:/localhost:3000/dashboards).
2. Choose **Spring Petclinic Metrics** dashboard to open it.
3. You will see 7 widgets in the dashboard.
   Each widget displays visualisation based on a query written in PromQL to the Prometheus data source that is running within the private Docker network (i.e., `prometheus-server` running on port `9090` in the internal network).
4. Let's try looking at one widget and the corresponding PromQL query example.
   Click on the **HTTP Request Activity** widget title and choose **Edit** button.
5. You will see a bigger widget view, including the query builder panel at the bottom of the view.
6. There are two existing queries in the query builder. The first query is `sum(rate(http_server_requests_seconds_count[1m]))`. It comprises of several parts, such as:
    - `http_server_requests_seconds_count` --> a _counter metric_ in Prometheus. It represents the total count of HTTP requests your server gets. The number increases over time each time new requests come in.
    - `[1m]` --> a _time window_ or _range vector_. It specifies that we are interested in data or changes in the last 1 minute.
    - `rate()` --> a function in PromQL that calculate the _per-second average rate_ of time series in a range vector.
    - `sum()` --> a function in PromQL that adds up all the values produced by the wrapped function.
7. The latter query, i.e., `sum(rate(http_server_requests_seconds_count{status=~"5.."}[1m]))`, is similar. However, it adds a condition for filtering metrics based on label values. See the `{status=~"5.."}` part in the query? It means the query only take metrics where the `status` label has matching any HTTP status codes that start with `5` (e.g., `500`, `502`, etc.).

## Simulate a Load Test

Now, go back to the main dashboard view and keep the browser window open.
Let's try to simulate a load test and see how the Grafana visualises the activities.

The project includes a JMeter test plan that can be found inside the [`spring-petclinic-api-gateway` test code](./spring-petclinic-api-gateway/src/test/jmeter).
The test plan requires two JMeter plugins: Plugins Manager and Custom Thread Groups.
First, install the Plugins Manager plugin by following the instructions [here (click)](https://jmeter-plugins.org/wiki/PluginsManager/).
Once it has been installed, open JMeter GUI and go to the Plugins Manager menu.
From there, you can install the Custom Thread Groups plugin.

Run the test plan by loading it to the JMeter GUI or via `jmeter` command.
Let the test plan run for a while. It will take five minutes to finish.
While waiting, you can see the Grafana dashboard is being updated.

## Possible Issues

- Cannot start containers due to conflicting ports.
  Make sure all the published ports are not used by other running processes.
  You can change the published ports by modifying the `ports` section of a service in [`docker-compose.yml`](./docker-compose.yml) file.
