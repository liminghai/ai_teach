#  PDPTooltips增加商品对比分支

_**原语雀文档链接：**_[https://yuque.antfin.com/lz37ik/vnpghp/uiw5guiqeqbh15h9](https://yuque.antfin.com/lz37ik/vnpghp/uiw5guiqeqbh15h9)

---
:::
**排期节奏**

​
:::
:::
[Lazzie迭代需求list](https://aliyuque.antfin.com/lz37ik/cpxybq/zglzamnv3gaatg4r?singleDoc#0Zmi) **《Lazzie迭代需求list》**
:::

**更新记录**

| **更新时间** | **更新人** | **更新内容** |
| --- | --- | --- |
| 11.11 | xx/xx同学 | prd完成 |
|  |  |  |

## 1.1.需求背景

突破单场景转化瓶颈，跨场景实现"1+1>2"的流转效率，为用户提供「场景感知型导购」体验，基于用户全站连续行为，叠加过往的被动信息输入（如历史行为或用户静态信息）进行复杂推理分析，实时判断用户意图后在当前场景。

**之前已经接入帮我凑、帮我算、帮我选，本次迭代增加帮我比。**

用户在浏览商品详情页时，常会有“对比其他商品”的潜在意图（如看了多款耳机/洗面奶/手机）。当前行为路径通常是：

> 商详页 → 返回列表页 → 筛选/搜索 → 打开另一个商品 → 手动对比

该路径繁琐，跳出率高。我们希望帮助用户快速完成对比决策，减少跳出、提升成交。

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/eYVOL5j400ZYAlpz/img/4ca51206-35c3-4bcd-8769-0cbce1457aed.png)

## 1.2.需求内容

![Frame 2117131004 (2).png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/1f4fc68c-034d-4025-acc6-5399db4d74e1.png)

| 模块 | 内容 | 说明 |
| --- | --- | --- |
| 数据输入 | 商品属性、参数、卖点、价格、评价 | 由商品底层数据-odps表提供 |
| 特征提取 | 抽取差异性字段 | 自动筛选主要差异字段（如功率/成分） |
| 权重排序 | 根据不同类目偏重打分 | 调整展示选项 |
| 模型生成 | 以Prompt生成自然语言对比 | 输出统一格式 |
| 输出呈现 | 简短总结+表格对比+商品卡片 | 前端渲染 |

离线获取类目&类目属性：dwd\_lzd\_pdp\_ai\_specification\_cate

attribute类型参考：[https://university.lazada.sg/course/learn?spm=a1zawg.20777315.sale-prop-complete-dialog.1.3fcf1e13A8ahFm&id=42602](https://university.lazada.sg/course/learn?spm=a1zawg.20777315.sale-prop-complete-dialog.1.3fcf1e13A8ahFm&id=42602)

```plaintext
SELECT DISTINCT 
    cp.category_id,
    ct.local_cate_id,
    cp.property_id,
    cp.property_name,
    v.value_id,
    COALESCE(vs.value_data, 'N/A') AS value_data  -- 处理空值
FROM (
    SELECT DISTINCT 
        category_id,
        property_id,
        property_name
    FROM glb_ods_sg.inherit_category_property_lazada
    WHERE category_id = '398483'
        AND dt = MAX_PT('glb_ods_sg.inherit_category_property_lazada')
        AND cp_status = 0
        AND c_status = 0
        AND (biz_type = 2 OR biz_type = 1)
) cp
-- 取local categoryID
LEFT JOIN lazada_cdm.dim_lzd_prd_cate_property_value ct
    ON cp.category_id = ct.union_cate_id
    AND ct.ds = MAX_PT('lazada_cdm.dim_lzd_prd_cate_property_value')
-- 属性值基础表
LEFT JOIN glb_ods_sg.inherit_category_property_value_lazada v 
    ON v.category_id = cp.category_id 
    AND v.property_id = cp.property_id
    AND v.dt = MAX_PT('glb_ods_sg.inherit_category_property_value_lazada')
-- 属性值映射表
LEFT JOIN lazada_ods.s_union_std_values vs 
    ON vs.value_id = v.value_id 
    AND vs.ds = MAX_PT('lazada_ods.s_union_std_values')
-- 国家映射验证（确保属性在6国有效）
WHERE EXISTS (
    SELECT 1
    FROM lazada_ods.s_union_std_categories_mapping c_mapping
    JOIN lazada_ods.s_union_std_properties_mapping p_mapping 
        ON c_mapping.market_id = p_mapping.market_id
    JOIN lazada_ods.s_std_category_properties local_cp 
        ON local_cp.category_id = c_mapping.local_category_id
        AND local_cp.property_id = p_mapping.local_property_id
        AND local_cp.nation = (
            CASE c_mapping.market_id
                WHEN 360 THEN 'ID'
                WHEN 458 THEN 'MY'
                WHEN 608 THEN 'PH'
                WHEN 702 THEN 'SG'
                WHEN 704 THEN 'VN'
                WHEN 764 THEN 'TH'
            END
        )
    WHERE c_mapping.ds = MAX_PT('lazada_ods.s_union_std_categories_mapping')
        AND p_mapping.ds = MAX_PT('lazada_ods.s_union_std_properties_mapping')
        AND local_cp.ds = MAX_PT('lazada_ods.s_std_category_properties')
        AND c_mapping.channel_id = 6
        AND p_mapping.channel_id = 6
        AND local_cp.status = 0
        AND c_mapping.category_id = cp.category_id
        AND p_mapping.property_id = cp.property_id
        AND (local_cp.features NOT LIKE '%biz_only:redmart%' OR local_cp.features IS NULL)
)

```

#### 1.2.1 一跳触达

| **触点类型** | **设计示意** | **触点说明** |
| --- | --- | --- |
| **一跳tooltips** | ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/94a35ff1-5f59-4632-b735-ca1fcbd8657d.png)<br>![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/3f3db64b-177b-4f14-b195-047cabd83022.png)<br>![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/05a18303-5fba-40d3-b300-699128ca477d.png) | **用户**<br>**1.透出逻辑**<br>*   范围：<br>    <br>    *   用户被动触达：PDP tooltips<br>        <br>    *   用户主动点击：PDP 参数模块<br>        <br>*   疲劳度：单自然日1次<br>    <br>*   对比数量：上限2个（包含主品）<br>    <br>**3.展示逻辑**<br>*   推转问标题：必现，高度动态适应，最多两行，限制字符数，超出省略号截断<br>    <br>    *   直接调用美杜莎文案模版<br>        <br>*   白盒化理由：必现，最多两行，不足两行居底，限制字符数，超出省略号截断<br>    <br>    *   直接调用美杜莎文案模版<br>        <br>*   商品：必现，出现1个，依据具体分支诉求，其余文字宽度自适应<br>    <br>*   行动点：Help me compare 高亮<br>    <br>**3.交互逻辑**<br>*   点击整个区域热区，均跳转至lazzie二级承接页，直出 ~~**默认开启DT模式（react），具备推理过程**~~<br>    <br>*   内容所见即所得<br>    <br>    *   推转问标题作为用户prompt文本发送<br>        <br>    *   商品需在承接页里所见即所得<br>        <br>**策略**<br>**1.意图时机**<br>*   关键特征采集：包含用户历史近期点击、收藏、加购商品记录<br>    <br>*   **意图预测**：判定用户有商品对比意图的参考规则如下<br>    <br>    *   浏览序列中，连续浏览同类商品 ≥2 个，且当前主品的商品停留时间＞3s<br>        <br>    *   收藏、加购了多个同类商品<br>        <br>*   **同类目品判定**：直接按照商品是否处于同一个叶子类目选择判定（后续可以优化）<br>    <br>*   **商品类目范围**：一期限制部分高决策成本类目，只在这部分类目的主品中触发<br>    <br>    *   具体类目范围id待定<br>        <br>*   **公共限制条件**：同类目行为商品需要库存>1，如果没有符合要求的同类目行为商品，则不触发本分支<br>    <br>    *   _后续可能放宽此条件_<br>        <br>    <br>    **2.候选商品选择**<br>    <br>*   候选商品池：++**本期**++++只从用户历史浏览/加购/收藏的与当前主品同类目的商品中选择推荐品++，与用户当前浏览的主品进行对比<br>    <br>    *   _后续方向上，可能会放宽连续浏览同类商品数的限制，调用推荐接口为用户推荐候选品，将原来线上的单品推荐分支与本场景结合_<br>        <br>*   候选品筛选规则：根据最近有行为的候选商品范围进行CTR预估，选择所有候选品中预估CTR分最高的品 |

**​**

#### 1.2.2 二跳承接

| **​** | **样式** | **需求说明** |
| --- | --- | --- |
| 二跳对比承接 | ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/eac7b780-cf72-47ef-b218-2b551b01a2cd.png) | **1.数据逻辑**<br>*   仍根据bizfrom参数进行隔离与保存<br>    <br>    *   bizfrom：PDP-国家-itemid<br>        <br>    *   历史记录：与pdp其他入口共享<br>        <br>**2.展示逻辑：**<br>*   用户发送文本：推转问所见即所得<br>    <br>*   lazzie回答：~~推理模式，展示推理过程，采用react模式~~ **直出**<br>    <br>    *   ~~推理过程：可以保留节点一或节点一+节点三，去除搜索商品部分~~<br>        <br>    <br>    *   正文结果<br>        <br>        *   结论部分：聚合对比的结论，简述推荐用户什么的提交下选择哪款更优的原因<br>            <br>        *   表格<br>            <br>            *   markdown生成表格格式或自行开发，包含：<br>                <br>                *   对比特征<br>                    <br>                *   商品1与商品2的特征精简描述<br>                    <br>        *   商品卡<br>            <br>            *   顺序：赢的商品在第一个，复用一排一样式<br>                <br>            *   标签：赢的商品打上winner<br>                <br>            *   推荐理由：赢的商品补充推荐理由（匹配条件/review高评分/券后减少的价格）<br>                <br>    *   追问<br>        <br>        *   增加询问用户是否对比其他商品的意图<br>            <br>**3.交互逻辑**<br>*   表格：支持特征列冻结，后续列左右滑动<br>    <br>*   商品：点击商品跳转PDP，加购唤起sku面板<br>    <br>*   追问：点击对比其他商品，唤起商品选择器<br>    <br>**4.策略逻辑**<br>*   **承接页大模型生成内容**：承接页内，需要结合用户历史行为进行商品对比结果的输出，需要输出的内容包括：推理过程、对比结果、胜出商品理由。[《商品对比-推理过程及文案示例及QC标准》](https://alidocs.dingtalk.com/i/nodes/93NwLYZXWyxXroNzCOOR6yAA8kyEqBQm)<br>    <br>    *   模型输入信息：需要包含用户近期行为，历史用户画像标签、商品详细参数、商品评价等信息<br>        <br>    *   推理过程文案：调用推理模型，直接使用推理模型的原生推理文案<br>        <br>    *   推理结果：需要包含（1）比较结论，纯文本结构（2）参数对比表格-参数参考RAG方式提供的类目-决策因子映射关系给出<br>        <br>    *   胜出商品理由，直接给出商品，生成一句短文本推荐理由<br>        <br>*   **性能要求**：首token返回rt<1.5s，完整rt<5s |

比较维度示例

| **维度** | **示例字段** | **说明** |
| --- | --- | --- |
| 基本信息 | 品牌、价格、销量 | 用户决策基线 |
| 设计与外观 | 尺寸、重量、颜色 | 便于感性对比 |
| 核心卖点 | 主打功能、技术亮点 | 模型需重点描述差异 |
| 适用人群 | 性别、肤质、场景 | 结合用户画像输出 |
| 性价比 | 价格÷核心能力 | 由AI计算后描述，部分类目 |
| 物流 | 运送时效、包装 |  |
| 评价 | 用户好评率、关键差评 |  |

#### 1.2.3 二跳延伸

| **​** | **样式** | **需求说明** |
| --- | --- | --- |
| 选择其他商品对比 | ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/24efa4f6-bdd4-4b06-a4ca-5814013e8c6d.png) | **选择商品入口卡**<br>**1.透出逻辑**<br>*   新增选择商品意图<br>    <br>*   用户点击追问问题，发送选择其他商品query上墙<br>    <br>**2.展示逻辑**<br>*   主标题：Select comparison products<br>    <br>*   副标题：From history, added to cart, wish list<br>    <br>*   行动点：Select<br>    <br>**3.交互逻辑**<br>*   回复卡片的同时唤起选择商品蒙层<br>    <br>**商品选择弹层**<br>**1.透出逻辑**<br>*   弹起来源：<br>    <br>    *   选择商品意图，自动唤起<br>        <br>    *   选择商品卡片，点击select入口<br>        <br>    *   PDP模版点击PK按钮<br>        <br>*   商品来源：<br>    <br>    *   Add to Cart：过滤当前购物车已加购的同叶子类目商品<br>        <br>    *   Recent Viewed：过滤当前最近浏览下（最近14天）的同叶子类目商品<br>        <br>    *   Wishlist：过滤当前已收藏的同叶子类目商品<br>        <br>**2.展示逻辑**<br>*   主标题：Please Select comparison products<br>    <br>*   主tab：Add to Cart/Recent Viewed/Wishlist<br>    <br>*   行动点：选择按钮<br>    <br>**3.交互逻辑**<br>*   tab切换：可以点击切换tab，展示对应tab下的商品列表，选中高亮态<br>    <br>*   商品列表：展示符合条件下的商品（每20个分页），包含商品主图，商品标题、商品券后价、商品折扣、USP，可以上下滑动<br>    <br>    *   点击单个商品，整条区域都为选中热区，单击选中，再次单击取消选择，选中态高亮，若选中2个后，则其他商品点击选中，toast提示“Compare up to two products at a time~”<br>        <br>    *   若单个商品失效，则灰置整条商品区域<br>        <br>    *   若单个tab商品全部为空，则展示兜底<br>        <br>*   行动条<br>    <br>    *   选中的商品按顺序依次进入候选框内，选满后，发送按钮高亮<br>        <br>        *   若已有传带主品，则默认选择主品已选中，且已放入候选框<br>            <br>    *   点击发送按钮，直接发送query上墙，进行新一轮的对比回复 |

