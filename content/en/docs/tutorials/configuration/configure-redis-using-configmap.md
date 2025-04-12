---
reviewers:
- eparis
- pmorie
title: Configuring Redis Using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

This page provides a real-world example of how to configure Redis using a [ConfigMap](/docs/concepts/configuration/configmap/) and builds upon the [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task. 



## {{% heading "objectives" %}}

* Create a ConfigMap with Redis configuration values.
* Create a Redis pod that mounts and uses the created ConfigMap.
* Verify that the configuration was correctly applied.

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

### Additional prerequisites

* The example shown on this page requires `kubectl` 1.14 or above.
* Understand [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

<!-- lessoncontent -->

## Real-world example: Configuring Redis using a ConfigMap

Follow the steps below to configure a Redis cache using data stored in a ConfigMap:

1. Create a ConfigMap to configure Redis for `maxmemory` of `3mb` and `maxmemory-policy` of `allkeys-lru`:

    ```shell
    cat <<EOF >./example-redis-config.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: |
        maxmemory 3mb
        maxmemory-policy allkeys-lru
    EOF
    ```

1. Apply the ConfigMap, along with an example Redis pod manifest:

    ```shell
    kubectl apply -f example-redis-config.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

1. Examine the contents of the Redis pod manifest and verify the following:

    * A volume named `config` is created by `spec.volumes[1]`.
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the 
      `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

    This exposes the data in `data.redis-config` from the `example-redis-config`
    ConfigMap above as `/redis-master/redis.conf` inside the pod.

    {{% code_sample file="pods/config/redis-pod.yaml" %}}

1. Examine the created objects:

    ```shell
    kubectl get pod/redis configmap/example-redis-config 
    ```

    You should see the following output:

    ```
    NAME        READY   STATUS    RESTARTS   AGE
    pod/redis   1/1     Running   0          8s

    NAME                             DATA   AGE
    configmap/example-redis-config   1      14s
    ```

1. Verify that the `redis-config` key in the `example-redis-config` ConfigMap contains values for `maxmemory` and `maxmemory-policy`:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    Under `redis-config`, you should see the values for `maxmemory` and `maxmemory-policy` as defined in the ConfigMap:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ----
    maxmemory 3mb
    maxmemory-policy allkeys-lru
    ```

1. Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

1. Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
     ```

    It should show the value of `3145728`:

    ```shell
    1) "maxmemory"
    2) "3145728"
    ```

1. Similarly, check `maxmemory-policy`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    It should show the value of `allkeys-lru`:

    ```shell
    1) "maxmemory-policy"
    2) "allkeys-lru"
    ```

1. Enter `exit` to exit the pod when you are done checking the configuration:
    ```shell
    127.0.0.1:6379> exit
    ```

### Changing an existing configuration

To change an existing Redis configuration using a ConfigMap, you must edit the ConfigMap and restart the pod for the changes to take effect:

1. Add, remove, or change configuration values in the `example-redis-config` ConfigMap. In this example, change `maxmemory` from `3mb` to `2mb`:

    {{% code_sample file="pods/config/example-redis-config.yaml" %}}

1. Apply the updated ConfigMap:

    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

1. Confirm that the ConfigMap was updated:

    ```shell
    kubectl describe configmap/example-redis-config
    ```

    You should see the new configuration values:

    ```shell
    Name:         example-redis-config
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    redis-config:
    ----
    maxmemory 2mb
    maxmemory-policy allkeys-lru
    ```

1. Delete and recreate the pod to load the updated values from the associated ConfigMap:

    ```shell
    kubectl delete pod redis
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

1. Check the configuration values to verify that the new value has been updated:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

    Check `maxmemory`:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    It should now return the updated value of `2097152`:

    ```shell
    1) "maxmemory"
    2) "2097152"
    ```

1. Enter `exit` to exit the pod when you are done checking the configuration:
    ```shell
    127.0.0.1:6379> exit
    ```

## {{% heading "cleanup" %}}

When you are finished with this tutorial, clean up your work by deleting the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}

* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).