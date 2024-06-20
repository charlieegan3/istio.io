---
title: "Can Your Platform Do Policy? Accelerate Teams With Platform L7 Policy Functionality"
description: Is policy your core competency? Likely not, but you need to do right. Do it once with Istio and OPA and get back team focus on what matters most.
publishdate: 2024-05-22
attribution: "Antonio Berben (Solo.io), Charlie Egan (Styra)"
keywords: [istio,opa,policy,platform,authorization]
---

The era of the platform is here. A platform refers to a centralized infrastructure provided by organizations to support application development and deployment. These platforms offer resources, tools, and shared functionalities that enable development teams to avoid building everything from scratch. Behind every great application team is a great platform, and a great platform team. Good platform teams, through the platforms they build, accelerate others, and the best platform teams are always thinking about new ways to do so more efficiently. If you're on a platform team and keen to make the best use of your team's time, now is a great time to ask: what’s the highest value platform feature you can offer the tenants of your platform?

Two of the main ways we can deliver efficient and valuable features to application teams are through standardization and automation.

* **Standardization** is doing things in a consistent way. When functionality is standardized, the benefits are accessed more easily. Teams can share knowledge,
contribute to shared code and documentation, and collaborate more effectively.

* **Automation** is about removing manual work. When functionality is automated, it can be replicated more quickly to new users and teams. Teams can focus on the high-value work that requires human creativity and problem-solving, rather than repetitive tasks like deployments and monitoring.

Istio is a powerful tool for standardization. When running in Istio, applications
run are exposed to, and interact with, other services in a consistent way. Platform features like traffic management and access control can be configured centrally to deliver shared functionality faster.

Open Policy Agent (OPA) also offers both a standard way to describe policy, and a means of automatically making it available where it's needed. Coupled with Envoy's external authorization integration this functionality can be made available to applications running in an Istio service mesh to quickly deliver layer 7 policy controls.

---

We're going to dive into how Istio and Open Policy Agent (OPA) can be used to enforce layer 7 policies in your platform. We'll show you how to get started with a simple example.

Hopefully you will come to see how Istio and Open Policy Agent (OPA) are solid option to deliver this valuable feature, quickly and transparently to application team everywhere in the business - while also providing the data the security teams need for audit and compliance.

## Try it out

When integrated with Istio, OPA can be used to enforce fine-grained access control policies for microservices. This guide shows how to integrate Istio with OPA to enforce access control policies for a simple microservices application.

### Prerequisites

- A Kubernetes cluster with Istio installed.
- The `istioctl` command-line tool installed.

Install Istio:

{{< text bash >}}
$ stioctl install -y -f iop.yaml
{{< /text >}}

Notice that in the configuration, we define an `extensionProviders` section that points to the OPA standalone installation.

Deploy the sample application

Httpbin is a well-known application that can be used to test HTTP requests and helps to show quickly how we can play with the request and response attributes.

{{< text bash >}}
$ kubectl create ns my-app
$ kubectl label namespace my-app istio-injection=enabled

$ kubectl apply -f apps.yaml
{{< /text >}}

Deploy OPA. It will fail because it expects a configMap containing the default rego rule to use. This configMap will be deployed later in our example.

{{< text bash >}}
$ kubectl create ns opa
$ kubectl label namespace opa istio-injection=enabled

$ kubectl apply -f opa.yaml
{{< /text >}}

Deploy the `AuthorizationPolicy` to define which services will be protected by OPA.

{{< text bash >}}
$ kubectl apply -f authorizationPolicy.yaml
{{< /text >}}

Notice that in this resource we define the opa extensionProvider you configured in the Istio installation:

{{< text yaml >}}
[...]
  provider:
    name: "opa.local"
[...]
{{< /text >}}

## How it works

When applying the AuthorizationPolicy, the Istio control plane (istiod) sends the required configurations to the istio-proxy (envoy) of the selected services in the policy. The envoy will then send the request to the OPA server to check if the request is allowed or not.

{{< image width="75%"
    link="./opa1.png"
    alt="Istio and OPA"
    >}}

Istio-proxy is an Envoy proxy that is deployed as a sidecar container in the same pod as the application container. The envoy proxy works by configuring filters in a chain.

One of those filters is ext_authz, which implements ext_authz protobuf service with a specific message. Any server implementing the protobuf can connect with the envoy proxy and provide the authorization decision. OPA is one of those servers.

{{< image width="75%"
    link="./opa2.png"
    alt="Filters"
    >}}

Before, when you installed OPA server, you used the envoy version of the server. This image allows the configuration of the GRPC plugin which implements the ext_authz protobuf service.

{{< text yaml >}}
[...]
      containers:
      - image: openpolicyagent/opa:0.61.0-envoy # This is the OPA image version which brings the envoy plugin
        name: opa
[...]
{{< /text >}}

In the configuration, you have enabled the envoy plugin and the port which will listen:

{{< text yaml >}}
[...]
    decision_logs:
      console: true
    plugins:
      envoy_ext_authz_grpc:
        addr: ":9191" # This is the port where the envoy plugin will listen
        path: mypackage/mysubpackage/myrule # Default path for grpc plugin
    # Here you can add your own configuration with services and bundles
[...]
{{< /text >}}

