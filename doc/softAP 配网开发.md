## 操作场景
设备通过 softAP 方式创建一个 Wi-Fi 热点，手机连接该热点，再通过数据通道例如 TCP/UDP 通讯，将目标 Wi-Fi 路由器的 SSID/PSW 传递该设备，设备获取后，即可连接 Wi-Fi 路由器从而连接互联网。同时，为了对设备进行绑定，手机 App 可以利用该 TCP/UDP 数据通道，将后台提供的配网 Token 发送给设备，并由设备转发至物联网后台，依据 Token 可以进行设备绑定。本文档主要指导您如何使用softAP 方式配网开发。

腾讯连连小程序已经支持 softAP 配网，并提供了相应的 [小程序 SDK](https://www.npmjs.com/package/qcloud-iotexplorer-appdev-sdk)。
基于 Token 的 softAP 方式配网及设备绑定的示例流程图，如下图所示：
![](https://main.qcloudimg.com/raw/5e30af733894a2fc1e5c36df817b5c51.png)

##  softAP 配网步骤
1. 腾讯连连小程序进入配网模式后，则可以在物联网开发平台服务获取到当次配网的 Token。小程序相关操作可以参考 [生成 Wi-Fi 设备配网 Token](https://cloud.tencent.com/document/product/1081/44044)。
2. 使 Wi-Fi 设备进入 softAP 配网模式，若设备有指示灯在快闪，则说明进入配网模式成功。    
3. 小程序按照提示依次获取 Wi-Fi 列表，输入家里目标路由器的 SSID/PSW，再选择设备 softAP 热点的 SSID/PSW。
4. 手机连接设备 softAP 热点成功后，小程序作为 UDP 客户端会连接 Wi-Fi 设备上面的 UDP 服务（默认 IP 为**192.168.4.1**，端口为**8266**）。
5. 小程序给设备 UDP 服务，发送目标 Wi-Fi 路由器的 SSID/PSW 以及配网 Token，JSON 格式为：
```json
   {"cmdType":1,"ssid":"Home-WiFi","password":"abcd1234","token":"6aa11111x1x123x1aa546xx6x111xxxd"} 
```
发送完成后，等待设备 UDP 回复设备信息及配网协议版本号：
```json  
   {"cmdType":2,"productId":"OSPB5ASRWT","deviceName":"dev_01","protoVersion":"2.0"}
```
6. 如果2秒之内，未收到设备回复，则重复步骤5，UDP 客户端重复发送目标 Wi-Fi 路由器的 SSID/PSW 及配网 Token。（如果重复发送5次，都没有收到回复，则认为配网失败，Wi-Fi 设备有异常）      
7. 如果步骤5收到设备回复，则说明设备端已收到 Wi-Fi 路由器的 SSID/PSW 及 Token，正在连接 Wi-Fi 路由器，并上报 Token。此时小程序会提示手机也将连接 Wi-Fi 路由器，并通过 Token 轮询物联网后台，来确认配网及设备绑定是否成功。小程序相关操作可以参考 [查询配网Token状态](https://cloud.tencent.com/document/product/1081/44045)。
8. 设备端在成功连接 Wi-Fi 路由器后，需要通过 MQTT 连接物联网后台，并将小程序发送的配网 Token，通过下面 MQTT 报文上报给后台服务：
```json
    topic: $thing/up/service/ProductID/DeviceName
    payload: {"method":"app_bind_token","clientToken":"client-1234","params": {"token":"6xx12345x9x1234ee777xx6e528a0fd"}}
```
设备端也可以通过订阅主题 $thing/down/service/ProductID/DeviceName 来获取 Token 上报的结果。
9. 在以上5 - 7步骤中，需观察以下情况：
  - 如果小程序收到设备 UDP 服务发送过来的错误日志，且 deviceReply 字段的值为"Current_Error"，则表示当前配网绑定过程中出错，需要退出配网操作。
  - 如果 deviceReply 字段是"Previous_Error"，则为上一次配网的出错日志，只需要上报，不影响当此操作。
错误日志 JSON 格式，示例如下：
```json
{"cmdType":2,"deviceReply":"Current_Error","log":"RTK WIFI connect error! (10, 2)"} 
```
10. 如果设备成功上报了 Token，物联网后台服务已确认 Token 有效性，小程序会提示配网完成，设备添加成功。
11. 设备端会记录配网的详细日志，如果配网或者添加设备失败，可以让设备端创建一个特殊的 softAP 和 UDP 服务，通过小程序可以从设备端获取更多日志用于错误分析。

>!UDP 相比 TCP 是不可靠的通讯，存在丢包的可能，特别在比较嘈杂的无线 Wi-Fi 环境中，丢包率会比较大。为了保证小程序和设备之间的数据交互是可靠的，需要在应用层设计一些应答以及超时重发的机制。

### 基于Ambz2 SDK使用腾讯云 softAP 配网协议
腾讯云softAP配网协议基于Ambz2 SDK的实现参见 GitHub 工程 [qcloud-iot-rtk-wifi-based-ambz2](https://github.com/tencentyun/qcloud-iot-rtk-wifi-based-ambz2.git)。

#### 配网代码示例
在 qcloud-iot-rtk-wifi-based-ambz2\component\common\example\qcloud_iot_c_sdk\wifi_config 目录下，提供了 softAP 配网 v2.0 在 Ambz2 SDK 上面的参考实现，配网接口说明请查看 wifi_config/qcloud_wifi_config.h。

配网框架对softAP配网做了封装，开发者只要调用API启动softAP配网，然后在配网结果回调中判断配网结果即可。`qcloud_demo_task` 中示例了softAP配网的使用：
```c
static void qcloud_demo_task(void *arg)
{
    int ret;

    set_wifi_config_result(false);
    while (!wifi_is_up(RTW_STA_INTERFACE)) {
        HAL_SleepMs(1000);
    }
#if WIFI_PROV_SOFT_AP_ENABLE
    Log_d("start softAP wifi provision");
    eSoftApConfigParams apConf = {"RTK8720-SAP", "12345678", 6};
    ret = qiot_wifi_config_start(WIFI_CONFIG_TYPE_SOFT_AP, &apConf, _wifi_config_result_cb);
#elif WIFI_PROV_SIMPLE_CONFIG_ENABLE
    Log_d("start simple config wifi provision");
    ret = qiot_wifi_config_start(WIFI_CONFIG_TYPE_SIMPLE_CONFIG, NULL, _wifi_config_result_cb);
#else
    Log_e("not supported wifi provision method");
    ret = -1;
#endif
    if (ret) {
        Log_e("start wifi config failed: %d", ret);
        goto exit;
    }
}		

```

使能宏定义 `WIFI_PROV_SOFT_AP_ENABLE`， 调用配网接口`qiot_wifi_config_start`，传入softAP配网模式、softAP的热点名及密码、配网结果回调，则配网结果回调函数中会返回配网的结果。

#### 代码设计说明
配网代码将核心逻辑与平台相关底层操作分离，便于移植到不同的硬件设备上。

| 代码 | 设计说明 |
|---------|---------|
| `qcloud_wifi_config.c` | 配网框架，统一各种配网方式，实现配网启动、配网停止、配网结果回调，用户只需要将特定配网方法注册到`sg_wifi_config_methods`即可，不依赖任何软硬件平台。 |
|`qiot_comm_service.c`|配网相关接口实现，实现 UDP 服务和与小程序的数据交互，主要依赖腾讯云物联网 C-SDK 及 FreeRTOS/lwIP 运行环境。|
|`qiot_device_bind.c`|配网相关接口实现，实现配网过程的token交互与设备绑定，主要依赖腾讯云物联网 C-SDK 及 FreeRTOS/lwIP 运行环境|
|`rtk_soft_ap.c`|softAP Wi-Fi 操作相关接口实现，依赖于 Ambz2 SDK提供的WiFi相关操作接口，当使用其他硬件平台时，可以参考移植适配。|
|`rtk_simple_config.c`|simpleConfig Wi-Fi 操作相关接口实现，依赖于 Ambz2 SDK提供的WiFi相关操作接口，当使用其他硬件平台时，可以参考移植适配。|
|`wifi_config_error_handle.c`|设备错误日志处理，主要依赖于 FreeRTOS。|
|`wifi_config_log_handle.c`|设备配网日志收集和上报，主要依赖于 FreeRTOS。|
