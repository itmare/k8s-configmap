Backend: spring boot의 application.properties
---------------------------------------------

-	docker image와 config를 decoupling하기 위함
-	대부분의 springboot 프로젝트는 변수들을 application.properties에 저장해서 사용한다.
-	container 이미지를 만들시, 해당 변수들이 동적으로 할당 되지 않으며, 환경에 따라 매번 이미지를 새로 만들어야 한다.
-	환경에 따른 변수들을 container 이미지와 함께 저장하지 않고, kubernetes에 저장하고 사용할 수 있는 configMap과 secrets 를 사용

-	기본 `configmap` 명령어

```shell
kubectl create configmap <map-name> <data-source>
```

-	`application.properties`을 읽어서 `data:`에 추가 후, configmap 생성

```shell
kubectl create configmap my-config --from-file=/deployments/config/application.properties
```

-	configmap 생성 확인

```shell
kubectl get configmap
kubectl get configmap my-config -o yaml   # 생성된 configmap을 yaml형식으로 확인

# 생성된 configmap output 예시
apiVersion: v1
data:
  application.properties: |
    db.ip= 34.64.0.0
    spring.datasource.url = jdbc:mysql://${db.ip}:3306/
    spring.datasource.username = hello
    spring.datasource.password = world
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-00T06:37:22Z"
  name: my-config
  namespace: my-namespace
  resourceVersion: "588650"
  selfLink: /api/v1/namespaces/my-namespace/configmaps/my-config
  uid: 0d8019bd-85f6-11ea-0000-42010ab0000
```

-	Dockerfile에 config 경로 설정

```shell
# 예시
FROM openjdk:8-jdk-alpine
EXPOSE 8080
ADD target/my-project.jar my-project.jar
ENTRYPOINT ["java", "-jar", "my-project.jar", "--spring.config.location=file:////deployments/config/application.properties"]
```

-	image 생성 및 푸시 (여기선 GCP Registry 사용)

```shell
# 예시
docker build -t gcr.io/<GCP_PROJECT_NAME>/my-img:v1 .
docker push gcr.io/<GCP_PROJECT_NAME>/my-img:v1
```

-	deployment 파일 생성 (기존 파일에 `volumeMounts`와 `volumes` 추가)

```shell
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bootsample
  labels:
    app: bootsample
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bootsample
  template:
    metadata:
        labels:
          app: bootsample
    spec:
      containers:
      - name: bootsample
        image: gcr.io/accu-platform/bootsample@sha256:c50f899c954ac2bb315a32e6325db580fcd2c06e627d0bad44e2500369aece2c
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: application-config # pod에 마운트할 volume 이름
          mountPath: "/deployments/config"  # 볼륨이 마운트될 pod 경로
          readOnly: true  # 읽기만 하던가 말던가
      volumes:
      - name: config-volume  # pod에 마운트할 volume 이름 설정
        configMap:
          name: my-config # 사용할 configmap
          items:
          - key: application.properties # item 이름 설정
            path: application.properties # volume에 마운트될 item
```

-	deployment 생성

```shell
kubectl create deployment.yaml
```

-	option) pod local로 curl을 통한 적용 확인

```shell
kubectl exec -it <POD_NAME> sh
apk add curl
curl localhost:8080
```

kubectl create configmap spring-app-config --from-file=src/main/resources/application.properties

kubectl exec -it <pod_name> sh apk add curl

You must create a ConfigMap before referencing it in a Pod specification If you reference a ConfigMap that doesn’t exist, the Pod won’t start.

java -jar easy-notes-1.0.0.jar --spring.config.location=/Users/jinwookchung/Downloads/ --spring.config.file=application-jw-db.properties

java -jar easy-notes-1.0.0.jar --spring.config.location=file:////Users/jinwookchung/IdeaProjects/spring-boot-mysql-rest-api-tutorial/src/main/resources/application-jw-db.properties

/Users/jinwookchung/IdeaProjects/spring-boot-mysql-rest-api-tutorial/src/main/resources/application-jw-db.properties

java -jar easy-notes-1.0.0.jar --spring.config.location=file:////deployments/config/application.properties

Config locations are searched in reverse order. By default, the configured locations are classpath:/,classpath:/config/,file:./,file:./config/. The resulting search order is the following:

1.	file:./config/
2.	file:./
3.	classpath:/config/
4.	classpath:/

.

.

.

.

.

.

.

.

.

.

Frontend: Angular의 environment.ts
----------------------------------

-	기본적으로 Angular는 docker build하면(angular 프로젝트 빌드 포함) environment 변경이 불가능하다.
-	간단한 angular app을 추가하여 environment를 docker image와 분리할 수 있다.
-	추가 파일
	-	environmentLoader.ts
	-	environment.json
-	수정 파일
	-	main.ts

<br><br>

#### environmentLoader.ts

-	src/environment 경로에 해당 파일 추가

#### environment.json

-	사용할 environment.ts를 json으로 변경
-	해당 json 파일을 src/assets에 생성

```shell
## 변경 예시

# environment.ts
export const environment =  Object.assign(CommonEnvironment, {
  production: true,
  webUrls: {
    main: {
      label: 'hello world',
      url: 'http://myservice/hello_world',
      inUse: true
    }
  }
});

# environment.json
{
  "production": true,
  "webUrls": {
    "main": {
      "label": "hello world",
      "url": "http://myservice/hello_world",
      "inUse": true
    }
  }
}
```

#### main.ts

-	json의 첫번째 level의 key를 추가
-	추가된 키에 대해서만 json파일의 값을 읽고, 없으면 기존 ts파일의 값을 읽는다.
-	변경 예시

```ts
// 추가
import { enableProdMode } from '@angular/core';
import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
import { AppModule } from './app/app.module';
import { environment } from './environments/environment';
import { environmentLoader as environmentLoaderPromise } from './environments/environmentLoader';

environmentLoaderPromise.then(env => {
  if (env.production) {
    enableProdMode();
  }

  // 사용할 level 1의 key값 추가
  environment.production = env.production;
  environment.webUrls = env.webUrls;


  platformBrowserDynamic().bootstrapModule(AppModule)
    .catch(err => console.log(err));
});
```

#### 변경 적용 테스트

-	`ng serve --host 0.0.0.0 --port 8080` 실행
-	localhost:8080 접속
-	src/assets/environment.json 값의 변경에 따라 적용되는지 확인
