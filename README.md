# # Supermarket Sales Analysis SQL Project

- **Level**: Advanced
- **Dataset**: Supermarket_sales
- **Language**: SQL

## Abstract and research questions

This is the second part of the study 'Supermarket Sales Analysis SQL', in which I focused on more advanced techniques to identify the main criticalities in the company's business strategy. First off, I implemented a query to answer the questions: how attraqctive has the company been for new customers trhoughout the years? What is the average time for a new customer to place a second order? What is the percentage of customers who actually place a second order? What was the entity of improvement in the latter metric (if any)? I answered these questions through the Customer Retention Analysis, which highlighted a descreasing tendency for the newly acired customers throughout years, and an increasing trend for the 'Average days to second order' value. This pattern describes probably a descresed quality of advertising campaigns throughout the years, while a constant improvement of product (and/or shipping/customer care) quality. 

### 3. Advanced-level business analysis
In this section, I carried out a more advanced analysis, emploing the following techniques:
- **Customer Retention Analysis**: It's a complex analysis that investigates how much a customer is 'loyal' to the company. It includes the calculation of the following metrics:
  1) 'Total acquired customers by year': it describes how many new customers were acquired for each year. It helps understand if, for example, the current sales volume is due to a good amount of 'loyal' customers, or just to a flow of new customers, maybe acquired by advertising campaigns.
  2) 'Average days to second order': it measures how many days, on average, a customer waits before placing a second order; it describes the risk of customers switching to a competitor and provides insights into the optimal timing for sending promotional emails.
  3) 'Retention rate': it is computed by the formula: $$\text{Retention Rate (\\%)} = \frac{\text{Customers with ≥ 2 orders}}{\text{Total Customers}} \times 100$$
  4) 'Annual improvement': computed by the formula $$\text{Annual Improvement(\\%)}  = \frac{\text{AvgDays (t-1)} - \text{AvgDays (t)}}{\text{AvgDays (t-1)}} \times 100$$ where AvgDays (t-1) represents the average shipping performance of the previous year, and AvgDays(t) represents that of the current year. This metric analyzes the year-over-year relative improvement in shipping performance. It provides useful complementary information that, when paired with customer retention trends, can help infer a causal effect.

```sql
-- Customer Retention analysis
WITH FirstOrders AS (
    SELECT 
        s.customer_id, 
        MIN(o.order_date) AS first_purchase_date,
        YEAR(MIN(o.order_date)) AS cohort_year
    FROM sales s 
    join orders o on o.order_id = s.order_id 
    GROUP BY s.customer_id
),
SecondOrders AS (
    SELECT 
        s.customer_id, 
        MIN(o.order_date) AS second_purchase_date
    FROM sales s
    join orders o on o.order_id = s.order_id
    JOIN FirstOrders f ON s.customer_id = f.customer_id
    WHERE o.order_date > f.first_purchase_date
    GROUP BY s.customer_id
)
SELECT 
    f.cohort_year AS `Acquition year (Cohort)`,
    COUNT(f.customer_id) AS `Total aquired customers`,
    ROUND(AVG(DATEDIFF(s.second_purchase_date, f.first_purchase_date)), 0) AS `Average days to second order`,
    ROUND((COUNT(s.second_purchase_date) / COUNT(f.customer_id)) * 100, 2) AS `Retention Rate (%)`,
	(LAG(ROUND(AVG(DATEDIFF(s.second_purchase_date, f.first_purchase_date)), 0),1) over (order by f.cohort_year) -
	ROUND(AVG(DATEDIFF(s.second_purchase_date, f.first_purchase_date)), 0))/
	LAG(ROUND(AVG(DATEDIFF(s.second_purchase_date, f.first_purchase_date)), 0),1) over (order by f.cohort_year)*100
	'Annual improvement in first/second order timespan (%)'
	FROM FirstOrders f
LEFT JOIN SecondOrders s ON f.customer_id = s.customer_id
GROUP BY 1
order by 1;
```
2014 was the most profitable year in terms of new customer acquisition, with 595 new customers, followed by a descending trend: 2015 (136), 2016 (51), and 2017 (11). An opposite trend is observed in the 'average days to second order' variable, which decreased constantly, from 292 days in 2014 to 105 in 2017 - the lower the values the better, since a low average means that a customer becomes loyal sooner. This measure reflects the true acquisition potential of the company, since it is tightly linked to product quality and customer satisfaction. Retention rates remain more or less constant, except for 2017, where there is a sudden drop. Taken together, these data suggest that the general attractiveness of the company went down throughout the years, attracting fewer and fewer first-time customers, whereas the product quality improved — confirmed also by the constant improvement in 'first/second order timespan'. Effective business advice would be to strengthen advertising campaigns to attract new customers, perhaps in areas where sales volumes are modest.

