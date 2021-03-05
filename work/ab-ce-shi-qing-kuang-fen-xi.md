---
description: AB测试
---

# AB测试情况分析

## 一、提取建立基本表

### 用户分层部分

```sql
---新访客/老访客/已下单用户
DROP TABLE IF EXISTS develop.wsj_personal_label;
CREATE TABLE develop.wsj_personal_label  AS (
    SELECT
        DATE ( dt ) AS dt,
        patpat_id,
        user_label
    FROM
        dwd.patpat_label_day
    WHERE
        --实验范围
        dt BETWEEN '20210109' AND '20210115'    
);
```

### 流量表部分

```sql
--流量表数据
DROP TABLE IF EXISTS develop.wsj_test_name_traffic;
CREATE TABLE develop.wsj_test_name_traffic AS (
    SELECT 
        DATE (source_created_at - INTERVAL '8 HOUR' ) AS dt,
        split_part(split_part(url,'test_name=',2),'&',1)  AS test_label,
        patpat_id,
        customer_id,
        page_url,
        page_type,
        unique_session_id
    FROM
        dwd.dwd_traffic_data 
    WHERE
        --实验范围
        (DATE (source_created_at - INTERVAL '8 HOUR') BETWEEN 'time1' AND 'time2')
        AND (lower(url) LIKE '%test_name=%')
        AND sales_platform_id IN ('ios','android','web','wap')
    ORDER BY 1,2,3
);
```

### 点击表部分

```sql
--点击表数据
DROP TABLE IF EXISTS develop.wsj_test_name_click;
CREATE TABLE develop.wsj_test_name_click AS (
    SELECT distinct 
        DATE (source_created_at - INTERVAL '8 HOUR' ) AS dt,
        split_part(split_part(url,'test_name=',2),'&',1)  AS test_label,
    --     CASE 
    --             WHEN url LIKE '%test_name=a%' THEN 'a'
    --             WHEN url LIKE '%test_name=b%' THEN 'b'
    --     END AS test_label,
        unique_session_id,
        customer_id,
        patpat_id,
        page_url,
        click_event_type
    FROM
        dwd.dwd_click_data 
    WHERE
        --实验范围
        (DATE (source_created_at - INTERVAL '8 HOUR') BETWEEN 'time1' AND 'time2')
        AND (lower(url) LIKE '%test_name=%')
        AND sales_platform_id IN ('ios','android','web','wap')
    ORDER BY 1,2
);
```

### 订单表部分

```sql
--时间范围内全量订单数据
DROP TABLE IF EXISTS develop.wsjtest_nameorder;
CREATE TABLE develop.wsjtest_nameorder AS (
    SELECT DISTINCT 
          dt,
          patpat_id,
          customer_id,
          unique_session_id,
          q1.order_id,
          all_discount_cost,
          purchase_cost,
          voucher_cost,
          coupon_cost,
          wallet_cost,
          discount_cost,
          gmv
    FROM (
          SELECT 
              DATE(source_created_at - INTERVAL '8 HOUR' ) AS dt,
              order_id,
              cart_id,
              customer_id,
              patpat_id
          FROM
              dwd.f_order 
          WHERE
              DATE (source_created_at - INTERVAL '8 HOUR') BETWEEN 'time1' AND 'time2' 
              AND platform IN ('ios','android')
              AND order_type IN ( 0, 4 )
              AND sale_type NOT IN ( 'test_sale', 'test sale' )
              AND current_status NOT in ('notpay')
              And is_payment=1
          ORDER BY 1,2,3,4
        ) q1
        LEFT JOIN (
            SELECT
                order_id,
                SUM ( nvl ( purchase_cost,0)) AS purchase_cost,
                SUM ( wallet ) AS wallet_cost,
                SUM ( nvl ( sku_input, 0 ) + nvl ( ship_input, 0 ) + nvl ( tax_input, 0 ) ) AS gmv 
            FROM
                "public".roi_result_d_new 
            WHERE
                tid BETWEEN 'time1' AND 'time2' 
            GROUP BY 1 
            ) q2 ON q2.order_id = q1.order_id
        LEFT JOIN (
            SELECT 
                order_id,
                SUM ( nvl (order_amt,0)) AS all_discount_cost,
                SUM (CASE WHEN flag_value='2' THEN nvl(order_amt,0) END) AS coupon_cost,
                SUM (CASE WHEN flag_value='3' THEN nvl(order_amt,0) END) AS voucher_cost,
                SUM (CASE WHEN flag_value IN ('5','6','7','8') THEN nvl(order_amt,0) END) AS discount_cost
            FROM
                "public".roi_discount_detail_new
            WHERE tid BETWEEN 'time1' AND 'time2'
            GROUP BY 1
        ) q3 ON q3.order_id=q1.order_id 
        LEFT JOIN (
            SELECT 
                cart_id,unique_session_id
            FROM
            "public".track_add_to_cart ) q4 ON q4.cart_id = q1.cart_id
    );
```

