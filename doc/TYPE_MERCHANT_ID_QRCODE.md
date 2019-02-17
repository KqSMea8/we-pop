
<html>
<head>
<style>
*
{
   font-family: 仿宋;
}

@media screen {
  div.divFooter {
    display: none;
  }
}
@media print {
  div.divFooter {
    position: fixed;
    bottom: 0;
  }
}
</style>
<!--区块链跨境钱包发卡行前置SDK使用文档-->
</head>
<body>
</body>
</html>


```mermaid

%% 主扫
sequenceDiagram
    participant 聚合支付服务商放置在商户处的二维码
    participant 聚合支付服务商后台系统
    participant BOCHK APP
    participant BOCHK后台系统
    participant 区块链节点
    participant WeBank收单后台系统(ATS)

    BOCHK APP->>聚合支付服务商放置在商户处的二维码: 扫描商户的收款二维码
    BOCHK APP->>BOCHK APP: 解析出二维码代表的字符串
    BOCHK APP->>BOCHK后台系统: 透传扫码信息
    BOCHK后台系统->>WeBank收单后台系统(ATS): 提交扫码信息（链上链下）
    WeBank收单后台系统(ATS)->>WeBank收单后台系统(ATS): 解析聚合支付服务商的二维码，获取到商户在聚合支付服务商的商户id
    WeBank收单后台系统(ATS)->>WeBank收单后台系统(ATS): 通过商户在聚合支付服务商的商户id查到商户在WeBank的商户id，验证商户合法性

    WeBank收单后台系统(ATS)-->>BOCHK后台系统: 返回商户信息（商户id，商户名称等）
    Note over WeBank收单后台系统(ATS),BOCHK后台系统: 平均延时：161.349ms（跨境）
    BOCHK后台系统-->>BOCHK APP: 返回
    BOCHK APP->>BOCHK APP: 展示商户信息
    BOCHK APP->>BOCHK APP: 用户输入金额

    %% 是否需要？
    BOCHK APP->>BOCHK后台系统: 点击确认支付按钮
    BOCHK后台系统->>BOCHK后台系统: 生成BOCHK订单
    BOCHK后台系统->>WeBank收单后台系统(ATS): 请求生成WeBank订单（带上BOCHK订单id,走链上链下）
    WeBank收单后台系统(ATS)->>WeBank收单后台系统(ATS): 生成WeBank订单（会映射聚合支付服务商订单id和BOCHK订单id）
    WeBank收单后台系统(ATS)->>聚合支付服务商后台系统: 下单，请求生成聚合支付服务商的订单
    聚合支付服务商后台系统->>聚合支付服务商后台系统: 生成聚合支付服务商的订单
    聚合支付服务商后台系统-->>WeBank收单后台系统(ATS): 返回聚合支付服务商的订单id
    WeBank收单后台系统(ATS)-->>BOCHK后台系统: 返回WeBank订单id,聚合支付服务商订单id
    Note over WeBank收单后台系统(ATS),BOCHK后台系统: 平均延时：281.744ms（跨境）
    BOCHK后台系统-->>BOCHK APP: 返回

    BOCHK APP->>BOCHK APP: 输入支付密码

    BOCHK APP->>BOCHK后台系统: 发起消费请求
    BOCHK后台系统->>BOCHK后台系统: 请求鉴权
    BOCHK后台系统->>BOCHK后台系统: 验证支付密码

    Note right of BOCHK后台系统: 下面的流程“主扫”“被扫”完全一样

    %%
    BOCHK后台系统->>区块链节点: 调用SDK汇率换算函数(Fx api)
    区块链节点-->>BOCHK后台系统: 返回汇率
    Note over 区块链节点,BOCHK后台系统: 平均延时：35.5801ms（非跨境）
    BOCHK后台系统->>BOCHK后台系统: 记减用户HKD
    BOCHK后台系统->>WeBank收单后台系统(ATS): 调用记账接口（链上链下）

    WeBank收单后台系统(ATS)->>区块链节点: 调用区块链记账接口
    区块链节点->>区块链节点: 调用清算行合约：合约记账
    区块链节点-->>WeBank收单后台系统(ATS): 返回
    WeBank收单后台系统(ATS)-->>BOCHK后台系统: 返回记账结果: 成功，失败，未知
    Note over WeBank收单后台系统(ATS),BOCHK后台系统: 平均延时：3462.83ms / 1856.36ms（跨境）
    Note over WeBank收单后台系统(ATS),BOCHK后台系统: WeBank与BOCHK交互总平均延时：3778ms
    opt WeBank返回超时
        BOCHK后台系统->>WeBank收单后台系统(ATS): 调用查询API,查询订单状态
    end
    alt 失败
        BOCHK后台系统->>BOCHK后台系统: 需要对用户扣掉的HKD进行冲正
    else 未知
        BOCHK后台系统->>BOCHK后台系统: 不做处理，对账的时候处理这种情况
    end
    BOCHK后台系统->>BOCHK后台系统: 更新订单状态
    BOCHK后台系统-->>BOCHK APP: 返回

    opt BOCHK后台系统返回超时
      BOCHK APP->>BOCHK后台系统: 查询支付结果
      BOCHK后台系统->>BOCHK后台系统: 查询支付结果
      BOCHK后台系统-->>BOCHK APP: 返回支付结果
    end

    WeBank收单后台系统(ATS)->>聚合支付服务商后台系统: 通知聚合支付服务商发货（会多次重试）
    聚合支付服务商后台系统->>聚合支付服务商后台系统: 通知商户发货
    聚合支付服务商后台系统-->>WeBank收单后台系统(ATS): 返回（重复的请求依然需要给WeBank回包）

    opt 未收到WeBank发起的发货通知，当用户询问商户是否收到款时，由商户后台人工触发请求
        聚合支付服务商后台系统->>WeBank收单后台系统(ATS): 调用查询订单API
        WeBank收单后台系统(ATS)-->>聚合支付服务商后台系统: 返回订单状态
    end

    聚合支付服务商后台系统->>聚合支付服务商后台系统: 更新订单状态

```