- **RFM analysis (Recency, Frequency, Monetary)**: here, each customer is analyzed individualy, and the metrics are defined as follows:
  1) Recency value = last order within the dataset - last order of the customer.It measures how much time passed since the last purchase (the lower the better).
  2) Frequency value = total sum of unique orders by customer. It measures the number of orders within a defined period (the higher the better).
  3) Monetary value = Total sum of sales amount by customer. How much income has the single customer
generated (the higher the better).
I completed the RFM with the normalization of each value (to a score between 1 and 5, where numbers near 5 are considered optimal) and their combination, generating a syntetic score for each customer (between 1 and 15, where higher scores indicate a high-quality customer).

```sql
/* RFM analysis (Recency, Frequency, Monetary)
*
* Recency value = last order within the dataset - last order of the customer
* Frequency value = total sum of unique orders by customer
* Monetary value = Total sum of sales amount by customer */
WITH RFM_Base AS (
    SELECT 
        s.customer_id,
        MAX(o.order_date) AS last_order_date,
        SUM(s.sales) AS monetary_value,
        COUNT(DISTINCT s.order_id) AS frequency_value
    FROM sales s
    JOIN orders o ON s.order_id = o.order_id AND s.product_id = o.product_id
    GROUP BY s.customer_id
)
SELECT 
    customer_id,
    DATEDIFF((SELECT MAX(order_date) FROM orders), last_order_date) AS 'Recency value',
    frequency_value 'Frequency value',
    ROUND(monetary_value, 2) AS 'Monetary value'
FROM RFM_Base
ORDER BY 4 DESC;

/* Assigning a score to each raw RFM value 
 * Each score represents a quintile */
WITH RFM_Calculated AS (
    SELECT 
        s.customer_id,
        DATEDIFF((SELECT MAX(order_date) FROM orders), MAX(o.order_date)) AS recency,
        COUNT(DISTINCT s.order_id) AS frequency,
        SUM(s.sales) AS monetary
    FROM sales s
    JOIN orders o ON s.order_id = o.order_id AND s.product_id = o.product_id
    GROUP BY s.customer_id
),
RFM_Scores AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM RFM_Calculated
)
SELECT 
    customer_id,
    recency 'Recency value', frequency 'Frequency value', monetary 'Monetary value',
    r_score 'Recency score', f_score 'Frequency score', m_score 'Monetary score',
    (r_score + f_score + m_score) AS 'RFM total score'
FROM RFM_Scores;
```
The RFM score computation highlights 22 customers with the maximum score of 15. It is interesting to note that, on average, the Monetary value remains in the medium range of the last quantile, suggesting that higher raw values of Recency and Frequency might weigh more on the combined outcome.

- **Pareto Analysis**: It aims to determine wich minority of customers generates the majority of profit. It follows the Pareto 80/20 rule, according to which 80\% of the profit is generated by 20% of customers. This technique is useful to determine who the 'big buyers' are, on whom the company should focus. Indeed, losing these subjects would mean significantly reducing the averall profit of the company. 

```sql
-- Pareto Analysis
WITH Customer_Profit AS (
    SELECT 
        customer_id,
        SUM(profit) AS total_customer_profit
    FROM sales
    GROUP BY customer_id
    HAVING SUM(profit) > 0
),
Cumulative_Analysis AS (
    SELECT 
        customer_id,
        total_customer_profit,
        SUM(total_customer_profit) OVER (ORDER BY total_customer_profit DESC) AS cumulative_profit,
        SUM(total_customer_profit) OVER () AS global_total_profit,
        ROW_NUMBER() OVER (ORDER BY total_customer_profit DESC) AS customer_rank,
        COUNT(*) OVER () AS total_customer_count
    FROM Customer_Profit
)
SELECT 
    customer_id,
    ROUND(total_customer_profit, 2) AS profit,
    ROUND((cumulative_profit / global_total_profit) * 100, 2) AS cumulative_percentage,
    ROUND((customer_rank / total_customer_count) * 100, 2) AS customer_percentage
FROM Cumulative_Analysis
ORDER BY total_customer_profit DESC;
```

The Pareto analysis reveals that a small percentage of big buyers sustain the overall company profit. Indeed, only around 3% of customers (19) generates 20% of the profit, posing the company at huge risk, as losing even a few of them to competitors would lead to a significant loss in total profit. Furthermore, it is observable a long tail of 'micro-buyers' - customers who have placed a few small orders - potentially increasing operating costs (shipping, packaging, customer care).
The company should implement a business strategy in order to maintain the big buyers through VIP programs and special customer care. These customers can be easily identified through the joint RFM score computed earlier.
The company should also focus on improving cost management for the long tail of micro-buyers, for instance by automating the purchase iter, and promoting larger orers through special offers. Last but not least, the IT department should implement more targeted recommendation algorithms to increase the average order value (AOV) of these lower-tier customers.


