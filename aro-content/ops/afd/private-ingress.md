## Create a Private Ingress Controller

As you may notice, the clusters that we have provisioned for this workshop are internet accessible. We refer to this as a "public" cluster, where the cluster API and default ingress controller are exposed to the Internet. We do this to eliminate complex networking in our workshop environments. To simulate a private environment for our applications though, we need to create a second ingress controller that is only exposed to the private network of our cluster.

1. To begin, we'll need to gather a few pieces of information from the existing default ingress controller. To do so, run the following commands:

    ```bash
    export CERT=$(oc get IngressController default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}')
    export DOMAIN=$(oc get IngressController default -n openshift-ingress-operator -o jsonpath='{.status.domain}' | sed "s/apps/apps-private/g")
    ```

1. Next, we'll apply the following YAML file to our cluster to create a second ingress controller by running the following command:

    ``` bash
    cat << EOF | oc apply -f -
    apiVersion: v1
    items:
    - apiVersion: operator.openshift.io/v1
      kind: IngressController
      metadata:
        finalizers:
        - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
        generation: 2
        name: private
        namespace: openshift-ingress-operator
      spec:
        clientTLS:
          clientCA:
            name: ""
          clientCertificatePolicy: ""
        defaultCertificate:
          name: ${CERT}
        httpCompression: {}
        httpEmptyRequestsPolicy: Respond
        httpErrorCodePages:
          name: ""
        replicas: 2
        tuningOptions: {}
        domain: ${DOMAIN}
        endpointPublishingStrategy:
          loadBalancer:
            scope: Internal
          type: LoadBalancerService
        observedGeneration: 2
        selector: ingresscontroller.operator.openshift.io/deployment-ingresscontroller=private
        tlsProfile:
          ciphers:
          - ECDHE-ECDSA-AES128-GCM-SHA256
          - ECDHE-RSA-AES128-GCM-SHA256
          - ECDHE-ECDSA-AES256-GCM-SHA384
          - ECDHE-RSA-AES256-GCM-SHA384
          - ECDHE-ECDSA-CHACHA20-POLY1305
          - ECDHE-RSA-CHACHA20-POLY1305
          - DHE-RSA-AES128-GCM-SHA256
          - DHE-RSA-AES256-GCM-SHA384
          - TLS_AES_128_GCM_SHA256
          - TLS_AES_256_GCM_SHA384
          - TLS_CHACHA20_POLY1305_SHA256
          minTLSVersion: VersionTLS12
    kind: List
    EOF
    ```

1. Once this has been done, we need to check to make sure the private ingress controller was created. To do so, run the following command:

    ```bash
    oc get IngressController private -n openshift-ingress-operator -o jsonpath='{.status.conditions}' | jq
    ```

    You'll see a lot of output, but the main thing you're looking for is that the deployment is available:

    ```json
    [...]
      {
        "lastTransitionTime": "2022-11-15T04:02:05Z",
        "message": "The deployment has Available status condition set to True",
        "reason": "DeploymentAvailable",
        "status": "True",
        "type": "DeploymentAvailable"
      },
    [...]
    ```

