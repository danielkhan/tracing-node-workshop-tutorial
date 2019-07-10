This repository contains material for my workshop Tracing Node.

# Setting up all services
1. Clone the repository `git clone git@github.com:danielkhan/tracing-node-workshop.git`.
2. Copy `/.env-sample` to `/.env`.
3. Fill in the information provided.
4. Create 4 terminal windows.
5. In each window, change into a service and run `npm install` followed by `npm start`.
6. Open `http://lcoalhost:8080` to try it out.

# Collecting metrics

## Server setup

To collect metrics, we need a server that contains services for receiving metrics
as well as some user interface.
For this workshop, this server exists already.

This provides the following services for our metrics monitoring solution:
- Telegraf: A dameon that can collect different kind of data and provides a statsd endpoint
- InfluxDB: A timeseries database
- Grafana: A user interface that lets us create dashboards from different datasources

To access Grafana, open our tracing host on port 3000.
The credentials will be provided during the workshop.

I have already configured an InfluxDB datasource in Grafana:
![InfluxDB Datasource in Grafana](./assets/influxdb-datasource-grafana.png)

Of course we don't collect any metrics yet. We will set this up now.

## Sending metrics from Node.js
Next we want to configure our services so that they collect metrics and send it
to the server.

The `monitoring` project will be our central location for all monitoring related code.

We will install a module that povides us Node.js metricsout-of-the-box:

In `/monitoring`we run:
`npm install -S appmetrics-statsd`

This will install a native module.
If the install fails and you are on windows, make sure that you have the required build tools available.
To install the build tools, run `npm install --global windows-build-tools`.

Next open `/montoring/index.js` and add

```js
module.exports.metrics = (host, prefix) => {
  const statsd = require('appmetrics-statsd').StatsD(
    { host, prefix }
  );
  return {
    statsd,
  }
};
```

This will set up statsd for us and start with metrics collection right away.

Of course, we have to add this module to our services.
So we open `app.js` in every service and add after `const serviceName ...`:

```js
const monitoring = require('../monitoring');
const metrics = monitoring.metrics(process.env.COLLECTOR, `${process.env.MY_HANDLE}_${serviceName}_`);
```

Hit `http://localhost:8080` a few times.

In Grafana, all your metrics will be prefixed with your handle.

## Creating a dashboard in Grafana
Follow the instructions during the workshop.
![Grafana Event Loop Dashboard](./assets/grafana-eventloop-dashboard.png)

## Define Alerts
Follow the instructions during the workshop.
![Grafana Create Alert](./assets/grafana-add-alert.png)

## Task: Create a memory monitoring dashboard Dashboard
Replicate what we've learned so far to create a dashboard that gives us
memory_process_virtual.

You might see, that the values on the Y-axis are not very user friendly.
Follow the instructions in the workshop to fix this.

Find all metrics descriptions here: https://github.com/RuntimeTools/appmetrics-statsd.

## Create a metrics collection middleware
We also want to get a bit more information, like status codes drectly from express.
Unfortunately, `appmetrics-statsd´ won't give us that.

Let's create a middleware for this.

For that we go to `/monitoring/index.js` and add:

```js
const middleware = (req, res, next) => {
  var startTime = new Date().getTime();
  // Function called on response finish that sends stats to statsd
  function sendStats() {
    var key = 'http-express-';
    // Status Code
    var statusCode = res.statusCode || 'unknown_status';
    statsd.increment(key + 'status_code.' + statusCode);
    // Response Time
    var duration = new Date().getTime() - startTime;
    statsd.timing(key + 'response_time', duration);
    cleanup();
  }
  // Function to clean up the listeners we've added
  function cleanup() {
    res.removeListener('finish', sendStats);
    res.removeListener('error', cleanup);
    res.removeListener('close', cleanup);
  }
  // Add response listeners
  res.once('finish', sendStats);
  res.once('error', cleanup);
  res.once('close', cleanup);
  if (next) {
    next();
  }
}
```

Finally, we add this middleware to our services:

```js
const app = express();
app.use(metrics.middleware);
```
Restart the services and hit `http://localhost:8080` a few times again.

## Create a counter dashboard for error 500
Follow the instructions in the workshop.
The metric to look for is `<your_handle>_service-gateway_http-express`.

# Install Jaeger Tracing

```bash
$ docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.13
```

Frontend `http://tracing.khan.io:16686`.

![Jaeger Architecture](./assets/jaeger-architecture.png)

# Install Opencensus Node

`npm install @opencensus/nodejs --save`

express-frontend: `app.js`
const tracing = require('@opencensus/nodejs');
tracing.start();

Hit `http://localhost:8080` a few times.


Install `npm install @opencensus/exporter-jaeger -S`

```js
// bin/www
const tracing = require('@opencensus/nodejs');
const JaegerTraceExporter = require('@opencensus/exporter-jaeger').JaegerTraceExporter;

const options = {
  serviceName: 'express-frontend',
  host: process.env.COLLECTOR,
}
const exporter = new JaegerTraceExporter(options);
tracing.start({ exporter });
```

Hit the webpage

Open `http://tracing.khan.io:16686`.

# Propagation

Add console.log(req.headers) to middleware in service-gateway

`npm install @opencensus/propagation-tracecontext -S`

const propagation = require('@opencensus/propagation-tracecontext');
const traceContext = new propagation.TraceContextFormat();

tracing.start({ exporter, propagation: traceContext });

Compare