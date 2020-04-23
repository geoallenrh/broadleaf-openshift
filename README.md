# broadleaf-openshift

Broadleaf Commerce is a popular opensource eCommerce plaform.  Given it's architecture, I was curious how much effort it would require to deploy in Red Hat Openshift.

With the Fabric8 Maven Plugin, it was super easy.  Here are the steps I followed to deploy the 
Broadleaf Commerce DemoSite - https://github.com/BroadleafCommerce/DemoSite

Versions 
OpenShift Version
4.3.5

Clone the DemoSite (I selected the default branch - 
https://github.com/BroadleafCommerce/DemoSite/tree/develop-6.0.x)

## Project Overview

The Community Demo is comprised of 4 individual projects:
```
├── admin
├── api
├── core
└── site
```

- `core` a common jar that all other projects depend on, used for common functionality like domain
- `site` - a Spring Boot application that runs the Heat Clinic UI built with Thymeleaf as traditional MVC

I did not make any changes to the 'core' or 'site' modules.  

- `admin` - a Spring Boot application for the Broadleaf admin for catalog management, see completed orders, etc
- `api` - a Spring Boot application that sets up the Broadleaf API endpoints

For the 'admin' and 'api' modules, I updated the /src/main/resources/runtime-properties/default.properties with the same 'server.port=8443' value as the 'site' module. 

Since these modules will be in their own kubernetes Pod, there will not be able port conflicts.  By default, Broadleaf uses HTTPS, so I just maintained the current configuration.

There is an option to update the environment variable in OpenShift and modify the port during deployment.

## Update the root pom.xml

```shell
Update the base pom.xml with with fabric8 maven plugin
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>fabric8-maven-plugin</artifactId>
    <version>4.4.1</version>
    <executions>
        <execution>
        <id>fmp</id>
        <goals>
            <goal>resource</goal>
            <goal>build</goal>
        </goals>
        </execution>
    </executions>
</plugin>
```

## Confirm the build
mvn clean install

## Ensure you have an OCP enviornment provisioned and have logged in with appropriate credentials and create a new project
oc new-project broadleaf-demo

## Deploy all modules
mvn fabric8:deploy

## Confirm the pods have deployed
oc get pods

```shell
kubectl get pods --field-selector status.phase=Running
NAME                                READY   STATUS    RESTARTS   AGE
boot-community-demo-admin-1-nj9hx   1/1     Running   0          7h31m
boot-community-demo-api-1-jj6lc     1/1     Running   0          7h28m
boot-community-demo-site-1-x9hfp    1/1     Running   0          7h34m
```

## Update generated Services and Routes
The auto generated Services only define port 8080.  But recall Broadleaf default configuration is https on 8443.  Until this is predefined, you need to update the Services and Routes for the correct port and TLS enabled route.

I update via the web console, but working to provide appropriate fragments to automate these.

## Services.yaml fragment
```shell
spec:
  ports:
    - name: https
      protocol: TCP
      port: 8443
      targetPort: 8443
```
## Route.yaml fragment
```shell
spec:
  host: >-
    broadleaf-site-broadleaf-demosite.apps.cluster-sgf-ca23.sgf-ca23.example.opentlc.com
  to:
    kind: Service
    name: boot-community-demo-site
    weight: 100
  port:
    targetPort: https
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: None
  wildcardPolicy: None
```

## Now test with appropriate Route urls
https://www.broadleafcommerce.com/docs/core/current/getting-started/running-locally

## Admin
Once this has finished executing, you should be able to see the Heat Clinic Admin by going to <AdminRoute>/admin in your browser. The username and password is admin/admin.

## Site
Go to base URL in your browser for the Heat Clinic Store

## API
Swagger API definitions by going to <APIRoute>/api/v1/swagger-ui.html.