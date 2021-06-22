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
Result |
--- |
Шорты джинсовые: 2, Кроссовки: 1
