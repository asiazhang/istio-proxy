# Istio Proxy

The Istio Proxy is a microservice proxy that can be used on the client and server side, and forms a microservice mesh.
It is based on [Envoy](http://envoyproxy.io) with the addition of several policy and telemetry extensions.

## 构建说明

一键式构建修改版本的istio-proxy `1.17.8`版本。

```bash
#!/bin/bash
set -ex

WORKDIR=$PWD
RELEASE=release-1.17
rm -rf istio-proxy/
git clone https://github.com/asiazhang/istio-proxy.git
cd istio-proxy
git checkout ${RELEASE}
git --no-pager log -1

mkdir -p /data/docker_bazel_cache
# make your changes to the source code
echo $PWD
docker run -it -w /work -v /data/docker_bazel_cache:/home/.cache/bazel -v $PWD:/work gcr.io/istio-testing/build-tools-proxy:${RELEASE}-latest bash -c "make build_envoy"

cp /data/docker_bazel_cache/_bazel_root/1e0bb3bee2d09d2e4ad3523530d3b40c/execroot/io_istio_proxy/bazel-out/aarch64-opt/bin/envoy $WORKDIR/envoy
file $WORKDIR/envoy

image_name=xxx/istio/proxyv2:1.17.8-enhance

cd $WORKDIR
docker build -t ${image_name} .
docker push ${image_name}
```

## 使用说明

增加了在Tracing的Span中额外记录http headers和http response的功能，方便定位。(仅适用于`release-1.17`版本)。

**1. 需要增加Envoy的http filter来写入额外数据。**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: log-body
  namespace: default # 请根据实际情况配置
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      patch:
        operation: INSERT_BEFORE
        value:
          name: cle-log-body.lua
          typed_config:
            '@type': type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
            inlineCode: |

              function envoy_on_request(request_handle)
                -- 检查是否是WebSocket连接，是则不处理
                local headers = request_handle:headers()
                local upgrade = headers:get("upgrade")
                if upgrade and upgrade:lower() == "websocket" then
                  return
                end

                request_handle:headers():remove("accept-encoding") # disable gzip
                local body = request_handle:body(true)
                local body_length = body:length()
                local bodyString = tostring(body:getBytes(0, body_length))
                request_handle:logWarn("Request body length: ".. body_length)
                request_handle:logWarn("Try adding HTTP request body to dynamic metadata.")
                request_handle:streamInfo():dynamicMetadata():set("cle.log.req.lua", "body", bodyString)
              end


              function envoy_on_response(response_handle)
                -- 检查是否是WebSocket连接，是则不处理
                local headers = request_handle:headers()
                local upgrade = headers:get("upgrade")
                if upgrade and upgrade:lower() == "websocket" then
                  return
                end

                local body = response_handle:body(true)
                local body_length = body:length()
                local bodyString = tostring(body:getBytes(0, body_length))
                response_handle:logWarn("Response body length: " .. body_length)
                response_handle:logWarn("Try adding HTTP response body to dynamic metadata.")
                response_handle:streamInfo():dynamicMetadata():set("cle.log.rsp.lua", "body", bodyString)
              end
```

往元数据中写入`cle.log.req.lua`和`cle.log.rsp.lua`数据，这样修改版本的Envoy才能正常上报包含http body信息的Span。

## 采样率

istio默认的采样率为1%，测试环境如果需要调整，那么可以使用如下配置，参考[istio官方文档](https://istio.io/latest/docs/tasks/observability/distributed-tracing/mesh-and-proxy-config/#customizing-trace-sampling)：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-control-plane
  namespace: istio-system
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100
```
