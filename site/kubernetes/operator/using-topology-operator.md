# Using RabbitMQ Messaging Topology Kubernetes Operator

## <a id='overview' class='anchor' href='#overview'>Overview</a>

This guide covers how to deploy Custom Resource objects that will be managed by the Messaging Topology Operator.
If RabbitMQ Cluster Kubernetes Operator is not installed, see the [quickstart guide](quickstart-operator.html).

This guide has the following sections:

* [Requirements](#requirements)
* [Scope across multiple namespaces](#namespace-scope)
* [Queues and policies](#queues-policies)
* [Users and user permissions](#users-permissions)
* [Exchanges and bindings](#exchanges-bindings)
* [Virtual hosts](#vhosts)
* [Federation](#federation)
* [Shovel](#shovel)
* [Update a resource](#update)
* [Delete a resource](#delete)
* [Limitations](#limitations)
* [TLS](#tls)

<p class="note">
  <strong>Note:</strong> Additional information about using the operator on Openshift can be found at
  [Using the RabbitMQ Kubernetes Operators on Openshift](using-on-openshift.html).
</p>

## <a id='requirements' class='anchor' href='#requirements'>Requirements</a>

* Messaging Topology Operator can only be used with RabbitMQ clusters deployed using the Kubernetes [Cluster Operator](https://github.com/rabbitmq/cluster-operator).
The minimal version required for Cluster Operator is `1.7.0`.
* Messaging Topology Operator custom resources can only be created in the same namespace as the RabbitMQ cluster is deployed. For a RabbitmqCluster deployed in namespace
"my-test-namespace", all Messaging Topology custom resources for this RabbitMQ cluster, such as `queues.rabbitmq.com` and `users.rabbitmq.com`, can only be created in namespace "my-test-namespace".

## <a id='namespace-scope' class='anchor' href='#namespace-scope'>Scope across multiple namespaces</a>

Messaging Topology Operator can reconcile its objects in *any* namespace. However, by default, it will limit its interactions to `RabbitmqCluster` objects within
same namespace as the topology object, for example `Queue`. In other words, Messaging Topology Operator will reconcile a `Queue` object, in namespace `default`,
only if `RabbitmqCluster` object is also in namespace `default`.

This is the default behaviour, and it can be customised to allow a specific list of namespaces to allow reconciliation from. To create a list of
allowed namespaces, the `RabbitmqCluster` object has to be annotated with key `rabbitmq.com/topology-allowed-namespaces`, and value a comma-separated list,
without spaces, of namespace names; for example `default,ns1,my-namespace`. The value asterisk `*` can be used to allow **all** namespaces.

Any topology object can be declared to target a `RabbitmqCluster` in a different namespace using `.spec.rabbitmqClusterReference.namespace`. Topology
objects within the same namespace as `RabbitmqCluster` are **always** allowed.

The following YAML declares a `RabbitmqCluster` object that allows topology objects from namespace `my-app`:

<pre class="lang-yaml">
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: example-rabbit
  namespace: rabbitmq-service
  annotations:
    rabbitmq.com/topology-allowed-namespaces: "my-app"
spec:
  replicas: 1
</pre>

Note that the above YAML specifies namespace `rabbitmq-service`. Then the following YAML will target the above RabbitMQ, from namespace `my-app`.

<pre class="lang-yaml">
apiVersion: rabbitmq.com/v1beta1
kind: Queue
metadata:
  name: test # name of this custom resource; does not have to the same as the actual queue name
  namespace: my-app
spec:
  name: test-queue # name of the queue
  rabbitmqClusterReference:
    name: example-rabbit
    namespace: rabbitmq-service
</pre>

### <a id='namespace-scope-note' class='anchor' href='#namespace-scope-note'>A note about forbidden cross-namespace objects</a>

If topology objects, for example `Queue`, target a `RabbitmqCluster`, and such `RabbitmqCluster` does not allow requests from the topology object's
namespace, a [status condition](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions) will be created with an error message,
and the object **will not be reconciled**, until it is updated.

For example, `RabbitmqCluster` in namespace `ns1` allows topologies from `my-app`, and topology object `Queue` in namespace `not-allowed` targets said
`RabbitmqCluster`, Messaging Topology Operator will log an error message, update a status condition in the `Queue` object, and give up reconciling the
`Queue` object. If the `RabbitmqCluster` object is updated to allow topology objects from namespace `not-allowed`, the `Queue` object **must be manually
updated** to trigger a reconciliation; for example, by adding a label to the `Queue` object.

## <a id='queues-policies' class='anchor' href='#queues-policies'>Queues and Policies</a>

Messaging Topology Operator can declare [queues](../../queues.html) and
[policies](../../parameters.html#how-policies-work) in a RabbitMQ cluster.

The following manifest will create a queue named 'test' in the default vhost:

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Queue
metadata:
  name: test # name of this custom resource; does not have to the same as the actual queue name
  namespace: rabbitmq-system
spec:
  name: test # name of the queue
  autoDelete: false
  durable: true
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

The following manifest will create a policy named 'lazy-queue' in default virtual host:

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Policy
metadata:
  name: policy-example # name of this custom resource
  namespace: rabbitmq-system
spec:
  name: lazy-queue
  pattern: "^lazy-queue-" # matches any queue begins with "lazy-queue-"
  applyTo: "queues"
  definition:
    queue-mode: lazy
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

Note that it's not recommended setting [optional queue arguments](../../queues.html#optional-arguments) on queues directly. Once set,
queue properties cannot be changed. Use [policies](../../parameters.html#policies) instead.

The Messaging Topology repo has more examples on [queues](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/queues)
and [policies](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/policies). 

### <a id='exchanges-bindings' class='anchor' href='#exchanges-bindings'> Exchanges and Bindings</a>

Messaging Topology Operator can manage [exchanges and bindings](../../publishers.html#basics).
The following manifest will create a fanout exchange:

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Exchange
metadata:
  name: fanout
  namespace: rabbitmq-system
spec:
  name: fanout-exchange # name of the exchange
  type: fanout # default to 'direct' if not provided; can be set to 'direct', 'fanout', 'headers', and 'topic'
  autoDelete: false
  durable: true
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

The following manifest will create a binding between an exchange and a queue:

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Binding
metadata:
  name: binding
  namespace: rabbitmq-system
spec:
  source: test # an existing exchange
  destination: test # an existing queue
  destinationType: queue # can be 'queue' or 'exchange'
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

More examples on [exchanges](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/exchanges)
and [bindings](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/bindings). 

### <a id='users-permissions' class='anchor' href='#users-permissions'>Users and User Permissions</a>

You can use Messaging Topology Operator to create RabbitMQ users and assign user permissions.
Learn more about user management in the [Access Control guide](../../access-control.html#user-management).

Messaging Topology Operator creates users with generated credentials by default.

The following manifest will create a user with generated username and password and the generated username and password can be
accessed via a Kubernetes secret object:

<pre class="lang-bash"> 
apiVersion: rabbitmq.com/v1beta1
kind: User
metadata:
  name: user-example
  namespace: rabbitmq-system
spec:
  tags:
  - policymaker
  rabbitmqClusterReference:
    name: example-rabbitmq
</pre>

To get the name of the kubernetes secret object that contains the generated username and password, run the following command:

<pre class="lang-bash"> 
kubectl get users.rabbitmq.com user-example -o jsonpath='{.status.credentials.name}'
</pre>

The Operator also supports creating RabbitMQ users with provided credentials. When creating a user with provided username and password, create a kubernetes
secret object contains keys `username` and `password` in its Data field.

The following manifest will create a user with username and password provided from secret 'my-rabbit-user' :

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: User
metadata:
  name: import-user-sample
spec:
  tags:
  - policymaker
  - monitoring # other available tags are 'management' and 'administrator'
  rabbitmqClusterReference:
    name: rabbit-example
  importCredentialsSecret:
    name: my-rabbit-user # name of the secret
</pre>

To set user permissions on an existing user, create `permissions.rabbitmq.com` resources.
The following example will assign permissions to user `rabbit-user-1`:

<pre class="lang-bash"> 
apiVersion: rabbitmq.com/v1beta1
kind: Permission
metadata:
  name: rabbit-user-1-permission
  namespace: rabbitmq-system
spec:
  vhost: "/"
  user: "rabbit-user-1" # name of the RabbitMQ user
  permissions:
    write: ".*"
    configure: ".*"
    read: ".*"
  rabbitmqClusterReference:
    name: sample
</pre>

More examples on [users](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/users)
and [permissions](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/permissions).

## <a id='vhosts' class='anchor' href='#vhosts'>Virtual Hosts</a>

Messaging Topology Operator can create [virtual hosts](../../vhosts.html).

The following manifest will create a vhost named 'test' in a RabbitmqCluster named 'example-rabbit':

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Vhost
metadata:
  name: test-vhost # name of this custom resource
  namespace: rabbitmq-system
spec:
  name: test # name of the vhost
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

## <a id='federation' class='anchor' href='#federation'>Federation</a>

Messaging Topology Operator can define [Federation upstreams](../../federation.html).

Because a Federation upstream URI contains credentials, it is provided through a Kubernetes Secret object.
The 'uri' key is mandatory for the Secret object. Its value can be either a single URI or a comma-separated list of URIs.

The following manifest will define an upstream named 'origin' in a RabbitmqCluster named 'example-rabbit':

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Federation
metadata:
  name: federation-example
  namespace: rabbitmq-system
spec:
  name: "origin"
  uriSecret:
    # secret must be created in the same namespace as this Federation object; in this case 'rabbitmq-system'
    name: {secret-name}
  ackMode: "on-confirm"
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

More [federation examples](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/federations).

## <a id='shovel' class='anchor' href='#shovel'>Shovel</a>

Messaging Topology Operator can declare [dynamic Shovels](../../shovel-dynamic.html).

Shovel source and destination URIs are provided through a Kubernetes Secret object.
The Secret Object must contain two keys, 'srcUri' and 'destUri', and the value of each key can be either a single URI
or a comma-separated list of URIs.

The following manifest will create a Shovel named 'my-shovel' in a RabbitmqCluster named 'example-rabbit':

<pre class="lang-bash">
apiVersion: rabbitmq.com/v1beta1
kind: Shovel
metadata:
  name: shovel-example
  namespace: rabbitmq-system
spec:
  name: "my-shovel"
  uriSecret:
    # secret must be created in the same namespace as this Shovel object; in this case 'rabbitmq-system'
    name: {secret-name}
  srcQueue: "the-source-queue"
  destQueue: "the-destination-queue"
  rabbitmqClusterReference:
    name: example-rabbit
</pre>

More [shovels examples](https://github.com/rabbitmq/messaging-topology-operator/tree/main/docs/examples/shovels).

## <a id='update' class='anchor' href='#update'>Update Resources</a>

Some custom resource properties are immutable. Messaging Topology Operator implements [validating webhooks](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook)
to prevent updates on immutable fields. Forbidden updates will be rejected. For example:

<pre class="lang-bash">
kubectl apply -f test-queue.yaml
Error from server (Forbidden):
...
Resource: "rabbitmq.com/v1beta1, Resource=queues", GroupVersionKind: "rabbitmq.com/v1beta1, Kind=Queue"
Name: "example", Namespace: "rabbitmq-system"
for: "test-queue.yaml": admission webhook "vqueue.kb.io" denied the request: Queue.rabbitmq.com "example" is forbidden: spec.name: Forbidden: updates on name, vhost, and rabbitmqClusterReference are all forbidden
</pre>

Properties that cannot be updated is documented in the [Messaging Topology Operator API docs](https://github.com/rabbitmq/messaging-topology-operator/blob/main/docs/api/rabbitmq.com.ref.asciidoc).

## <a id='delete' class='anchor' href='#delete'>Delete a resource</a>

Deleting custom resources will delete the corresponding resources in the RabbitMQ cluster. Messaging Topology Operator sets kubernetes
[finalizers](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#finalizers) on all custom
resources. If the object has already been deleted in the RabbitMQ cluster, or the RabbitMQ cluster has been deleted, Messaging Topology
Operator will delete the custom without trying to delete the RabbitMQ object.

## <a id='limitations' class='anchor' href='#limitations'>Limitations</a>

Messaging Topology Operator does not work for RabbitmqCluster deployed with importing definitions. The Operator reads the default user secret
set in RabbitmqCluster status `status.binding`, if the RabbitmqCluster is deployed with imported definitions, definitions will overwrite the
default user credentials and may cause Messaging Topology Operator not be able to access the RabbitmqCluster. To get around this issue, either
you can create a RabbitmqCluster without importing definitions, or you can manually update the default user kubernetes secret to the actual
user credentials set in the definitions.

## <a id='tls' class='anchor' href='#tls'>TLS</a>

If the RabbitmqClusters managed by the Messaging Topology Operator are configured to serve the Management over HTTPS, there are some additional
steps required to configure Messaging Topology Operator. Follow this [TLS dedicated guide](/kubernetes/operator/tls-topology-operator.html) to configure
the Operator.
