## 背景
Mtop是阿里巴巴集团的API网关，为集团内客户端、H5提供统一的API接入服务。新三消游戏《我就爱消除》初期在支付宝/淘宝等阿里内部平台上线，因此采用Mtop网关接入API服务。
随着游戏稳定运行，为扩大用户覆盖范围，需拓展至微信、抖音等外部平台。然而，由于微信/抖音等非阿里系平台无法直接访问Mtop服务，开发者需自行实现登录授权并申请集团开放权限，导致新平台上线流程复杂。同时，Mtop不支持独立APP或小米应用商店等小众平台平台接入。
此外，随着阿里集团“1+6+N”组织架构调整，各业务集团独立核算后需自行承担API调用成本。Mtop的按调用量计费机制导致服务成本随DAU增长呈线性上升趋势。
为应对多平台扩展需求及降本增效目标，需构建轻量级通用网关服务。
## 设计目标
+ **协议对齐**
    - 采用与Mtop协议完全兼容的交互规范，确保前端可通过配置化方式无缝切换网关
+ **安全与鉴权**
    - 请求鉴权：基于HMAC-SHA256的请求签名校验和时效性验证
+ **流量控制**
    - 提供全局/单机/单接口限流能力
    - 基于CPU水位动态调节请求，避免服务过载
+ **可观测性**
    1. 监控指标：实时实例健康度（CPU/内存/GC）、接口监控（QPS/P99/成功率）
    2. 链路追踪：全链路traceId透传
## 方案选型
<img width="960" alt="image" src="https://github.com/user-attachments/assets/f7eafab6-e4d9-4ea8-a457-632b41fa6e15" />

<font style="color:rgb(44, 44, 54);">多个项目共享同一 Maven 父项目，升级 Spring Boot 版本成本高且无计划。</font>
因此，为<font style="color:rgb(44, 44, 54);">兼容 Spring Boot 1.5 的 API 网关方案，短期内</font>我们选择自建API网关，技术方案采用异步Servlet和AsyncHttpClient。
当然，后续有升级jdk和SpringBoot版本时，我们也会将API网关切换为Spring Cloud Gateway。
## 架构
### 产品架构
![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/42512334/1747148520709-18747a4c-5735-4603-86ee-167ea576a6ae.jpeg)
## 功能设计
### 高可用设计
#### 异步方案
![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/42512334/1747232216804-a87ee77d-c76b-4dcc-816c-8c2db3feca75.jpeg)
<font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">API网关请求做了全异步化处理，请求通过Jetty IO线程异步提交到业务处理线程池，然后使用AsyncHttpClient调用调用业务服务，释放了由于网络等待引起的线程占用，使线程数不再成为网关的瓶颈。</font>
#### <font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">应用预热</font>
<font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">为了避免业务应用启动大量请求涌入导致的毛刺问题，我们加入了应用预热，具体逻辑如下：  
</font>
![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/42512334/1747233142199-fbe7980f-9698-439c-854b-0e9537c4af00.jpeg)
### 安全与鉴权
+ <font style="color:rgb(44, 44, 54);">防止未经授权的客户端伪装成合法调用方，API网关加入了基于HMAC-SHA256的请求签名校验</font>
+ <font style="color:rgb(44, 44, 54);">防止请求过期或被重复利用（重放攻击），客户端在请求中携带时间戳，服务端解密后验证时间戳 + 有效期是否超过当前时间</font>
### 可灰度
<font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">API网关作为请求入口，往往肩负着请求流量灰度验证的重任。</font>
<font style="color:rgb(51, 51, 51);background-color:rgb(253, 253, 253);">我们设计基于URL参数的灰度分离功能，携带特殊版本号的请求会请求到灰度机器</font>
![](https://cdn.nlark.com/yuque/0/2025/png/42512334/1747235612636-01509725-0be1-4a61-b7b4-76acd6bd2ae4.png)
### 接口限流
+ <font style="color:rgb(44, 44, 54);">接入sentinel限流功能，支持单机、接口、CPU使用率等级别流量控制，防止网关超载崩溃</font>
### 监控告警
+ <font style="color:rgb(44, 44, 54);">容器基础指标监控：CPU、内存、JVM、带宽等</font>
+ <font style="color:rgb(44, 44, 54);">客户端、网关、下游业务机指标监控：调用量、调用方式、响应时间、错误率等</font>
## 压测
机器：8核CPU、16GB内存、120GB磁盘
### apache bench压测单接口
QPS约为15000，CPU使用率稳定在95%
### 真实流量
QPS约为8000，CPU使用率稳定在90%


