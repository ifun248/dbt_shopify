models目录结构：
-- intermediate
    -- intermediate.yml
    -- shopify__customers__order_aggregates.sql
    -- shopify__orders__order_line_aggregates.sql
    -- shopify__orders__order_refunds.sql
-- utils
    -- shopify__calendar.sql
-- shopify.yml
-- shopify__customer_cohorts.sql
-- shopify__customers.sql
-- shopify__order_lines.sql
-- shopify__orders.sql
-- shopify__products.sql
-- shopify__transactions.sql

一 目录intermediate
文件intermediate/shopify__customers__order_aggregates.sql的作用：
1 将原始数据表shopify_order与预处理表shopify__transactions进行关联，关联条件（order_id, source_relation），
2 根据customer_id和source_relation(店铺ID)进行分组统计出：
  min(orders.created_timestamp) as first_order_timestamp,（客户第一次订单时间）
  max(orders.created_timestamp) as most_recent_order_timestamp,（客户最后一次订单时间）
  avg(case when transactions.kind in ('sale','capture') then transactions.currency_exchange_calculated_amount) as average_order_value（客户在店铺的每次平均订单金额）,
  sum(case when transactions.kind in ('sale','capture') then transactions.currency_exchange_calculated_amount) as lifetime_total_spent（客户在店铺的总订单金额）,
  sum(case when transactions.kind in ('refund') then transactions.currency_exchange_calculated_amount) as lifetime_total_refunded（客户在店铺的总退款金额）,
  count(distinct orders.order_id) as lifetime_count_orders（客户在店铺的总订单次数）
  
文件intermediate/shopify__orders__order_line_aggregates.sql的作用：
1 对原始数据表shopify_order_line进行order_id和source_relation(店铺ID)分组统计：
  count(*) as line_item_count（count of a line item of an order）
  
文件intermediate/shopify__orders__order_refunds.sql的作用：
1 将原始shopify_refund（退款表）和原始shopify_order_line_refund（退款订单项表）进行关联，关联条件为（refund_id, source_relation）
2 选择需要的信息：
  refunds.refund_id（退款ID）,
  refunds.created_at（退款的创建时间）,
  refunds.order_id（退款的订单ID）,
  refunds.user_id（退款用户ID）,
  refunds.source_relation（退款的店铺ID）,
  order_line_refunds.order_line_id,（退款的订单项id）
  order_line_refunds.restock_type,（退款的进货类型）
  order_line_refunds.quantity,（退款的数量）
  order_line_refunds.subtotal,（退款的金额）
  order_line_refunds.total_tax（退款总税额）
  
二 目录utils
文件utils/shopify__calendar.sql的作用：
1 生成日期

三 目录models
文件shopify__customer_cohorts.sql的作用：
  1 将预处理表shopify__calendar（每月首日表）和shopify__customers预处理表（）进行关联，关联条件（shopify__customers.first_order_timestamp <= shopify__calendar.date_month）
  2 选择信息：
    calendar.date_day as date_month,（每月第一天）
    customers.customer_id,（客户ID）
    customers.first_order_timestamp,（客户第一次下单时间戳）
    customers.source_relation,（客户所在店铺ID）
    {{ dbt_utils.date_trunc('month', 'shopify__customers.first_order_timestamp') }} as cohort_month（客户第一次下单时间所处的每月第一天）
  3 将上一步结构表与预处理表shopify__orders进行关联，关联条件（customer_id, source_relation, date_month）
  4 根据customer_calendar.date_month, customer_calendar.customer_id, customer_calendar.first_order_timestamp, customer_calendar.cohort_month和customer_calendar.source_relation进行分组统计：
    count(distinct orders.order_id) as order_count_in_month,（每个月的订单数）
    sum(orders.order_adjusted_total) as total_price_in_month,（每个月的订单调整总数）
    sum(orders.line_item_count) as line_item_count_in_month（每个月的订单具体项总数）
  5 通过窗口函数（partition by customer_id, source_relation order by date_month rows between unbounded preceding and current row），进一步统计出：
    sum(order_count_in_month),（一个客户按月累积的订单数）
    sum(total_price_in_month),（一个客户按月累积的订单调整数）
    sum(line_item_count_in_month)（一个客户按月累计的订单具体项总数）
