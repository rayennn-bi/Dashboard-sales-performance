![Dashboard](Dashboard/Dashboard_Page_Sales_perfromance.png)
# Dashboard Sales & Shipment performance
Mini Project Dashboard Sales Performance

# Deskripsi Dashboard
Project ini adalah mini project dashboard untuk monitoring kinerja penjualan & pengiriman tahun berjalan (YTD) pada perusahaan ritel.

# Tools 
- sql (postgresql)
- power BI
- dbeaver (IDE)
- figma (mockup dashboard)

# Sumber dataset 
Database (postgresql) :
- data order
- data order_details
- data products
- data customer
- data shipment
- data shipper

# Data Dictionary


# SQL Script
```sql   
WITH line AS (
      SELECT
        o.order_id,
        o.order_date,
        o.required_date,
        o.shipped_date,
        o.ship_via,
        o.customer_id,
        o.employee_id,
        o.ship_name,
        o.ship_address,
        o.ship_city,
        o.ship_region,
        o.ship_postal_code,
        o.ship_country,
        o.freight,                             -- freight di header order
        od.product_id,
        od.unit_price       AS trx_unit_price, -- harga transaksi saat order
        od.quantity,
        od.discount,
        (od.unit_price * od.quantity * (1 - od.discount))::numeric(14,2) AS line_amount
      FROM orders o
      JOIN order_details od ON od.order_id = o.order_id
    ),
    order_totals AS (
      SELECT order_id, SUM(line_amount) AS order_amount
      FROM line
      GROUP BY order_id
    )
    SELECT
      -- Keys
      l.order_id,
      l.product_id,
      l.customer_id,
      l.employee_id,
      l.ship_via        AS shipper_id,
    
    -- Dates + handy parts
    TO_CHAR(l.order_date, 'YYYY-MM-DD') AS order_date,
    TO_CHAR(date_trunc('month', l.order_date), 'YYYY-MM-DD') AS month_order,
    EXTRACT(YEAR  FROM l.order_date)::int  AS order_year,
    EXTRACT(QUARTER FROM l.order_date)::int AS order_quarter,
    EXTRACT(MONTH FROM l.order_date)::int   AS order_month,
    TO_CHAR(l.order_date, 'Mon')            AS order_month_abbr,
    TO_CHAR(l.required_date, 'YYYY-MM-DD') AS required_date,
    TO_CHAR(l.shipped_date, 'YYYY-MM-DD') AS shipped_date,
    
    
    -- Status kirim
    CASE
    WHEN l.shipped_date IS NULL THEN 'Pending'
    WHEN l.shipped_date > l.required_date THEN 'Late'
    ELSE 'On time'
    END AS ship_status,
    
      -- Shipping info
      l.ship_name,
      l.ship_address,
      l.ship_city,
      l.ship_region,
      l.ship_postal_code,
      l.ship_country,
    
      -- Customer
      c.company_name        AS customer_name,
      c.contact_name        AS customer_contact,
      NULLIF(TRIM(c.phone), '') AS customer_phone,
      c.city                AS customer_city,
      c.region              AS customer_region,
      c.postal_code         AS customer_postal_code,
      c.country             AS customer_country,
    
      -- Employee (sales rep)
      (e.first_name || ' ' || e.last_name) AS employee_name,
      e.title         AS employee_title,
      e.city          AS employee_city,
      e.country       AS employee_country,
    
      -- Shipper
      s.company_name  AS shipper_name,
    
      -- Product + Category + price band (dari katalog saat ini)
      p.product_name,
      p.unit_price    AS catalog_unit_price,
      CASE
        WHEN p.unit_price < 10 THEN '<10'
        WHEN p.unit_price >= 10 AND p.unit_price < 20 THEN '10â€“20'
        WHEN p.unit_price >= 20 AND p.unit_price <= 50 THEN '20â€“50'
        ELSE '>50'
      END AS price_band,
      cat.category_id,
      cat.category_name,
    
      -- Transaksi per baris
      l.trx_unit_price,         -- harga transaksi (bukan katalog)
      l.quantity,
      l.discount,               -- 0..1
      (l.discount * 100.0)::numeric(5,2) AS discount_pct,
      l.line_amount             AS net_sales,        -- revenue murni per baris (tanpa freight)
    
      -- Alokasi freight proporsional berdasarkan porsi line_amount dalam order_amount
      (CASE WHEN ot.order_amount > 0
            THEN (l.freight * (l.line_amount / ot.order_amount))
            ELSE 0
       END)::numeric(14,2) AS freight_alloc,
    
      -- Total billed (jika ingin memasukkan freight)
      (l.line_amount
       + CASE WHEN ot.order_amount > 0
              THEN (l.freight * (l.line_amount / ot.order_amount))
              ELSE 0 END)::numeric(14,2) AS sales_plus_freight
    FROM line l
    JOIN order_totals ot ON ot.order_id = l.order_id
    LEFT JOIN products   p   ON p.product_id   = l.product_id
    LEFT JOIN categories cat ON cat.category_id = p.category_id
    LEFT JOIN customers  c   ON c.customer_id  = l.customer_id
    LEFT JOIN employees  e   ON e.employee_id  = l.employee_id
    LEFT JOIN shippers   s   ON s.shipper_id   = l.ship_via;
```

