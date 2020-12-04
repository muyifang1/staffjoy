# 笔记 #

## 单体仓库 和 多仓库 ##
- Multi and Mono

## 微服务接口参数校验 ##
- Spring框架 支持JSR-303和JSR-380 开发人员只需要对需要校验的参数添加注解
1. 在控制器接口参数校验 例如： @NotBlank和@Min(0)
`public GenericAccountResponse getAccount(@RequestParam @NotBlank String userId){...}`
`public ListAccountResponse listAccounts(@RequestParam int offset, @RequestParam @Min(0) int limit){...}`
`public GenericAccountResponse updateAccount(@RequestBody @Valid AccountDto newAccountDto){...}`
2. 自定义注解校验
`public GenericAccountResponse getAccountByPhonenumber(@RequestParam @PhoneNumber String phoneNumber){...}`

## 统一异常处理 ##
1. RESTful 异常 底层使用 `@RestControllerAdvice`
  - 封装一个 `GlobalExceptionTranslator`自定义 `handleError` 拦截不同级别的异常 例如： `@ExceptionHandler(Throwable.class)`
  - 处理时先输出log，然后返回值统一封装一个 `BaseResponse` 内部包含http状态Code
2. Web MVC 异常 底层使用 `ErrorController`
  - 封装一个 `GlobalErrorController` 实现 `ErrorController` 接口
  - 处理时先解析StatusCode，输入log，然后跳转到统一Error页面

## DTO 和 DMO 相互转换 ##
- DTO Data Transfer Object 用于通信，api 输入输出
- DMO Data Model Object 用于持久化，需要`@Set` `@Get`
- 业界使用modelmapper自动映射工具进行convertTo。在Service层进行互转
`https://github.com/modelmapper/modelmapper`

## 强类型 VS 弱类型 ##
* GRPC 强类型 接口规范，有严格服务契约 contract概念
    - 优点：自动代码生成，自动编码解码，编译器自动类型检查。
    - 缺点：服务器端和客户端强耦合。测试不方便
* REST服务 弱类型
    - 优点：开发测试方便，Json作为传输消息，http作为传输协议
* Spring Feign底层原理是动态代理
  1. 在API Interfacece层面拦截（可以给一个强类型java api接口）
  2. 通过 Encoder 完成RequestBean到RequestJson之间转换 (亦或Decoder 完成 ResponseJson 到 ResponseBean转换)
  3. 可以根据Interceptor进行截获加工处理
  4. 再传递给Http Client
* StaffJoy中使用 Spring Feign与Restful 组成强弱结合
* XXXapi 接口模块中强类型的java api接口 包括dto，他们被Spring Feign引用拼装出强类型客户端
* 异常响应返回 BaseResponse , 正常响应返回 ListAccountResponse这个类继承BaseResponse

## Swagger 接口文档 ##
1. pom中引入 swagger依赖
```
<!-- Swagger -->
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.9.2</version>
</dependency>
```
2. 代码中 加入注解 @EnableSwagger2
```
@EnableSwagger2
public class SwaggerConfig {
```

# Charpter4 微服务网关设计 #
* BFF概念 Backend for Frontend 前端人员开发，代理适配服务，为前端开发的后端服务
* 以下是 SOA V4 分层 (根据项目规模小公司网关层和BFF层不分离)
1. 用户体验层：第三方应用、H5前后分离应用、无线应用、浏览器
2. 网关层：   开放平台网关、H5网关     、 无线网关、Web应用网关
3. BFF层：    开放平台BFF层、 H5 BFF层、  无线BFF层、服务器端Web应用
4. 微服务层： 各个业务服务

## 自主搭建一个网关 Faraday 内核设计 ##
1. 路由解析模块
2. 路由映射表模块
3. HttpClient 映射表模块
4. 请求转发模块
5. 截获器模块(包含请求截获器和响应截获器)

## 生产级别网关需要考虑哪些环节 ##
1. 限流熔断
2. 动态路由和负载均衡
3. 基于Path的路由
    - api.xxx.com/pathx
4. 截获器链
5. 日志采集和Metrics埋点
6. 响应流优化

# Charpter5 安全相关架构 #

## 认证授权安全演进 ##
1. 单机 服务器存储Session并设定超时时间，浏览器缓存Session
2. 粘性Session (Sticky Session)集群部署，Ngix 确保相同session id 分发到上一次ServerIP
  - 缺点：session id 深度耦合在某一台服务器上
    - 解决方案一：会话同步复制(再几个服务器之间复制Session)
    - 解决方法二：无状态会话，session不存在服务器上 而是存在浏览器上。
      * 缺点：在浏览器上存储有泄漏风险，需要加密，而且缓存大小往往4k上限。
    - 解决方案三：集中状态会话，session集中存储在redis缓存上。
## 微服务认证授权挑战 ##
1. 挑战一：前端用户体验形式众多，后台微服务也众多
2. 挑战二：单点登录 SSO
* Auth Service + Token 来实现，独立数据库UserDB
  - 登录认证，会话管理，令牌颁发，校验等职责
  - 引入令牌Token概念，Token本身不包含数据，是一个随机引用标识符，各个服务凭此到Auth Service校验
  - 前端应用 和 后台服务之间 传递Token,后台服务拿token去Auth Service 换取UserInfo
  - 缺点：代码侵入到每一个微服务中，每个微服务都需要与Auth Service交互
* Auth 3.5 版本 Token + Gateway
  - 网关替后台微服务与Auth Service交互，换取userinfo 网关传递到后台。
  - 优点：安全性高
  - 缺点：Auth Service 压力比较大，未来可能是瓶颈，相对重量级。