总结：对客户，以月为单位，统计其订单总数、订单调整总数和订单具体项总数。（可以用BI工具进行展示说明）

shopify__customers.sql的作用：
  1 将原始表shopify_customer和预处理表shopify__customers__order_aggregates进行关联，关联条件（customer_id, source_relation）
  2 选择需要的信息：
    customers.*,
    shopify__customers__order_aggregates.first_order_timestamp,（客户第一次订单时间）
    shopify__customers__order_aggregates.most_recent_order_timestamp,（客户最后一次订单时间）
    shopify__customers__order_aggregates.average_order_value as average_order_value,（客户在店铺的每次平均订单金额）
    shopify__customers__order_aggregates.lifetime_total_spent as lifetime_total_spent,（客户在店铺的总订单金额）
    shopify__customers__order_aggregates.lifetime_total_refunded as lifetime_total_refunded,（客户在店铺的总退款金额）
    shopify__customers__order_aggregates.lifetime_total_spent - orders.lifetime_total_refunded as lifetime_total_amount,（客户在店铺的总订单金额 - 总退款金额）
    shopify__customers__order_aggregates.lifetime_count_orders as lifetime_count_orders（客户在店铺的总订单次数）

shopify__order_lines.sql的作用：
  

shopify__orders.sql的作用：
  1 基于预处理表shopify__orders__order_refunds进行关联，根据（order_id, source_relation）统计出:
    sum(subtotal) as refund_subtotal,
    sum(total_tax) as refund_total_tax
  2 基于原始数据表shopify_order_adjustment进行关联，根据（order_id, source_relation）统计出:
    sum(amount) as order_adjustment_amount,
    sum(tax_amount) as order_adjustment_tax_amount
  3 基于原始数据表shopify_order、预处理表shopify__orders__order_line_aggregates，以及上面两个聚合表进行关联，根据id和source_relation，选择需要的信息
    json_parse("total_shipping_price_set",["shop_money","amount"]),
    order_adjustment_amount,
    order_adjustment_tax_amount,
    refund_subtotal,
    refund_total_tax,
    (shopify_order.total_price + order_adjustment_amount + order_adjustment_tax_amount - refund_subtotal - refund_total_tax) as order_adjusted_total,（总的调整金额）
    order_lines.line_item_count（订单项的总数）
  4 根据partition by customer_id, source_relation order by created_timestamp，针对用户在店铺中的记录，给出购买记录是第一次new，还是不是第一次repeat
    

shopify__products.sql的作用：
  1 将预处理表shopify__order_lines和预处理表shopify__orders进行关联，关联条件（order_id, source_relation），
  2 选择需要的信息：
    shopify__order_lines.product_id,
    shopify__order_lines.source_relation,
    sum(shopify__order_lines.quantity) as quantity_sold,（订单项的数量）
    sum(shopify__order_lines.pre_tax_price) as subtotal_sold,（订单项的）
    sum(shopify__order_lines.quantity_net_refunds) as quantity_sold_net_refunds,（）
    sum(shopify__order_lines.subtotal_net_refunds) as subtotal_sold_net_refunds,（）
    min(shopify__orders.created_timestamp) as first_order_timestamp,（最早订单时间）
    max(shopify__orders.created_timestamp) as most_recent_order_timestamp（最晚订单时间）
  3 将上面结果与原始数据表shopify_product进行关联，关联条件为（product_id, source_relation），
  4 选择需要的信息：
    products.*,
    quantity_sold,
    subtotal_sold,
    quantity_sold_net_refunds,
    subtotal_sold_net_refunds,
    first_order_timestamp,
    most_recent_order_timestamp

shopify__transactions.sql的作用：
  1 将原表shopify_transaction中的字段receipt进行拆解，统计：
    统计receipt中字段的exchange_rate,
    统计receipt中字段的exchange_rate * amount,

  
