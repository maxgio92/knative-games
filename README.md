# [Knative](https://knative.dev) PoC: crawling via VPN

This is a simple scenario to demonstrate event-driven and serverless architectures, leveraging Knative serving and eventing capabilities.

### The scenario

In this scenario a crawler app needs to crawl data through a VPN tunnel, which needs to be provisioned.

Crawling data requests are produced as [CloudEvents](https://cloudevents.io/).

The crawler app will consume pending crawling requests, and will trigger a request for a VPN tunnel to satisfy the request.

In turn, a VPN provisioner app, listening for VPN-related events will satisfy the requests, and will response in turn with an appropriate event message.

Finally, the crawler application will satisfy the crawling request as soon as the VPN will be ready, and it will notify the completion.

### Requirements

- [`kn` CLI](https://knative.dev/docs/client/install-kn/)
- [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

#### Deploy KNative on KinD

```shell
kn quickstart kind
```

### Deploy the setup with Knative `Service`s, `Broker` and `Trigger`s

Deploy a broker of class `MTChannelBasedBroker`:

```shell
kubectl apply -f deploy/broker.yaml
```

Deploy the crawler:

```shell
kubectl apply -f deploy/crawler.yaml
```

Deploy the VPN provisioner:

```shell
kubectl apply -f ./deploy/vpn-provisioner.yaml
```

Deploy a fancy CloudEvents web app to send and visualize on a dashboard event messages:

```shell
kubectl apply -f ./deploy/cloudevents-player.yaml
```

Optionally, you can deploy an CloudEvents logger:

```shell
kubectl apply -f ./deploy/cloudevents-logger.yaml
```

### Produce Events

Now navigate with the browser on http://cloudevents-player.default.127.0.0.1.sslip.io, and send events with:
- ID: generate it
- Type: `dev.knative.crawling.pending`
- Source: as you prefer
- SpecVersion: 1.0
- Message:
  ```json
  {
   "message": "Hello CloudEvents!",
   "vpn_provider": "expressvpn"
  }
  ```

You see the events will be sent, consumed and they will produce further events until an event of type `dev.knative.crawling.ready` will be generated.

The ordered chain of events is as follows:
1. `dev.knative.crawling.pending`
1. `dev.knative.vpn.pending`
1. `dev.knative.vpn.ready`
1. `dev.knative.crawling.ready`

#### Produce events programmatically

By using a simple cURL client:

```shell
kubectl -n default run curl --image=radial/busyboxplus:curl -it

$ curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/default/default" \
-X POST \
-H "Ce-Id: 536808d3-88be-4077-9d7a-a3f162705f79" \
-H "Ce-Specversion: 1.0" \
-H "Ce-Type: dev.knative.crawling.request" \
-H "Ce-Source: dev.knative.samples/helloworldsource" \
-H "Content-Type: application/json" \
-d '{"message":"Hello!","vpn_provider":"expressvpn"}'
```

### Sources

- https://github.com/maxgio92/cloudevents-vpn-provisioner
- https://github.com/maxgio92/cloudevents-crawler