**3.3**

## 1.3.AB实验

|  | **base桶** | **test桶-A** |
| --- | --- | --- |
| **tooltips实验** | **不出商品对比分支** | **增加商品对比分支** |

​

## 1.4.埋点补充

| **PDP点位来源** | **需要代入的spm点位到Lazzie** | **备注** |
| --- | --- | --- |
| 顶部bar点位 | a211g0.pdp.lazzie.topbar | PDP研发已加 |
| review点位 | a211g0.pdp.lazzie.reviews | Lazzie tpp研发传参（已加） |
| 详描点位 | a211g0.pdp.lazzie.description | Lazzie tpp研发传参（已加） |
| tooltips点位 | a211g0.pdp.lazzie.tooltips | Lazzie tpp研发传参（已加） |
| **PK点位** | **a211g0.pdp.lazzie.pk** | **本次新增** |

| **埋点描述** | **事件类型** | **arg1** | **args** | **demo** |
| --- | --- | --- | --- | --- |
| **​**<br>**tooltips** | **2001页面事件** | / | **埋入spm，含c、d位**<br>**C位：**lazzie<br>D位：tooltips | **​** |
|  | **2201曝光事件**（exposure）<br>**2101点击事件**<br>（click） | lazzie\_chat\_card\_tooltips\_exp<br>lazzie\_chat\_card\_tooltips\_clk | **新增参数**<br>1.trackinfo<br>2.clickinfo<br>3.bizfrom<br>4.页面来源id/pagename<br>5.活动id<br>6.tooltipstype=推转问/超级推荐<br>*   askrec<br>    <br>*   superrec<br>    <br>7.分支类型=单品替代/凑单/...<br>*   **新增商品对比** | ![image](https://intranetproxy.alipay.com/skylark/lark/0/2025/png/206750/1744802513210-becf0da1-b552-4d39-8bef-d545dc481fce.png) |
| **泛推荐卡片** | **2201曝光事件**（exposure）<br>**2101点击事件**<br>（click） | lazzie\_chat\_card\_rec\_exp<br>lazzie\_chat\_card\_rec\_clk | **新增intent类型**<br>*   **新增商品对比** |  |
| **商品选择入口** | **2201曝光事件**（exposure） | 商品选择器商品曝光：lazzie\_chat\_itemselect\_entry\_exp | **新增参数**<br>1.bizfrom | ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/27649145-0da0-4d51-a041-e1f1f7852a52.png) |
|  | **2101点击事件**<br>（click） | 商品选择器商品曝光：lazzie\_chat\_itemselect\_entry\_clk |
| **商品选择弹层** | **2201曝光事件**（exposure） | 商品选择器商品曝光：lazzie\_chat\_itemselect\_exp | **新增参数**<br>1.bizfrom<br>2.当前tab：Add to Cart/Recent Viewed/Wishlist<br>3.itemid | ![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/J9LnW6jkdEDgAlvD/img/2b8b1678-aef5-4144-8e88-45db3e2cca32.png) |
|  | **2101点击事件**<br>（click） | 商品选择器tab点击：<br>lazzie\_chat\_tabselect\_clk<br>商品选择器商品点击：lazzie\_chat\_itemselect\_clk<br>商品选择器发送点击：<br>lazzie\_chat\_itemselect\_send\_clk | **新增参数**<br>1.bizfrom<br>2.当前tab：Add to Cart/Recent Viewed/Wishlist<br>3.itemid |
