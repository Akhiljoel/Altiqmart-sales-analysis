# Altiqmart-sales-analysis
BUSINESS QUESTIONS BY STAKEHOLDERS THAT NEED REPORT GENERATION IN SQL

1. list of products with base price>500 and featured in bogof

    select  distinct d.product_name  
    from fact_events f  
    join dim_products d  
    on f.product_code=d.product_code  
    where f.base_price>500 and f.promo_type='BOGOF';  

2. report based on stores in each city in desc order 

    select city,count(store_id) as store_count  
    from dim_stores  
    group by city  
    order by store_count desc;  

3.campaign name revenue before promo vs revenue after promo 

    first change column names as they are conflicting with inbuilt datatypes and function names  
    alter table fact_events    
    change column `quantity_sold(before_promo)` quantity_sold_beforePromo int;  
    alter table fact_events  
    change column `quantity_sold(after_promo)` quantity_sold_afterPromo int;  
    
    select c.campaign_name as campaign_name,concat(round(sum(f.base_price*f.quantity_sold_beforePromo)/1000000,3),' M') as revenue_beforePromo_in_million,  
    concat(round(sum(case when promo_type='50% OFF' then f.base_price*0.5 * f.quantity_sold_afterPromo  
    	when promo_type='25% OFF' then f.base_price*0.75 * f.quantity_sold_afterPromo  
        when promo_type='33% OFF' then f.base_price*0.667 * f.quantity_sold_afterPromo  
        when promo_type='BOGOF' then f.base_price*0.5 *f. quantity_sold_afterPromo  
        when promo_type='500 Cashback' then (f.base_price-500) * f.quantity_sold_afterPromo   
        else f.base_price * f.quantity_sold_afterPromo end)/1000000,2),' M') as revenue_afterPromo_in_million  
    from fact_events f  
    join dim_campaigns c  
    on f.campaign_id=c.campaign_id  
    group by campaign_name;  

4.report on incremental quantity change percentage

    select avg(quantity_sold_beforePromo),avg(quantity_sold_afterPromo)  
    from fact_events  
    where promo_type='BOGOF';  -- since avg quantity is triple we can safely assume that extra inventory left  
    
    
    with quantitycount as (  
    select d.category,sum(f.quantity_sold_beforePromo)as beforequantity,sum(f.quantity_sold_afterPromo)as afterquantity  
    from fact_events f  
    join dim_products d  
    on f.product_code=d.product_code  
    where f.campaign_id LIKE'%DIW%'  
    group by d.category  
    )  
    
    select category,(afterquantity-beforequantity)*100/beforequantity as ICR_Percent,  
    dense_rank()over(order by (afterquantity-beforequantity)*100/beforequantity desc) as rank_of_ICR  
    from quantitycount  
    order by ICR_Percent desc;  

5. Report featuring top5 categories ranked by incremental revenue percentage across all campaigns

    SELECT 
        p.category,  
        p.product_name,  
        ROUND(  
            (  
                SUM(  
                    CASE   
                        WHEN promo_type = '50% OFF' THEN f.base_price * 0.5 * f.quantity_sold_afterPromo  
                        WHEN promo_type = '25% OFF' THEN f.base_price * 0.75 * f.quantity_sold_afterPromo  
                        WHEN promo_type = '33% OFF' THEN f.base_price * 0.667 * f.quantity_sold_afterPromo  
                        WHEN promo_type = 'BOGOF' THEN f.base_price * 0.5 * f.quantity_sold_afterPromo  
                        WHEN promo_type = '500 Cashback' THEN (f.base_price - 500) * f.quantity_sold_afterPromo  
                        ELSE f.base_price * f.quantity_sold_afterPromo  
                    END  
                )  
                - SUM(f.base_price * f.quantity_sold_beforePromo)  
            )   
            / NULLIF(SUM(f.base_price * f.quantity_sold_beforePromo), 0) * 100  
        , 2) AS revenue_change_percent  
    FROM fact_events f  
    JOIN dim_products p  
        ON f.product_code = p.product_code  
    JOIN dim_campaigns c  
        ON f.campaign_id = c.campaign_id  
    GROUP BY p.category, p.product_name  
    order by revenue_change_percent desc  
    limit 5;  
