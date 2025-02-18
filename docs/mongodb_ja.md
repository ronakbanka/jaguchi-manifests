## MongoDBインスタンスの作成とバインド

MongoDBのデプロイには [Bitnami Helm Chart](https://github.com/bitnami/charts/tree/master/bitnami/mongodb) を使用し、 [Direct Secret Reference](https://github.com/servicebinding/spec#direct-secret-reference) でService Bindingでアプリに接続する。
Secretはsecretgen controllerの [Secret Template](https://github.com/vmware-tanzu/carvel-secretgen-controller/blob/develop/docs/secret-template-field.md) で生成する。

### Helm ChartからYAML生成

```
NAMESPACE=<GitHubのアカウント名>
helm repo add bitnami https://charts.bitnami.com/bitnami
helm template -n ${NAMESPACE} music-db bitnami/mongodb --set "auth.usernames[0]=music,auth.databases[0]=music,auth.existingSecret=music-db" > music-db.yaml
```

### Password CRの生成

Secret Templateに設定するキーは、Helm Chartで必須のキー(`mongodb-passwords`, `mongodb-root-password`)と[Spirng Cloud Bindings](https://github.com/spring-cloud/spring-cloud-bindings#mongodb)で期待されているキー(`uri`)

```yaml
cat <<EOF > music-db-password.yaml
apiVersion: secretgen.k14s.io/v1alpha1
kind: Password
metadata:
  name: music-db
  namespace: ${NAMESPACE}
spec:
  secretTemplate:
    type: servicebinding.io/mongodb
    stringData:
      type: mongodb
      uri: mongodb://music:\$(value)@music-db-mongodb:27017/music
      mongodb-passwords: \$(value)
      mongodb-root-password: \$(value)
EOF
```

### MongoDBのデプロイ

[`kapp`](https://github.com/vmware-tanzu/carvel-kapp) CLIでデプロイする

```
kapp deploy -n ${NAMESPACE} -a music-db -f music-db.yaml -f music-db-password.yaml
```

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/168867610-3d7fb9b1-0c14-4824-9103-9ac5b0d615da.png">

`y`を入力


<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/168868192-ffa5d968-646e-4a04-8c40-d8f5876ec38c.png">


↓のコマンドで作成されたリソースを確認できる
```
kapp inspect -n ${NAMESPACE} -a music-db -t
```

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/168868996-c42e5ff6-2d06-424b-a696-a0a3199b5fd7.png">


↓のコマンドで作成されたSecretが [Provisioned Service](https://github.com/servicebinding/spec#provisioned-service) の形式であることを確認できる。
```
kubectl get secret -n ${NAMESPACE} music-db -ojson | jq '.data | map_values(@base64d)'
```
<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/168869966-c0bf7310-f78a-42e2-85d1-502689b145c8.png">

### music-dbをspring-musicアプリにService Bind

Workloadの作成
```
tanzu apps workload apply spring-music \
  --app spring-music \
  --git-repo https://github.com/tanzu-japan/spring-music \
  --git-branch tanzu \
  --type web \
  --annotation autoscaling.knative.dev/minScale=1 \
  --service-ref music-db=v1:Secret:music-db \
  -n ${NAMESPACE}
```

↓でログを確認
```
tanzu apps workload tail spring-music -n ${NAMESPACE}
```

↓で進捗を確認
```
watch kubectl get pod,workload,gitrepo,build,taskrun,pod,workload,gitrepo,build,taskrun,imagerepository,app,ksvc,servicebinding,app,ksvc,servicebinding -l app.kubernetes.io/part-of=spring-music -n ${NAMESPACE}
```

↓のような結果になればOK

```
NAME                                                 READY   STATUS      RESTARTS   AGE
pod/spring-music-00004-deployment-5dc6d99594-jfhrj   2/2     Running     0          113s
pod/spring-music-build-1-build-pod                   0/1     Completed   0          107m
pod/spring-music-config-writer-njcqt-pod             0/1     Completed   0          2m18s
pod/spring-music-config-writer-wzz88-pod             0/1     Completed   0          100m

NAME                              SOURCE                                        SUPPLYCHAIN     READY   REASON   AGE
workload.carto.run/spring-music   https://github.com/tanzu-japan/spring-music   source-to-url   True    Ready    108m

NAME                                                  URL                                           READY   STATUS                                                             AGE
gitrepository.source.toolkit.fluxcd.io/spring-music   https://github.com/tanzu-japan/spring-music   True    Fetched revision: tanzu/7a7641687498e837a34a3e07964bab589285084d   108m

NAME                                  IMAGE                                                                                                         SUCCEEDED
build.kpack.io/spring-music-build-1   ghcr.io/jaguchi/spring-music-making@sha256:b0891f45ae297987607d2099b6c5a0a760c5736c1f09bb019d8c64cf734faba3   True

NAME                                                  SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/spring-music-config-writer-njcqt   True        Succeeded   2m20s       2m8s
taskrun.tekton.dev/spring-music-config-writer-wzz88   True        Succeeded   100m        100m

NAME                                                                 IMAGE                                                                             URL                                                                                                                                                                                                       READY   REASON   AGE
imagerepository.source.apps.tanzu.vmware.com/spring-music-delivery   ghcr.io/jaguchi/spring-music-making-bundle:dc601cd8-8d38-4df1-bbc9-8e51c6865b8b   http://source-controller-manager-artifact-service.source-system.svc.cluster.local./imagerepository/making/spring-music-delivery/f0a5556ac4fe2d80c96c3af20a0df07637c777f9b2ed52c26bd2823e69b81d16.tar.gz   True    Ready    107m

NAME                                DESCRIPTION           SINCE-DEPLOY   AGE
app.kappctrl.k14s.io/spring-music   Reconcile succeeded   119s           107m

NAME                                       URL                                                 LATESTCREATED        LATESTREADY          READY   REASON
service.serving.knative.dev/spring-music   https://spring-music-making.apps.jaguchi.maki.lol   spring-music-00004   spring-music-00004   True    

NAME                                                     READY   REASON   AGE
servicebinding.servicebinding.io/spring-music-music-db   True    Ready    99m
```

`tanzu apps workload get`でも確認。

```
$ tanzu apps workload get spring-music -n ${NAMESPACE}
# spring-music: Ready
---
lastTransitionTime: "2022-05-17T15:25:26Z"
message: ""
reason: Ready
status: "True"
type: Ready

Services
CLAIM      NAME       KIND     API VERSION
music-db   music-db   Secret   v1

Pods
NAME                                             STATUS      RESTARTS   AGE
spring-music-00004-deployment-5dc6d99594-jfhrj   Running     0          2m32s
spring-music-build-1-build-pod                   Succeeded   0          108m
spring-music-config-writer-njcqt-pod             Succeeded   0          2m57s
spring-music-config-writer-wzz88-pod             Succeeded   0          101m

Knative Services
NAME           READY   URL
spring-music   Ready   https://spring-music-making.apps.jaguchi.maki.lol
```

↓のログを確認してMongoDBに接続できていることを確認。

```
$ kubectl logs -l app.kubernetes.io/component=run,app.kubernetes.io/part-of=spring-music -c workload -n ${NAMESPACE} --tail=1000
Setting Active Processor Count to 4
Calculating JVM memory based on 12912960K available memory
`For more information on this calculation, see https://paketo.io/docs/reference/java-reference/#memory-calculator
Calculated JVM Memory Configuration: -XX:MaxDirectMemorySize=10M -Xmx12267527K -XX:MaxMetaspaceSize=133432K -XX:ReservedCodeCacheSize=240M -Xss1M (Total Memory: 12912960K, Thread Count: 250, Loaded Class Count: 21144, Headroom: 0%)
Enabling Java Native Memory Tracking
Adding 128 container CA certificates to JVM truststore
Spring Cloud Bindings Enabled
Picked up JAVA_TOOL_OPTIONS: -Dmanagement.endpoint.health.probes.add-additional-paths="true" -Dmanagement.endpoint.health.show-details=always -Dmanagement.endpoints.web.base-path="/actuator" -Dmanagement.endpoints.web.exposure.include=* -Dmanagement.health.probes.enabled="true" -Dmanagement.server.port="8081" -Dserver.port="8080" -Dserver.shutdown.grace-period="24s" -Djava.security.properties=/layers/paketo-buildpacks_bellsoft-liberica/java-security-properties/java-security.properties -XX:+ExitOnOutOfMemoryError -XX:ActiveProcessorCount=4 -XX:MaxDirectMemorySize=10M -Xmx12267527K -XX:MaxMetaspaceSize=133432K -XX:ReservedCodeCacheSize=240M -Xss1M -XX:+UnlockDiagnosticVMOptions -XX:NativeMemoryTracking=summary -XX:+PrintNMTStatistics -Dorg.springframework.cloud.bindings.boot.enable=true

  .   ____          _            __ _ _
 /¥¥ / ___'_ __ _ _(_)_ __  __ _ ¥ ¥ ¥ ¥
( ( )¥___ | '_ | '_| | '_ ¥/ _` | ¥ ¥ ¥ ¥
 ¥¥/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_¥__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.7)

👇👇👇👇👇👇👇👇👇👇
2022-05-17 17:04:26.678  INFO 1 --- [           main] .m.c.SpringApplicationContextInitializer : Found bindings [mongodb]
2022-05-17 17:04:26.693  INFO 1 --- [           main] .m.c.SpringApplicationContextInitializer : Setting service profile mongodb
👆👆👆👆👆👆👆👆👆👆
2022-05-17 17:04:26.706  INFO 1 --- [           main] o.c.samples.music.Application            : Starting Application using Java 11.0.15 on spring-music-00004-deployment-5dc6d99594-jfhrj with PID 1 (/workspace/BOOT-INF/classes started by cnb in /workspace)
2022-05-17 17:04:26.707  INFO 1 --- [           main] o.c.samples.music.Application            : The following 1 profile is active: "mongodb"
2022-05-17 17:04:26.873  INFO 1 --- [           main] .BindingSpecificEnvironmentPostProcessor : Creating binding-specific PropertySource from Kubernetes Service Bindings
2022-05-17 17:04:28.323  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Multiple Spring Data modules found, entering strict repository configuration mode!
2022-05-17 17:04:28.324  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data MongoDB repositories in DEFAULT mode.
2022-05-17 17:04:28.354  INFO 1 --- [           main] .RepositoryConfigurationExtensionSupport : Spring Data MongoDB - Could not safely identify store assignment for repository candidate interface org.cloudfoundry.samples.music.repositories.jpa.JpaAlbumRepository. If you want this repository to be a MongoDB repository, consider annotating your entities with one of these annotations: org.springframework.data.mongodb.core.mapping.Document (preferred), or consider extending one of the following types with your repository: org.springframework.data.mongodb.repository.MongoRepository.
2022-05-17 17:04:28.533  INFO 1 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 204 ms. Found 1 MongoDB repository interfaces.
2022-05-17 17:04:29.118  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-05-17 17:04:29.130  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-05-17 17:04:29.130  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.62]
2022-05-17 17:04:29.216  INFO 1 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-05-17 17:04:29.216  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2332 ms
2022-05-17 17:04:29.467  INFO 1 --- [           main] org.mongodb.driver.cluster               : Cluster created with settings {hosts=[music-db-mongodb:27017], mode=SINGLE, requiredClusterType=UNKNOWN, serverSelectionTimeout='30000 ms'}
👇👇👇👇👇👇👇👇👇👇
2022-05-17 17:04:29.575  INFO 1 --- [b-mongodb:27017] org.mongodb.driver.connection            : Opened connection [connectionId{localValue:2, serverValue:312}] to music-db-mongodb:27017
2022-05-17 17:04:29.576  INFO 1 --- [b-mongodb:27017] org.mongodb.driver.cluster               : Monitor thread successfully connected to server with description ServerDescription{address=music-db-mongodb:27017, type=STANDALONE, state=CONNECTED, ok=true, minWireVersion=0, maxWireVersion=13, maxDocumentSize=16777216, logicalSessionTimeoutMinutes=30, roundTripTimeNanos=37954237}
2022-05-17 17:04:29.575  INFO 1 --- [b-mongodb:27017] org.mongodb.driver.connection            : Opened connection [connectionId{localValue:1, serverValue:311}] to music-db-mongodb:27017
👆👆👆👆👆👆👆👆👆👆
2022-05-17 17:04:30.028  INFO 1 --- [           main] o.s.b.a.w.s.WelcomePageHandlerMapping    : Adding welcome page: class path resource [static/index.html]
2022-05-17 17:04:30.330  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-05-17 17:04:30.460  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8081 (http)
2022-05-17 17:04:30.461  INFO 1 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-05-17 17:04:30.461  INFO 1 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.62]
2022-05-17 17:04:30.478  INFO 1 --- [           main] o.a.c.c.C.[Tomcat-1].[localhost].[/]     : Initializing Spring embedded WebApplicationContext
2022-05-17 17:04:30.484  INFO 1 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 145 ms
2022-05-17 17:04:30.504  INFO 1 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 13 endpoint(s) beneath base path '/actuator'
2022-05-17 17:04:30.505  INFO 1 --- [nio-8080-exec-2] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-05-17 17:04:30.505  INFO 1 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-05-17 17:04:30.507  INFO 1 --- [nio-8080-exec-2] o.s.web.servlet.DispatcherServlet        : Completed initialization in 2 ms
2022-05-17 17:04:30.638  INFO 1 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8081 (http) with context path ''
2022-05-17 17:04:30.693  INFO 1 --- [           main] o.c.samples.music.Application            : Started Application in 4.818 seconds (JVM running for 5.796)
2022-05-17 17:04:30.934  INFO 1 --- [           main] org.mongodb.driver.connection            : Opened connection [connectionId{localValue:3, serverValue:313}] to music-db-mongodb:27017
2022-05-17 17:04:31.264  INFO 1 --- [nio-8081-exec-1] o.a.c.c.C.[Tomcat-1].[localhost].[/]     : Initializing Spring DispatcherServlet 'dispatcherServlet'
2022-05-17 17:04:31.265  INFO 1 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2022-05-17 17:04:31.266  INFO 1 --- [nio-8081-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 1 ms

```

### 動作確認

`https://spring-music-${NAMESPACE}.apps.jaguchi.maki.lol` にアクセス

右上のℹ️ボタンを押すとMongoDBがバインドされていることがわかる
<img width="2160" alt="image" src="https://user-images.githubusercontent.com/106908/168871963-b745c2a2-371e-4ab3-8491-8ec831a4746d.png">

Albumを追加

<img width="1024" alt="image" src="https://user-images.githubusercontent.com/106908/168528692-9ce6eed7-4c4f-43d9-b300-95af5bbee965.png">

```
kubectl delete pod -l app.kubernetes.io/component=run,app.kubernetes.io/part-of=spring-music -n ${NAMESPACE}
```

削除後もデータが残っていることを確認。


<img width="1024" alt="スクリーンショット 2022-05-16 15 02 49" src="https://user-images.githubusercontent.com/106908/168529101-9d716f8e-37d4-41fb-9cf7-fce847c01080.png">

### リソースの削除

不要になったら次のコマンドでWorkloadとデータベースを削除してください。

```
tanzu apps workload delete spring-music -n ${NAMESPACE} -y
kapp delete -n ${NAMESPACE} -a music-db -y
```