## 二、确定实验组用户划分依据：

### 设备or用户

```sql
--观察用户或者设备在多个用户组的占比，确定是否排除
SELECT
    cnt,
    count(1)
FROM ( 
    SELECT 
        dt,
        unique_session_id, 
        COUNT ( DISTINCT test_label ) AS cnt 
    FROM 
        develop.wsj_personal_traffic
    GROUP BY 1,2 
    ORDER BY 1 desc
) T 
GROUP BY 1
ORDER BY 1
```

### 验证分析

```sql
--反例名单
SELECT
    dt,
    customer_id
FROM ( 
    SELECT 
        dt,
        customer_id, 
        COUNT ( DISTINCT lable ) AS cnt 
    FROM 
        develop.gzw_newcomer_experient_v3_data_old   
    --WHERE sales_platform_id='ios'
    GROUP BY 1,2
    ORDER BY 1 desc
) T 
WHERE cnt>1
ORDER BY 1 DESC
LIMIT 10;

--反例详细信息
SELECT
    * 
FROM
    dwd.dwd_traffic_data 
WHERE
    unique_session_id = '2d4cc076-75c4-3441-bc89-79df31ff649a' 
    AND DATE ( source_created_at - INTERVAL '8 hour' ) = 'time';
```

## 三、取数逻辑

### 流量表取数

```sql
--流量表相关事件查询
SELECT 
    t1.dt,
    t0.test_label,
    count ( DISTINCT t1.customer_id) AS UV,
    COUNT ( DISTINCT CASE WHEN page_type = 'product_detail' THEN t1.customer_id ELSE NULL END ) AS 商品详情uv,
    COUNT ( DISTINCT  t2.customer_id ) as 加购_uv,
    COUNT ( DISTINCT CASE WHEN page_type = 'cart' THEN t1.customer_id ELSE NULL END ) AS cart页面uv,
    COUNT ( DISTINCT CASE WHEN page_type = 'checkout' THEN t1.customer_id ELSE NULL END ) AS checkout_out页面uv,
    COUNT ( DISTINCT CASE WHEN page_type = 'confirmation' THEN t1.customer_id ELSE NULL END ) AS confirmation页面uv
FROM (
    --只取只有一个标签的用户
    SELECT DISTINCT
        q2.dt,
        q1.customer_id,
        test_label
    FROM (
        (SELECT
            customer_id,
            count(distinct test_label) as cnt 
        FROM 
            develop.wsj_test_name_traffic
        GROUP BY 1 HAVING( COUNT( DISTINCT test_label )=1)) q1
        INNER JOIN
        develop.wsj_test_name_traffic q2 ON q1.customer_id=q2.customer_id
    ) 
    )t0
    LEFT JOIN
    develop.wsj_test_name_traffic t1 ON t0.customer_id=t1.customer_id AND t0.dt=t1.dt
    LEFT JOIN 
    dwd.dwd_add_to_cart t2 ON t1.dt = DATE ( t2.source_created_at - INTERVAL '8 hours' ) AND t1.customer_id = t2.customer_id 
GROUP BY 1,2
ORDER BY 1,2;
```

### 点击表取数

