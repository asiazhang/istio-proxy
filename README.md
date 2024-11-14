# Istio Proxy

The Istio Proxy is a microservice proxy that can be used on the client and server side, and forms a microservice mesh.
It is based on [Envoy](http://envoyproxy.io) with the addition of several policy and telemetry extensions.

## 使用说明

增加了在Tracing的Span中额外记录http headers和http response的功能，方便定位。(仅适用于`release-1.17`版本)。

**需要增加Envoy的http filter来写入额外数据。**

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
                local body = request_handle:body(true)
                local body_length = body:length()
                local bodyString = tostring(body:getBytes(0, body_length))
                # request_handle:logWarn("Request body length: ".. body_length)
                request_handle:logWarn("Try adding HTTP request body to dynamic metadata.")
                request_handle:streamInfo():dynamicMetadata():set("cle.log.req.lua", "body", bodyString)
              end


              function envoy_on_response(response_handle)
                local body = response_handle:body(true)
                local body_length = body:length()
                local bodyString = tostring(body:getBytes(0, body_length))
                # response_handle:logWarn("Response body length: " .. body_length)
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
