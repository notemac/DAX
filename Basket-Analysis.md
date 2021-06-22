# Basket Analysis

## URLs
- [Basket analysis – DAX Patterns](https://www.daxpatterns.com/basket-analysis/)
- [Market Basket Analysis (Power BI DAX Tutorial) - YouTube](https://www.youtube.com/watch?v=3OlZuXH9Y_g)
- [Chris Webb's BI Blog: Simple Basket Analysis in DAX Chris Webb's BI Blog (crossjoin.co.uk)](https://blog.crossjoin.co.uk/2010/10/20/simple-basket-analysis-in-dax/)
- [Basket Analysis Introduction - Best Practice Tips For Power BI (enterprisedna.co)](https://blog.enterprisedna.co/basket-analysis-introduction-best-practice-tips-for-power-bi-using-dax/)
- [Basket Analysis Example - Power BI Advanced Analytics | Enterprise DNA](https://blog.enterprisedna.co/advanced-basket-analysis-example-in-power-bi-cross-selling/)
- [Advance DAX Tutorial: Basket Analysis 2.0 | by Davis Zhang | Towards Data Science](https://towardsdatascience.com/explore-the-potential-of-products-through-customers-purchase-behaviour-in-power-bi-basket-a1f77e8a2bf6)
- [Power BI: Basket Analysis Full Tutorial - Finance BI (finance-bi.com)](https://finance-bi.com/blog/power-bi-basket-analysis/)

### Example 1
> Топ3 категорий, которые чаще всего встречаются в чеках совместно с отобранной категорией. Например, с категорией «молокопродукты» чаще всего встречаются в чеках «кондитерские изделия». Нужно в каждый чек залезть.
> Задача понять какая категория товаров чаще всего встречается в чеках вместе с выбранной категорией (в идеале топ3 наиболее часто встречающихся в чеках категории)

1) [Run code on Dax.do](https://dax.do/rKt1jW3ncLGGBw/)

```
VAR p_test = SELECTEDVALUE ( Sales[ProductKey] )
VAR ord =
    FILTER (
        CALCULATETABLE (
            GENERATE (
                VALUES ( Sales[Order Number] ),
                ROW (
                    "@чек",
                        CALCULATE (
                            COUNTROWS (
                                FILTER ( VALUES ( Sales[ProductKey] ), Sales[ProductKey] = p_test )
                            )
                        )
                )
            ),
            REMOVEFILTERS ( Sales )
        ),
        NOT ISBLANK ( [@чек] )
    )
VAR r1 =
    FILTER (
        GENERATE ( ord, CALCULATETABLE ( VALUES ( Sales[ProductKey] ) ) ),
        Sales[ProductKey] <> p_test
    )
VAR r2 =
    GROUPBY ( r1, Sales[ProductKey], "@Частота", SUMX ( CURRENTGROUP (), 1 ) )
VAR r3 =
    SELECTCOLUMNS (
        GROUPBY ( TOPN ( 3, r2, [@Частота] ), Sales[ProductKey] ),
        "@cod", Sales[ProductKey]
    )
VAR result =
    CONCATENATEX (
        FILTER ( CROSSJOIN ( ALL ( 'Product' ), r3 ), 'Product'[ProductKey] = [@cod] ),
        'Product'[Product Name],
        ", ",
        Product[Product Name], ASC
    )
RETURN    result
```

2. [Run code on Dax.do](https://dax.do/PSPrcrkpr3ePLQ/)

```
VAR p = SELECTEDVALUE ( Sales[ProductKey] )
VAR ord =
    FILTER (
        CALCULATETABLE (
            ADDCOLUMNS (
                SUMMARIZE ( Sales, Sales[Order Number] ),
                "@чек",
                    CALCULATE (
                        COUNTROWS ( FILTER ( VALUES ( Sales[ProductKey] ), Sales[ProductKey] = p ) )
                    )
            ),
            REMOVEFILTERS ( Sales )
        ),
        NOT ISBLANK ( [@чек] )
    )
VAR r1 =
    FILTER (
        GENERATE ( ord, CALCULATETABLE ( VALUES ( Sales[ProductKey] ) ) ),
        Sales[ProductKey] <> p
    )
VAR r2 =
    GROUPBY ( r1, Sales[ProductKey], "@Частота", SUMX ( CURRENTGROUP (), 1 ) )
VAR r3 =
    GROUPBY ( TOPN ( 3, r2, [@Частота] ), Sales[ProductKey] )
VAR result =
    CONCATENATEX (
        CALCULATETABLE (
            'Product',
            TREATAS ( r3, 'Product'[ProductKey] ),
            REMOVEFILTERS ( 'Product' )
        ),
        'Product'[Product Name],
        ", ",
        Product[Product Name], ASC
    )
RETURN result
```

3. Data model: PRODUCT -> ORDER_ITEM

```
// Example for ORDER_ITEM[ORDER_ID] = 3472929224
// Suppose the category PRODUCT[CATEGORY3_AXA] = "Шорты" is choosen in filter
DEFINE
    // Orders with buyed products from category: PRODUCT[CATEGORY3_AXA] = "Шорты"
    VAR ord =
        CALCULATETABLE (
            DISTINCT ( ORDER_ITEM[ORDER_ID] ),
            REMOVEFILTERS ( ORDER_ITEM ),
            KEEPFILTERS ( ORDER_ITEM[ORDER_ID] = 3472929224 )
        )
    // All products from ORD from other categories: PRODUCT[CATEGORY3_AXA] <> "Шорты"
    VAR prod =
        CALCULATETABLE (
            DISTINCT ( 'ORDER_ITEM'[PRODUCT_ID] ),
            ord,
            REMOVEFILTERS ( PRODUCT[CATEGORY3_AXA] ),
            RELATEDTABLE ( 'PRODUCT' ),
            PRODUCT[CATEGORY3_AXA] <> "Шорты"
        )
    VAR r1 =
        UNION (
            CALCULATETABLE (
                GENERATE (
                    ord,
                    SUMMARIZE (
                        CALCULATETABLE ( 'PRODUCT', TREATAS ( prod, 'PRODUCT'[PRODUCT_ID] ) ),
                        PRODUCT[CATEGORY3_AXA]
                        --PRODUCT[PRODUCT_ID]
                    )
                )
            ),
            ROW ( -- fake row for testing
                "ORDER_ID", 999999999,
                "CATEGORY3_AXA", "Кроссовки"
                --"PRODUCT_ID", 9999999999
            ),
            ROW ( -- fake row for testing
                "ORDER_ID", 111,
                "CATEGORY3_AXA", "Шорты джинсовые"
                --"PRODUCT_ID", 9177453506
            )
        )
    VAR r2 =
        GROUPBY (
            r1,
            'PRODUCT'[CATEGORY3_AXA],
            "@Частота", SUMX ( CURRENTGROUP (), 1 )
            --"@PID", MINX ( CURRENTGROUP (), PRODUCT[PRODUCT_ID] )
        )
    VAR result =
        CONCATENATEX (
            r2,
            'Product'[CATEGORY3_AXA] & ": " & [@Частота],
            ", ",
            [@Частота], DESC
        )
EVALUATE row("result", result)
--EVALUATE ord
--EVALUATE prod
--EVALUATE r1
--EVALUATE r2
```
*ord*

ORDER_ID |
--- |
3472929224

*prod*

PRODUCT_ID |
--- |
8841999520
9177453506
9177453486
9335401332
9413841339
9413841322
9594943279
9744296302

*r1*

ORDER_ID | CATEGORY3_AXA
--- | --- |
3472929224	| Шорты джинсовые
999999999	| Кроссовки
111	| Шорты джинсовые

*r2*

result |
--- |
Шорты джинсовые: 2, Кроссовки: 1