# Dashboard Overview
## Page Sales
![Dashboard](Dashboard/Dashboard_Page_Sales_perfromance.png)

## Page Shipment
![Dashboard](Dashboard/Dashboard_page_shipment.png)

# Key Insight
## SALES PERFORMANCE INSIGHT

### 1️. Revenue & Order Growth Solid

- Total Sales: $753.05K (+52.18% YoY)
- Total Orders: 473 (+37.90% YoY)
- Total Quantity: $28.60K (+31.05% YoY)
- AOV: $1.59K (+10.36% YoY)

📌 Insight :
Pertumbuhan revenue lebih cepat dibanding pertumbuhan order → strategi pricing, bundling, atau upselling berjalan efektif. Kenaikan AOV menunjukkan kualitas order meningkat, bukan hanya volume.


### 2️. Seasonal Trend & Volatility

Penjualan turun signifikan di April, lalu meningkat konsisten hingga puncak di September ($130K).
Oktober mengalami sharp drop, meskipun masih di atas beberapa bulan awal.

📌 Insight:
Ada pola musiman kuat (Q3 peak). Penurunan di Oktober bisa mengindikasikan:
efek pasca peak season, keterbatasan stok, atau bottleneck pengiriman (terkonfirmasi di dashboard shipment).


### 3️. Sales Contributor Analysis (Employee)

- Top performer menyumbang >20% total sales.
- Beberapa sales dengan order tinggi tapi AOV rendah.
- Ada gap performa antar sales yang cukup besar.

📌 Insight:
Perlu Segmentasi Sales Performance :
- High Order – Low AOV → fokus upselling
- Low Order – High AOV → fokus lead generation

### 4️. Product & Market Concentration Risk

Price band €20–€50 adalah kontributor terbesar (~€0.34M).
USA & Germany menyumbang mayoritas revenue.

📌 Insight:
Bisnis masih terkonsentrasi pada mid-price segment & beberapa negara utama.
Risiko: shock pasar / regulasi → revenue langsung terdampak.

## SHIPMENT PERFORMANCE INSIGHT
### 5️. Service Level Sangat Baik, Tapi Ada Early Warning

- On-Time Rate: 95.80%
- Late Rate: 4.20%
- Total Shipped Orders: 452 (+31.78%)

📌 Insight:
Secara SLA masih sangat sehat, namun tren late shipment meningkat seiring volume order naik → potensi future risk jika demand terus tumbuh.

### 6️. Late Shipment Spike di Mid-Year

Keterlambatan mulai meningkat di Mei–Juni, berbarengan dengan kenaikan order. Puncak order di Agustus–September → mulai muncul tekanan distribusi.

📌 Insight:
Masalah bukan vendor tunggal, tapi capacity planning. Sistem shipment belum sepenuhnya scalable terhadap growth.

### 7️. Freight Cost Inefficiency

- Total Freight Cost: €39.71K
- Biaya tertinggi: USA, Germany, Australia
- Cost tidak selalu linear dengan jumlah order.

📌 Insight:
Ada inefisiensi rute atau carrier selection, khususnya di negara dengan volume tinggi.
Potensi saving melalui: renegosiasi kontrak, shipment consolidation, atau zone-based pricing.

### 8️. Carrier Performance Comparison

Semua shipper memiliki on-time rate di atas 94%. Namun late rate dan freight cost berbeda signifikan.

📌 Insight:
Decision saat ini belum cost-performance optimized. Idealnya carrier dipilih berdasarkan Cost per On-Time Shipment, bukan hanya SLA.