```sql
--点击事件查询
SELECT 
    t1.dt,
    t0.test_label,
    COUNT ( DISTINCT CASE WHEN click_event_type = 'add_to_cart' THEN t1.customer_id ELSE NULL END ) AS 加购点击uv,
    COUNT ( DISTINCT CASE WHEN click_event_type = 'cart-checkoutbtn' THEN t1.customer_id ELSE NULL END ) AS check_out点击uv,
    COUNT ( DISTINCT CASE WHEN click_event_type = 'submit_order' THEN t1.customer_id ELSE NULL END ) AS submit_order点击uv
FROM (
    --只取只有一个标签的用户
    SELECT DISTINCT
        q2.dt,
        q1.customer_id,
        test_label
    FROM (
        (SELECT
            customer_id,
            count(distinct test_label) as cnt 
        FROM 
            develop.wsj_test_name_traffic
        GROUP BY 1 HAVING( COUNT( DISTINCT test_label )=1)) q1
        INNER JOIN
        develop.wsj_test_name_traffic q2 ON q1.customer_id=q2.customer_id 
    ) 
    ) t0
    LEFT JOIN
    develop.wsj_test_name_click t1 ON t0.customer_id=t1.customer_id AND t0.dt=t1.dt
GROUP BY 1,2
ORDER BY 1,2;
```

### 订单表取数

```sql
--订单相关查询
SELECT 
    t1.dt,
    test_label,
    COUNT ( DISTINCT t1.customer_id ) AS 购买用户数,
    COUNT ( DISTINCT order_id ) AS 订单数,
    SUM ( DISTINCT nvl(gmv,0) ) AS GMV,
    SUM ( DISTINCT CAST(nvl(coupon_cost,0) as decimal(38, 2)) ) AS coupon折扣,
    SUM ( DISTINCT CAST(nvl(wallet_cost,0) as decimal(38, 2)) ) AS wallet折扣,
    SUM ( DISTINCT CAST(nvl(voucher_cost,0) as decimal(38, 2)) ) AS voucher折扣,
    SUM ( DISTINCT CAST(nvl(purchase_cost,0) as decimal(38, 2)) ) AS 商品成本,
    SUM ( DISTINCT CAST(nvl(discount_cost,0) as decimal(38, 2)) ) AS 商品降价成本
FROM (
    --只取只有一个标签的用户
    SELECT DISTINCT
        q2.dt,
        q1.customer_id,
        test_label
    FROM (
        (SELECT
            customer_id,
            count(distinct test_label) as cnt 
        FROM 
            develop.wsj_test_name_traffic
        GROUP BY 1 HAVING( COUNT( DISTINCT test_label )=1)) q1
        INNER JOIN
        develop.wsj_test_name_traffic q2 ON q1.customer_id=q2.customer_id
    ) 
    ) t0
    LEFT JOIN 
    develop.wsjtest_nameorder t1 ON t0.customer_id=t1.customer_id AND t0.dt=t1.dt
GROUP BY 1,2
ORDER BY 1,2;
```

## 四、取数结果汇总

### 汇总

