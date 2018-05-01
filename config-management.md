## Configuration management with Spring Boot

So far our catalog service has been using an in-memory H2 database. Although H2 is a
convenient database to run locally on your laptop, it's in no ways appropriate for production
or even integration tests. Since it's strongly recommended to use the same technology stack
(operating system, JVM, middleware, database, etc) that is used in production across all environments,
we will modify the Catalog service to use a PostgreSQL database instead of the H2 in-memory database.

Fortunately, OpenShift supports stateful applications such as databases which required access to a
persistent storage that survives the container itself. You can deploy databases on OpenShift and
regardless of what happens to the container itself, the data is safe and can be used by the next
database container.

As part of this lab, a PostgreSQL database has already been deployed, which you can see:

~~~console
$ oc get pods -l app=catalog
NAME                         READY     STATUS    RESTARTS   AGE
catalog-2-cb9lk              1/1       Running   0          6m
catalog-postgresql-1-c4rsm   1/1       Running   0          41m
~~~

Our Spring Boot Catalog application configuration is provided via a properties filed called `application-default.properties`
and can be overriden and [overlayed via multiple mechanisms](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html).

In this scenario, you will configure the Catalog service which is based on Spring Boot to override the default
H2 database configuration using alternative configuration values backed by an [OpenShift ConfigMap](https://docs.openshift.org/latest/dev_guide/configmaps.html).

The ConfigMap has been created for you already, which supplies the necessary PostgreSQL configuration to be
able to connect to the PostgreSQL database:

~~~yaml
% oc get configmap/catalog -o yaml
apiVersion: v1
data:
  application.properties: |-
    spring.datasource.url=jdbc:postgresql://catalog-postgresql:5432/catalog
    spring.datasource.username=********
    spring.datasource.password=********
    spring.datasource.driver-class-name=org.postgresql.Driver
    spring.jpa.hibernate.ddl-auto=create
kind: ConfigMap
metadata:
    ...
~~~

The username and password was generated when you first deployed the catalog template, and the ConfigMap shown
above was also created at that time.

### Use ConfigMap

Let's modify our application to use it. We will use the [Spring Cloud Kubernetes](https://github.com/fabric8io/spring-cloud-kubernetes#configmap-propertysource)
project. Using this dependency, Spring Boot will search for a ConfigMap (by default with the same name as
the application) to use as the source of application configuration during application bootstrapping and
if enabled, triggers hot reloading of beans or Spring context when changes are detected on the ConfigMap.
Add the following dependency to your `pom.xml` beneath the existing
dependencies (look for the `<!-- add additional dependencies -->` comment):

~~~xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
~~~

Although Spring Cloud Kubernetes tries to discover ConfigMaps, due to security reasons containers
by default are not allowed to snoop around OpenShift clusters and discover objects. Security comes first,
and discovery is a privilege that needs to be granted to containers in each project.

Since you do want Spring Boot to discover the ConfigMaps inside the **dev** project, you
need to grant permission to the Spring Boot service account to access the OpenShift REST API and find the
ConfigMaps. To grant this permission, run:

~~~bash
oc policy add-role-to-user view -n dev -z default
~~~

Let's re-run our tests just to make sure our new additions don't cause breakage:

Click on **View command palette** button up to the right
![View command palette button]({% image_path service-to-service-view-cmd-palette.png %})

Then double click on the **TEST** command:
![View command palette test button]({% image_path service-to-service-cmd-palette.png %})

At the end of the output you should see `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0`
and finally `BUILD SUCCESS`.

If your test fails go back and check previous steps before moving to the next section.

### Commit code changes

Commit the `pom.xml` changes to code repo:

* In the project explorer, right click on catalog and then Git > Commit
* Make sure `pom.xml` is checked
* Add a commit message "externalized database configuration"
* Check the "Push commit changes to..."
* Click on "Commit"

This will trigger the pipeline build.

Open the OpenShift Web Console and click on **Builds** > **Pipelines**: to watch the pipeline build and deploy the updated catalog code:
`{{OPENSHIFT_MASTER_URL}}`{: style="color: blue"}

|**CAUTION:** Replace `GUID` with the guid provided to you.

![Build Pipeline]({% image_path create-catalog-pipeline.png %}){:width="700px"}

|**NOTE:** If your pipeline is not triggered, you may have forgotten to *Push* the changes after commit. In that case, simply use the `Git -> Remotes -> Push...` menu to push the committed changes.

Wait for it to complete. At this point the application
should fetch the configuration and start using PostgreSQL instead of H2.

### Verify configuration

Let's verify that PostgreSQL is being used. First, let's dump the database contents by
running the `psql` utility command from within the PostgreSQL container to verify that the seed data is loaded:

~~~sh
oc rsh dc/catalog-postgresql /bin/bash -c \
  "psql -U \$POSTGRESQL_USER -d \$POSTGRESQL_DATABASE -c \"select itemId, name, price from catalog\""
~~~

You should see the seed data gets listed.

~~~
 itemid |          name          | price
--------+------------------------+-------
 329299 | Red Fedora             | 34.99
 329199 | Forge Laptop Sticker   |   8.5
 165613 | Solid Performance Polo |  17.8
 165614 | Ogio Caliber Polo      | 28.75
 165954 | 16 oz. Vortex Tumbler  |     6
 444434 | Pebble Smart Watch     |    24
 444435 | Oculus Rift            |   106
 444436 | Lytro Camera           |  44.3
(8 rows)
~~~

Now let's do a quick update of the price of the _Lytro Camera_ product to make it cost 100.00 by referring to its
itemid of `444436`:

~~~sh
oc rsh dc/catalog-postgresql /bin/bash -c \
  "psql -U \$POSTGRESQL_USER -d \$POSTGRESQL_DATABASE -c \"update catalog set price=100.0 where itemid='444436'\""
~~~

You will see a returned message of `UPDATE 1` indicating that the price was updated in Postgres. Let's re-fetch the catalog
and verify the new price:

~~~sh
oc rsh dc/catalog-postgresql /bin/bash -c \
  "psql -U \$POSTGRESQL_USER -d \$POSTGRESQL_DATABASE -c \"select itemId, name, price from catalog\""
~~~

The _Lytro Camera_ product should now cost `100`:

~~~
 itemid |          name          | price
--------+------------------------+-------
 329299 | Red Fedora             | 34.99
 329199 | Forge Laptop Sticker   |   8.5
 165613 | Solid Performance Polo |  17.8
 165614 | Ogio Caliber Polo      | 28.75
 165954 | 16 oz. Vortex Tumbler  |     6
 444434 | Pebble Smart Watch     |    24
 444435 | Oculus Rift            |   106
 444436 | Lytro Camera           |   100  <----- New price!
~~~

### Congratulations!

You've now got a quick way to alter service configuration without redeploying! As the application moves
through different environments (test, staging, production), it will pick up its configuration via a
[ConfigMap](https://docs.openshift.org/latest/dev_guide/configmaps.html) within each environment, rather than being re-compiled with the new configuration each time.
This mechanism can also be used to alter business logic in addition to infrastructure (database, etc)
configuration.