* Auth 3.6 版本 JWT + Gateway 根据无状态会话延展出来，应用场景安全性要求低
  - 客户端通过Auth service 取到 JWT令牌，并存储在客户端
  - 客户端把JWT令牌传递给网关，网关解析 JWT令牌(其中包含用户信息)

## JWT 原理 base64编码 ##
  - Json Web Token 紧凑自包含的json格式，信息经过数字签名，可校验、可确认。
  - 多用于 认证授权，信息交换等场景。
  - JWT令牌结构 `[Header].[Payload].[Signature]`
  `base64Url(Header) + "." + base64Url(Payload) + "." + base64Url(Signature)`
  - 信息是公开的，任何人都可以在 jwt.io 网站解析信息。
  - 但是可以保证可依赖且不可篡改。通过Signature实现

## HMAC流程 前提：Auth Server 和 Resource Server 之前提前约定secret作为校验密钥 ##
 1. Client客户端 用户登录Auth Server
 2. Auth Server 认证办法JWT令牌给 Client 客户端本地存储
 3. 客户端 携带JWT调用API访问Resource Server
 4. Resource Server根据和Auth Server约定好的secret 校验是否被篡改，并解析出用户信息进行权鉴
## RSA流程 和上述一直，加密方式不同，AuthServer保存privatekey私钥, ResourceServer保存公钥 ##
+ 优点：
    1. 紧凑轻量
    2. 对AuthServer压力小，简化AuthServer实现。无需对用户会话状态维护管理。
+ 缺点：无状态和吊销无法两全。假如被入侵，只有等该令牌失效，别无他法。

## 服务之间调用鉴权 ##
- 服务间调用授权截获器 `common/AuthorizeInterceptor`
- 自定义一个注解`Authorize`
- 用户角色和环境鉴权 `account-svc/AccountController#validateAuthenticatedUser`和 `validateEnv()`
- 授权Header定义 `common-lib/AuthConstant`

# Apollo vs Spring Cloud Config vs K8s ConfigMap #

# 调用链监控产品 CAT vs Zipkin vs Skywalking #

## EFK + K8s ##
- EFK Elastic Fluentd Kibana
  > K8s 中Docker产生log信息 -> fluentd 收集信息 -> Kafka队列解耦 -> 自定义LogParser -> elastic存储 -> kibana前端画面展示

- SkyWalking + K8s

# Dockerfile镜像构建 #
> Account服务Dockerfile
```
FROM java:8-jdk-alpine
COPY ./target/account-svc-1.0.0.jar /usr/app/
WORKDIR /usr/app
ENTRYPOINT ["java", "-jar", "account-svc-1.0.0.jar"]
```
> MyAccount单页应用Dockerfile
```
FROM node:alpine as builder
WORKDIR '/build'
COPY myaccount ./myaccount
COPY resources ./resources
COPY third_party ./third_party

WORKDIR '/build/myaccount'

RUN npm install
RUN npm rebuild node-sass
RUN npm run build

RUN ls /build/myaccount/dist

FROM nginx
EXPOSE 80
COPY --from=builder /build/myaccount/dist /usr/share/nginx/html
```

## Kubernetes 基本概念 ##
1. Cluster 集群 超大计算机抽象，由节点组成。
2. Container 容器 应用居住和运行在容器中。
3. POD 这是K8s的基本调度单位
    - 一个POD可以有一个或者多个Container内部共享文件系统和网络。
    - 每个POD有独立ip,POD中容器共享这个IP和端口空间。
    - 大部分时候一个POD只放一个容器
4. ReplicaSet 副本集 支持无状态应用
    - 在实际运行中根据监控信息按照配置动态增加或者减少POD
    - 一个应用发布时不会单独发一个POD，为了实现高可用发布一组
    - 通过模板 yml 或者 Json 来规范容器的端口、镜像、副本数量、健康检查机制等
5. Service 这是k8s抽象出的一个概念，可以理解成 gateway，它底层实现寻址和负载均衡
    - 创建和管理POD，支持无状态应用
6. Deployment 发布 用来管理ReplicaSet 实现高级的发布机制策略，如金丝雀，蓝绿
7. ConfigMap / Secrets 应用配置，secret敏感数据配置
8. DaemonSet 保证每个节点有且仅有一个POD，常见于监控。
9. StatefulSet 类似ReplicaSet，但支持有状态应用。
10. Job 运行一次就结束的任务。
11. CronJob 周期性运行的任务。
12. Volume 可装载磁盘文件存储。
13. PersisentVolume / PersistenVlumeClaims 超大磁盘存储抽象和分配机制
14. Label / Selector 资源打标签和定位机制
15. Namespace 资源逻辑隔离机制
16. Readiness Probe 就绪探针，流量接入Pod判断依据
17. Liveness Probe 存活探针，是否kill pod的判断依据

## 理解 Kubernetes 节点网络 和 POD网络 ##
| |作用|实现|
|:-|:-:|-:|
|节点网络|Master/Worker节点之间网络互通|路由器，交换机，网卡|
|Pod网络|Pod之间互通|虚拟网卡，虚拟网桥，路由器|
|Service网络|屏蔽Pod 地址变化 + 负载均衡|Kube-proxy,Netfilter,Api-Server,DNS|
|NodePort|将Service暴露在节点网络上|Kube-proxy + Netfliter|
|LoadBalancer|将Service暴露在公网上+负载均衡|公有云LB + NodePort|
|Ingress|反向路由，安全，日志监控|Nginx/Envoy/Traefik/Faraday|