```sql
WITH w1 AS(
    SELECT 
        t1.dt,
        t0.test_label,
        count(DISTINCT t1.customer_id) AS UV,
        COUNT ( DISTINCT CASE WHEN page_type = 'product_detail' THEN t1.customer_id ELSE NULL END ) AS 商品详情uv,
        COUNT ( DISTINCT  t2.customer_id ) as 加购_uv,
        COUNT ( DISTINCT CASE WHEN page_type = 'cart' THEN t1.customer_id ELSE NULL END ) AS cart页面uv,
        COUNT ( DISTINCT CASE WHEN page_type = 'checkout' THEN t1.customer_id ELSE NULL END ) AS checkout_out页面uv,
        COUNT ( DISTINCT CASE WHEN page_type = 'confirmation' THEN t1.customer_id ELSE NULL END ) AS confirmation页面uv
    FROM (
        --只取只有一个标签的用户
        SELECT DISTINCT
            q2.dt,
            q1.customer_id,
            test_label
        FROM (
            (SELECT
                customer_id,
                count(distinct test_label) as cnt 
            FROM 
                develop.wsj_test_name_traffic
            GROUP BY 1 HAVING( COUNT( DISTINCT test_label )=1)) q1
            INNER JOIN
            develop.wsj_test_name_traffic q2 ON q1.customer_id=q2.customer_id
        ) 
        ) t0
        LEFT JOIN
        develop.wsj_test_name_traffic t1 ON t0.customer_id=t1.customer_id AND t0.dt=t1.dt
        LEFT JOIN 
        dwd.dwd_add_to_cart t2 ON t1.dt = DATE ( t2.source_created_at - INTERVAL '8 hours' ) AND t1.customer_id = t2.customer_id
    GROUP BY 1,2
    ORDER BY 1,2),

w2 AS (
    --点击事件查询
    SELECT 
        t1.dt,
        t0.test_label,
        COUNT ( DISTINCT CASE WHEN click_event_type = 'add_to_cart' THEN t1.customer_id ELSE NULL END ) AS 加购点击uv,
        COUNT ( DISTINCT CASE WHEN click_event_type = 'cart-checkoutbtn' THEN t1.customer_id ELSE NULL END ) AS check_out点击uv,
        COUNT ( DISTINCT CASE WHEN click_event_type = 'submit_order' THEN t1.customer_id ELSE NULL END ) AS submit_order点击uv
    FROM (
        --只取只有一个标签的用户
        SELECT DISTINCT
            q2.dt,
            q1.customer_id,
            test_label
        FROM (
            (SELECT
                customer_id,
                count(distinct test_label) as cnt 
            FROM 
                develop.wsj_test_name_traffic
            GROUP BY 1 HAVING( COUNT( DISTINCT test_label )=1)) q1
            INNER JOIN
            develop.wsj_test_name_traffic q2 ON q1.customer_id=q2.customer_id
        ) 
        ) t0
        LEFT JOIN
        develop.wsj_test_name_click t1 ON t0.customer_id=t1.customer_id AND t0.dt=t1.dt
    GROUP BY 1,2
    ORDER BY 1,2),

w3 AS (
    --订单相关查询
    SELECT 
        t1.dt,
        test_label,
        COUNT ( DISTINCT t1.customer_id ) AS 购买用户数,
        COUNT ( DISTINCT order_id ) AS 订单数,
        SUM ( DISTINCT nvl(gmv,0) ) AS GMV,
        SUM ( DISTINCT CAST(nvl(coupon_cost,0) as decimal(38, 2)) ) AS coupon折扣,
        SUM ( DISTINCT CAST(nvl(wallet_cost,0) as decimal(38, 2)) ) AS wallet折扣,
        SUM ( DISTINCT CAST(nvl(voucher_cost,0) as decimal(38, 2)) ) AS voucher折扣,
        SUM ( DISTINCT CAST(nvl(purchase_cost,0) as decimal(38, 2)) ) AS 商品成本,
        SUM ( DISTINCT CAST(nvl(discount_cost,0) as decimal(38, 2)) ) AS 商品降价成本
    FROM (
        --只取只有一个标签的用户
        SELECT DISTINCT
            q2.dt,
            q1.customer_id,
            test_label
        FROM (
            (SELECT
                customer_id,
                count(distinct test_label) as cnt 
            FROM 
                develop.wsj_test_name_traffic
            GROUP BY 1 HAVING( COUNT( DISTINCT test_label )=1)) q1
            INNER JOIN
            develop.wsj_test_name_traffic q2 ON q1.customer_id=q2.customer_id
        ) 
        ) t0
        LEFT JOIN 
        develop.wsjtest_nameorder t1 ON t0.customer_id=t1.customer_id AND t0.dt=t1.dt
    GROUP BY 1,2
    ORDER BY 1,2)

SELECT 
    w1.dt,
    w1.test_label,
    w1.UV,
    w1.商品详情uv,
    w1.加购_uv,
    w2.加购点击uv,
    w1.cart页面uv,
    w2.check_out点击uv,
    w1.checkout_out页面uv,
    w2.submit_order点击uv,
    w1.confirmation页面uv,
    w3.购买用户数,
    w3.订单数,
    w3.GMV,
    w3.coupon折扣,
    w3.wallet折扣,
    w3.voucher折扣,
    w3.商品成本,
    w3.商品降价成本
FROM 
    w1 LEFT JOIN w2 ON w1.dt=w2.dt AND w1.test_label=w2.test_label
    LEFT JOIN w3 ON w1.dt=w3.dt AND w1.test_label=w3.test_label
WHERE
    --实验分组不等于x
    w1.test_label <> 'x'
ORDER BY 1,2;
```

