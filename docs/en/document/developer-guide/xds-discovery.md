# Service Discovery Based on xDS Protocol

This document introduces how to use the xDS service discovery capability provided by Sermant in development.

## Feature Introduction

**Service discovery based on xDS protocol** allows Sermant to connect to Istio's control plane to obtain Kubernetes service information and the corresponding specific service instance information.

> Note: Using Sermant's xDS service discovery capability requires the service to be deployed in a Kubernetes container environment. 

## Development Example

This development example is based on the project created in the [Creating Your First Plugin](README.md) document and demonstrates how the plugin uses the xDS service discovery capability provided by the Sermant framework to obtain service instances:

1. Add the `xdsServiceDiscovery` variable in the `com.huaweicloud.sermant.template.TemplateDeclarer` class under `template\template-plugin` in the project to get the xDS service discovery service provided by the Sermant framework for obtaining service instances:

   ```java
   XdsServiceDiscovery xdsServiceDiscovery = ServiceManager.getService(XdsCoreService.class).getXdsServiceDiscovery();
   ```

2. After obtaining the service discovery instance, you can call the APIs provided by `XdsServiceDiscovery` to perform corresponding actions. This example demonstrates how to directly obtain service instance information for a service named `service-test`:

   ```java
   @Override
   public ExecuteContext before(ExecuteContext context) throws Exception {
     Set<ServiceInstance> serviceInstance = xdsServiceDiscovery.getServiceInstance("service-test");
     Iterator<ServiceInstance> iterator = serviceInstance.iterator();
     while (iterator.hasNext()) {
         ServiceInstance instance = iterator.next();
         System.out.println("ServiceInstance: [" + instance.getHost() + " : "
                 + instance.getPort() + "]");
     }
   }
   ```

After completing the development, refer to the [Packaging and Building](README.md#Packaging and Building) process in the Creating Your First Plugin document. Execute **mvn package** in the root directory of the project to generate the build artifact.

   > Note: `service-test` must be the name of the Kubernetes service. The obtained service instance information corresponds to the pod instance data of that service.

4. Enable the xDS service and set it to be enabled in `agent/config/config.properties` with the following configuration:
   ```properties
   # xDS service switch
   agent.service.xds.service.enable=true
   ```

5. After completing the execution, package the Sermant image and the host microservice image. Start the `service-test`service in the k8s environment and create the corresponding service instances (implemented by the user). Finally, start the host application in Kubernetes and mount Sermant. Refer to the [Sermant Injector User Guide](../user-guide/sermant-injector.md#Startup and Result Verification) for guidance on packaging Sermant and the host image in the Kubernetes environment, as well as mounting Sermant for startup.

6. After the host microservice pod with Sermant mounted starts successfully, you can execute the following command to obtain the host microservice logs and view the service instances obtained through the xDS service discovery capability:

   ```shell
   $ kubectl logs -f ${POD_NAME}
   ServiceInstance: [xxx.xxx.xxx.xxx : xxxx]
   ```
	> Note: `${POD_NAME}` must be the name of the pod running the host microservice, which can be viewed using the `kubectl get pod` command.

## API & Configuration

#### Obtain xDS Service Discovery Service

```java
XdsServiceDiscovery xdsServiceDiscovery = ServiceManager.getService(XdsCoreService.class).getXdsServiceDiscovery();
```

### API

The **xDS Service Discovery** service has two interface methods, used respectively for directly obtaining service instances and obtaining service instances through subscription: 

```java
Set<ServiceInstance> getServiceInstance(String serviceName);

void subscribeServiceInstance(String serviceName, XdsServiceDiscoveryListener listener);
```

#### Directly Obtain Service Instance

- Obtain the service instance information of the Kubernetes service, returning a `java.util.Set` containing all instances of the service.

    ```java
    xdsServiceDiscovery.getServiceInstance("service-test")
    ```

#### Subscribe to Obtain Service Instances

- Subscribe to the service instance information of the Kubernetes service and register a service discovery listener. The listener's `process` function can execute custom operations when service instances change.

  ```java
  xdsServiceDiscovery.subscribeServiceInstance("service-test", new XdsServiceDiscoveryListener() {
              @Override
              public void process(Set<ServiceInstance> instances) {
                  // do something
              }
          });
  ```

- [XdsServiceDiscoveryListener](https://github.com/sermant-io/Sermant/blob/develop/sermant-agentcore/sermant-agentcore-core/src/main/java/io/sermant/core/service/xds/listener/XdsServiceDiscoveryListener.java)，contains the following interface methods:

  | Method                                       | Description                                                  |
  | :------------------------------------------- | :----------------------------------------------------------- |
  | void process(Set<ServiceInstance> instances) | Callback interface for processing the latest service instance information |

- [ServiceInstance](https://github.com/sermant-io/Sermant/blob/develop/sermant-agentcore/sermant-agentcore-core/src/main/java/io/sermant/core/service/xds/entity/ServiceInstance.java), with the following methods:

  | Return Type         | Method Name      | Description                                             |
  | :------------------ | :--------------- | :------------------------------------------------------ |
  | String              | getClusterName() | Get the Istio cluster name the instance belongs to      |
  | String              | getServiceName() | Get the Kubernetes service name the instance belongs to |
  | String              | getHost()        | Get the Pod IP of the instance                          |
  | int                 | getPort()        | Get the port of the instance                            |
  | Map<String, String> | getMetaData()    | Get the metadata of the instance                        |
  | boolean             | isHealthy()      | Check if the service is healthy                         |

### Configuration

In the `agent/config/config.properties` file of the Sermant Agent package, you can enable the xDS service using the `agent.service.xds.service.enable` configuration item. Other xDS service configurations are also set in this file:

```
# xDS service switch
agent.service.xds.service.enable=false
```
