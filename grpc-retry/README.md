# gRPC Retry

This is an example of gRPC retry using Envoy Gateway. The fault injection in the fake-service is configured to return an error with a certain probability. Envoy Gateway will retry the request if an error occurs.

## Setup

Install Envoy Gateway in your cluster. Make sure the `envoyproxies.gateway` CRD is installed.

```sh
kubectl get crd | grep envoyproxies.gateway
```

If it is not installed, install it using the following command:

```shell
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.3.0 -n envoy-gateway-system --create-namespace
```

Deploy the sample gRPC application.

```shell
kubectl apply -f https://raw.githubusercontent.com/Nishikoh/envoy-sandbox/refs/heads/main/grpc-retry/grpc-routing.yaml
```

## Verifying Retry Behavior

Verify the gRPC retry behavior in Envoy Gateway.

```shell
export GATEWAY_HOST=$(kubectl get gateway/eg-example -o jsonpath='{.status.addresses[0].value}')
```

```shell
grpcurl -plaintext -authority=grpc-example.com ${GATEWAY_HOST}:80 FakeService.Handle
```

The result will look like the following:

```json
{
  "Message": "{\n  \"name\": \"Service\",\n  \"type\": \"gRPC\",\n  \"ip_addresses\": [\n    \"x.y.z.32\"\n  ],\n  \"start_time\": \"2025-02-19T06:06:12.655413\",\n  \"end_time\": \"2025-02-19T06:06:12.655454\",\n  \"duration\": \"40.876µs\",\n  \"body\": \"Hello World\",\n  \"code\": 0\n}\n"
}
```

```sh
kubectl logs svc/fake -f
```

When viewing the logs with the above command, you will see that an error occurred due to the `error_injector`. From the same `x-request-id` value, it is clear that Envoy Gateway retried the request until it succeeded.

```log
2025-02-19T06:06:12.653Z [INFO]  Handling request gRPC request:
  context=
  | key: :authority value: [grpc-example.com]
  | key: content-type value: [application/grpc]
  | key: grpc-accept-encoding value: [gzip]
  | key: x-forwarded-for value: [xx.yy.zz.1]
  | key: user-agent value: [grpcurl/1.9.2 grpc-go/1.61.0]
  | key: x-forwarded-proto value: [http]
  | key: x-envoy-external-address value: [xx.yy.zz.1]
  | key: x-request-id value: [b93fa578-5291-4023-9f2a-63feca3d07ea]

2025-02-19T06:06:12.653Z [INFO]  error_injector: Injecting error: request_count=2 error_percentage=0.5 error_type=http_error
2025-02-19T06:06:12.653Z [ERROR] Error handling request: error="Service error automatically injected"
2025-02-19T06:06:12.653Z [INFO]  Finished handling request: duration="125.503µs"
2025-02-19T06:06:12.655Z [INFO]  Handling request gRPC request:
  context=
  | key: user-agent value: [grpcurl/1.9.2 grpc-go/1.61.0]
  | key: grpc-accept-encoding value: [gzip]
  | key: x-request-id value: [b93fa578-5291-4023-9f2a-63feca3d07ea]
  | key: :authority value: [grpc-example.com]
  | key: x-forwarded-for value: [xx.yy.zz.1]
  | key: x-forwarded-proto value: [http]
  | key: x-envoy-external-address value: [xx.yy.zz.1]
  | key: content-type value: [application/grpc]

2025-02-19T06:06:12.655Z [INFO]  Finished handling request: duration="52.877µs"
```

## Verifying Access from the Same Namespace

When accessing Envoy Gateway from the same namespace (default) to the fake service, use the following command. This sends a request to the Envoy Gateway service via `grpcurl`.

```shell
kubectl run grpcurl --image=fullstorydev/grpcurl:latest --namespace default -it --rm -- -plaintext -authority=grpc-example.com grpc-gateway.envoy-gateway-system.svc.cluster.local:80 FakeService.Handle
```

By checking the client logs, you can confirm that the communication was successful.

```shell
kubectl logs grpcurl
```

```json
{
  "Message": "{\n  \"name\": \"Service\",\n  \"type\": \"gRPC\",\n  \"ip_addresses\": [\n    \"xxx.yyy.zzz.34\"\n  ],\n  \"start_time\": \"2025-02-20T07:55:33.765312\",\n  \"end_time\": \"2025-02-20T07:55:33.765457\",\n  \"duration\": \"145.169µs\",\n  \"body\": \"Hello World\",\n  \"code\": 0\n}\n"
}
```

By checking the server logs, you can see that after a failure, the request is retried with the same `x-request-id`.

```log
fake-74cb56bd56-kfx9h grpcsrv 2025-02-20T07:58:17.985Z [INFO]  Handling request gRPC request:
fake-74cb56bd56-kfx9h grpcsrv   context=
fake-74cb56bd56-kfx9h grpcsrv   | key: x-forwarded-for value: [xxx.yyy.zzz.42]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-forwarded-proto value: [http]
fake-74cb56bd56-kfx9h grpcsrv   | key: :authority value: [grpc-example.com]
fake-74cb56bd56-kfx9h grpcsrv   | key: content-type value: [application/grpc]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-envoy-external-address value: [xxx.yyy.zzz.42]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-request-id value: [6e8c6e70-0133-475c-a070-6aeb3638de5c]
fake-74cb56bd56-kfx9h grpcsrv   | key: user-agent value: [grpcurl/v1.9.2 grpc-go/1.61.0]
fake-74cb56bd56-kfx9h grpcsrv   | key: grpc-accept-encoding value: [gzip]
fake-74cb56bd56-kfx9h grpcsrv
fake-74cb56bd56-kfx9h grpcsrv 2025-02-20T07:58:17.985Z [INFO]  error_injector: Injecting error: request_count=36 error_percentage=0.5 error_type=http_error
fake-74cb56bd56-kfx9h grpcsrv 2025-02-20T07:58:17.986Z [ERROR] Error handling request: error="Service error automatically injected"
fake-74cb56bd56-kfx9h grpcsrv 2025-02-20T07:58:17.986Z [INFO]  Finished handling request: duration=1.097642ms
fake-74cb56bd56-kfx9h grpcsrv 2025-02-20T07:58:18.052Z [INFO]  Handling request gRPC request:
fake-74cb56bd56-kfx9h grpcsrv   context=
fake-74cb56bd56-kfx9h grpcsrv   | key: content-type value: [application/grpc]
fake-74cb56bd56-kfx9h grpcsrv   | key: grpc-accept-encoding value: [gzip]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-request-id value: [6e8c6e70-0133-475c-a070-6aeb3638de5c]
fake-74cb56bd56-kfx9h grpcsrv   | key: user-agent value: [grpcurl/v1.9.2 grpc-go/1.61.0]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-forwarded-for value: [xxx.yyy.zzz.42]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-forwarded-proto value: [http]
fake-74cb56bd56-kfx9h grpcsrv   | key: x-envoy-external-address value: [xxx.yyy.zzz.42]
fake-74cb56bd56-kfx9h grpcsrv   | key: :authority value: [grpc-example.com]
fake-74cb56bd56-kfx9h grpcsrv
fake-74cb56bd56-kfx9h grpcsrv 2025-02-20T07:58:18.052Z [INFO]  Finished handling request: duration="210.211µs"
```
