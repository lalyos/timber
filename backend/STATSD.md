this exercise describes how to easily add metrics with graph to a python/flask app

## Flask runner

By default you are running your flask app ass `python app.py`. In this case you are using the built-in
wsgi (Web Server Gateway Interface) implementatioj. But the are more production redy implementations. see options: http://flask.pocoo.org/docs/1.0/deploying/

## Gunicorn

For example one widely used standelone wsgi is: [gunicorn](http://docs.gunicorn.org).
To install use pip: `pip install gunicorn` or put it into your `requirements.txt`

Starting the server:
```
gunicorn -w 4 -b 127.0.0.1:4000 myproject:app
```

There are a lot of configuration options for gunicornl like [logging](http://docs.gunicorn.org/en/latest/settings.html#accesslog) or [debugging](http://docs.gunicorn.org/en/latest/settings.html#debugging).

## Monitoring

One interesting aspect is [instrumentation](http://docs.gunicorn.org/en/latest/instrumentation.html).
Gunicorn can report metrics like requests/200resp/404resp to a statsd server.

## statsd

What is a [statsd](https://github.com/statsd/statsd) server? Its a metrics collector listening normally on *UDP 8125*.
which emits simple text messages in the form of:
```
<metricname>:<value>|<type>
```

So sending a couple of example random metric is easy as:
```
echo "foo:1|c" | nc -u -w0 127.0.0.1 8125
```

## statsd + graphite

But statsd itself only collects metrics, how to display them? [Graphite](https://graphiteapp.org) is a general purpose enterprise-ready monitoring tool, which can use statsd as data source.

Fortunately there is ready to use ststd+graphite docker image: https://hub.docker.com/r/graphiteapp/docker-graphite-statsd.

To deploy it in your k8s namespace as a deployment and NodePort service, use the yaml below.
After that you can just refer to the stats server with the simple DNS name `graphite`.

## Gunicorn + Graphite FTW

So to put together all the pieces. We have to configure gunicorn to send metrics to statsd:
```
gunicorn -b 0.0.0.0:$TIMBER_LISTEN_PORT app:app  --statsd-host=graphite:8125 --statsd-prefix=timber-backend
```

## Downward API

Instead of hardcoding the statsd prefix, lets use the **pod name**.
Read [Pod Information to Containers Through Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)

So add a new env variable to the deployment:
```
        - name: MY_POD_NAME           
          valueFrom:  
            fieldRef:                               
              apiVersion: v1                  
              fieldPath: metadata.name    
```

To use the previous env var as a gunicorn option, we use the `$(OTHER_ENV)` notation.
```
        - name: GUNICORN_ARGS                    
          value: --statsd-host=graphite:8125 --statsd-prefix=$(MY_POD_NAME)
```
## Patching the deployment

Changing the deployment on the fly:
```
kubectl patch deployment backend --patch '{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"backend"},{"name":"log-sidecar"}],"containers":[{"$setElementOrder/env":[{"name":"TIMBER_ACCESS_LOG"},{"name":"XXX"},{"name":"MY_POD_NAME"},{"name":"GUNICORN_ARGS"}],"env":[{"name":"MY_POD_NAME","valueFrom":{"fieldRef":{"fieldPath":"metadata.name"}}},{"name":"GUNICORN_ARGS","value":"--statsd-host=graphite:8125 --statsd-prefix=$(MY_POD_NAME)"}],"name":"backend"}]}}}}'

```

## Deploy stats and graphite
```
$ kubectl apply -f - <<EOF>>
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: graphite
  name: graphite
spec:
  replicas: 1
  selector:
    matchLabels:
      run: graphite
  strategy: {}
  template:
    metadata:
      labels:
        run: graphite
    spec:
      containers:
      - image: graphiteapp/graphite-statsd
        name: graphite
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 8125
          name: statsd
          protocol: UDP
        resources: {}
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: graphite
  name: graphite
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  - name: statsd
    port: 8125
    protocol: UDP
    targetPort: 8125
  selector:
    run: graphite
  type: NodePort
EOF
```