Reviewing the [Envoy's Authorization service documentation](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/auth/v3/external_auth.proto), you can see that the message has these attributes:

{{< text json >}}
OkHttpResponse
{
  "status": {...},
  "denied_response": {...},
  "ok_response": {
      "headers": [],
      "headers_to_remove": [],
      "dynamic_metadata": {...},
      "response_headers_to_add": [],
      "query_parameters_to_set": [],
      "query_parameters_to_remove": []
    },
  "dynamic_metadata": {...}
}
{{< /text >}}

This means that based on the response from the Authz server, the envoy can add or remove headers, query parameters, and even change the response status. So that OPA can do the same as it is described in the [OPA's documentation](https://www.openpolicyagent.org/docs/latest/envoy-primer/#example-policy-with-additional-controls).

## Testing

Let's test the simple usage (authorization) and then let's create a more advanced rule to show how we can use OPA to modify the request and response.

Deploy an app to run curl commands to the Httpbin sample application:

{{< text bash >}}
$ kubectl -n my-app run --image=curlimages/curl curly -- /bin/sleep 100d
{{< /text >}}

Apply the first rego rule and restart the OPA deployment:

{{< text bash >}}
$ kubectl apply -f opa-example-1-configmap.yaml

$ kubectl rollout restart deployment -n opa
{{< /text >}}

The simple scenario is to allow requests if they contain the header `x-force-authorized` with the value `enabled` or `true`. If the header is not present or has a different value, the request will be denied.

There are multiple ways to create the rego rule. In this case, we created two different rules. Executed in order, the first one which satisfies all the conditions will be the one that will be used.

### Simple rule

The following request will return `403`:

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s -w "\nhttp_code=%{http_code}" httpbin/get
{{< /text >}}

The following request will return `200` and the body:

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s -w "\nhttp_code=%{http_code}" httpbin/get -H "x-force-authorized: enabled"
{{< /text >}}

### Advanced manipulations

Now the more advanced rule. Apply the second rego rule and restart the OPA deployment:

{{< text bash >}}
$ kubectl apply -f opa-example-2-configmap.yaml

$ kubectl rollout restart deployment -n opa
{{< /text >}}

In that rule, you can see:

{{< text plain >}}
myrule["allowed"] := allow # Notice that `allowed` is mandatory when returning an object, like here `myrule`
myrule["headers"] := headers
myrule["response_headers_to_add"] := response_headers_to_add
myrule["request_headers_to_remove"] := request_headers_to_remove
myrule["body"] := body
myrule["http_status"] := status_code
{{< /text >}}

And those are the values that will be returned to the envoy proxy. The envoy will use those values to modify the request and response.

Notice that `allowed is required when returning a JSON object instead of only true/false. This can be found in the documentation [here](https://www.openpolicyagent.org/docs/latest/envoy-primer/#output-document).

#### Change returned body

Let's test the new capabilities:

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s -w "\nhttp_code=%{http_code}" httpbin/get
{{< /text >}}

Now we can change the response body. With "403" the body in the Rego rule is changed to "Unauthorized Request". With the previous command, you should receive:

{{< text plain >}}
Unauthorized Request
http_code=403
{{< /text >}}

#### Change returned body and status code

Running the request with the header "x-force-authorized: enabled" you should receive the body "Authentication Failed" and error "401":

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s -w "\nhttp_code=%{http_code}" httpbin/get -H "x-force-unauthenticated: enabled"
{{< /text >}}

#### Adding headers to request

Running a valid request, you should receive the echo body with the new header "x-validated-by: my-security-checkpoint" and the header "x-force-authorized" removed:

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s httpbin/get -H "x-force-authorized: true"
{{< /text >}}

#### Adding headers to response

Running the same request but showing only the header, you will find the response header added during the Authz check "x-add-custom-response-header: added":

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s -I httpbin/get -H "x-force-authorized: true"
{{< /text >}}

#### Sharing data between filters

Finally, since the envoy works with filters, you can pass data to the following envoy filters. That is done using "dynamic_metadata".

This is useful when you want to pass data to another ext_authz filter in the change or you want to print it in the application logs.

{{< image width="75%"
    link="./opa3.png"
    alt="Metadata"
    >}}

To do so, review the access log format defined in IstioOperator:

{{< text bash >}}
$ cat iop.yaml
{{< /text >}}

You will see this access logs format:

{{< text plain >}}
[...]
    accessLogFormat: |
      [%START_TIME%] my-new-dynamic-metadata: "%DYNAMIC_METADATA(envoy.filters.http.ext_authz)%"
[...]
{{< /text >}}

The "DYNAMIC_METADATA" is a reserved keyword to access the metadata object. The rest is the name of the filter that you want to access. In your case, the name "envoy.filters.http.ext_authz" is created automatically by Istio. You could verify this by dumping the envoy configuration:

{{< text bash >}}
$ istioctl pc all deploy/httpbin -n my-app -oyaml | grep envoy.filters.http.ext_authz
{{< /text >}}

And you will see the configurations for the filter.

Let's test the dynamic metadata. In the advance rule, you are creating a new metadata entry: `{"my-new-metadata": "my-new-value"}`.

Run the request and check the logs of the application:

{{< text bash >}}
$ kubectl exec -n my-app curly -c curly  -- curl -s -I httpbin/get -H "x-force-authorized: true"

$ kubectl logs -n my-app deploy/httpbin -c istio-proxy --tail 1
{{< /text >}}

You will see in the output the new attributes configured by OPA rego rules:

{{< text plain >}}
[...]
 my-new-dynamic-metadata: "{"my-new-metadata":"my-new-value","decision_id":"8a6d5359-142c-4431-96cd-d683801e889f","ext_authz_duration":7}"
[...]
{{< /text >}}

## Conclusion

In this guide, we have shown how to integrate Istio and OPA to enforce policies for a simple microservices application. We also showed how to use Rego to modify the request and response attributes. This is the foundational example for building a platform-wide policy system that can be used by all application teams.

Some links:

<https://www.openpolicyagent.org/docs/latest/management-decision-logs/>
<https://www.openpolicyagent.org/docs/latest/envoy-tutorial-istio/>
<https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/security/ext_authz_filter.html>
Envoy examples at <https://play.openpolicyagent.org>
