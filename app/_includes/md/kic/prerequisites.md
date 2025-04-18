{% unless include.disable_accordian %}
<details markdown="1">
<summary>
  <strong>Prerequisites:</strong> Install {{site.kic_product_name}} {% unless include.disable_gateway_api %}with Gateway API support {% endunless %}in your Kubernetes cluster and connect to Kong. {% if include.enterprise %}This guide requires <strong>{{site.ee_product_name}}</strong>.{% endif %}
</summary>

## Prerequisites
{% endunless %}

{% unless include.disable_gateway_api %}

{% assign gw_api_crd_version = "v1.1.0" %}
{% if_version lte:3.1.x %}
{% assign gw_api_crd_version = "v1.0.0" %}
{% endif_version %}

### Install the Gateway APIs

   {% if include.gateway_api_experimental %}

1. Install the experimental Gateway API CRDs before installing {{ site.kic_product_name }}.

    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/{{ gw_api_crd_version}}/experimental-install.yaml
    ```

   {% else %}

1. Install the Gateway API CRDs before installing {{ site.kic_product_name }}.

    ```bash
    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/{{ gw_api_crd_version}}/standard-install.yaml
    ```

   {% endif %}

1. Create a `Gateway` and `GatewayClass` instance to use.

{% assign gwapi_version = "v1" %}
{% if_version lte:2.12.x %}
{% assign gwapi_version = "v1beta1" %}
{% endif_version %}

   ```bash
   echo "
   ---
   apiVersion: gateway.networking.k8s.io/{{ gwapi_version }}
   kind: GatewayClass
   metadata:
     name: kong
     annotations:
       konghq.com/gatewayclass-unmanaged: 'true'

   spec:
     controllerName: konghq.com/kic-gateway-controller
   ---
   apiVersion: gateway.networking.k8s.io/{{ gwapi_version }}
   kind: Gateway
   metadata:
     name: kong
   spec:
     gatewayClassName: kong
     listeners:
     - name: proxy
       port: 80
       protocol: HTTP
       allowedRoutes:
         namespaces:
            from: All
   " | kubectl apply -f -
   ```

   The results should look like this:
   ```text
   gatewayclass.gateway.networking.k8s.io/kong created
   gateway.gateway.networking.k8s.io/kong created
   ```
{% endunless %}

### Install Kong
You can install Kong in your Kubernetes cluster using [Helm](https://helm.sh/).
1. Add the Kong Helm charts:

    ```bash
    helm repo add kong https://charts.konghq.com
    helm repo update
    ```

{% if include.enterprise %}
1. Create a file named `license.json` containing your {{site.ee_product_name}} license and store it in a Kubernetes secret:

    ```bash
    kubectl create namespace kong
    kubectl create secret generic kong-enterprise-license --from-file=license=./license.json -n kong
    ```

1. Create a `values.yaml` file:

    ```yaml
    gateway:
      image:
        repository: kong/kong-gateway
      env:
        LICENSE_DATA:
          valueFrom:
            secretKeyRef:
              name: kong-enterprise-license
              key: license
    ```
{% endif %}

1. Install {{site.kic_product_name}} and {{ site.base_gateway }} with Helm:

{% if include.enable_oss %}

{% capture values_file %}
{% navtabs_ee codeblock %}
{% navtab Kong Gateway %}
```bash
helm install kong kong/ingress -n kong --create-namespace
```
{% endnavtab %}
{% navtab Kong Gateway (OSS) %}
```bash
helm install kong kong/ingress -n kong --create-namespace --set gateway.image.repository=kong --set gateway.image.tag="{{site.latest_gateway_oss_version}}"
```
{% endnavtab %}
{% endnavtabs_ee %}
{% endcapture %}

{{ values_file | indent }}
{% else %}
    ```bash
    helm install kong kong/ingress -n kong --create-namespace {% if include.enterprise %}--values ./values.yaml{% endif %}
    ```
{% endif %}

{% if include.gateway_api_experimental %}
1. Enable the Gateway API Alpha feature gate:

    ```bash
    kubectl set env -n kong deployment/kong-controller CONTROLLER_FEATURE_GATES="GatewayAlpha=true" -c ingress-controller
    ```

   The results should look like this:
   ```text
   deployment.apps/kong-controller env updated
   ```
{% endif %}

### Test connectivity to Kong

Kubernetes exposes the proxy through a Kubernetes service. Run the following commands to store the load balancer IP address in a variable named `PROXY_IP`:

1. Populate `$PROXY_IP` for future commands:

    ```bash
    export PROXY_IP=$(kubectl get svc --namespace kong kong-gateway-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    echo $PROXY_IP
    ```

2. Ensure that you can call the proxy IP:

    ```bash
    curl -i $PROXY_IP
    ```

    The results should look like this:

    ```bash
    HTTP/1.1 404 Not Found
    Content-Type: application/json; charset=utf-8
    Connection: keep-alive
    Content-Length: 48
    X-Kong-Response-Latency: 0
    Server: kong/3.0.0
  
    {"message":"no Route matched with those values"}
    ```

{% unless include.disable_accordian %}
</details>
{% endunless %}
