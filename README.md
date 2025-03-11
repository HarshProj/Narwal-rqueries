# Narwal-rqueries
## ** Average cost per PO (USD) - Monthly average cost for delivery per PO that have completed last mile delivery within the month**

 If you need insights on imports (POs, total invoice values), use the first query.
✅ If you need accurate cost tracking for deliveries with currency conversion, the second query is better.

SELECT 
    DATE_TRUNC('month', import_timestamp) AS month,
    COUNT(id) AS total_pos, 
    SUM(total_invoice_value) AS total_cost,
    ROUND(AVG(total_invoice_value), 2) AS avg_cost_per_po
FROM public.imports
WHERE DATE_PART('month', import_timestamp) = DATE_PART('month', import_timestamp) 
AND DATE_PART('year', import_timestamp) = DATE_PART('year', import_timestamp) 
GROUP BY month
ORDER BY month;

or

//better one
WITH completed_deliveries AS (
  SELECT 
    DATE_TRUNC('month', d.latest_event_timestamp) as month,
    d.id as delivery_id
  FROM deliveries d
  JOIN shipments s ON d.id = s.delivery_id
  WHERE s.transport_mode = 'local delivery'
  AND d.latest_event_timestamp >= '2024-01-01'
  AND d.latest_event_timestamp < '2025-04-01'
),
delivery_costs AS (
  SELECT 
    cd.month,
    dq.quotation_id,
    -- Convert all costs to USD using approximate exchange rates
    SUM(
      CASE qsf.currency
        WHEN 'USD' THEN qsf.cost * qsf.quantity
        WHEN 'EUR' THEN qsf.cost * qsf.quantity * 1.08  -- EUR to USD
        WHEN 'JPY' THEN qsf.cost * qsf.quantity * 0.0067  -- JPY to USD
        WHEN 'KRW' THEN qsf.cost * qsf.quantity * 0.00075  -- KRW to USD
        ELSE qsf.cost * qsf.quantity  -- Default to original value if currency not handled
      END
    ) as total_cost_usd
  FROM completed_deliveries cd
  JOIN delivery_quotations dq ON cd.delivery_id = dq.delivery_id
  JOIN quotation_schedules qs ON dq.quotation_id = qs.quotation_id
  JOIN quotation_schedule_fees qsf ON qs.id = qsf.schedule_id
  GROUP BY cd.month, dq.quotation_id
)
SELECT 
  month,
  COUNT(DISTINCT quotation_id) as number_of_deliveries,
  SUM(total_cost_usd) as total_cost_usd,
  ROUND(AVG(total_cost_usd), 2) as avg_cost_per_delivery_usd
FROM delivery_costs
GROUP BY month
ORDER BY month DESC;


/*Output*/

{
    "success": true,
    "data": [
        {
            "month": "2025-01-31T18:30:00.000Z",
            "number_of_deliveries": "294",
            "total_cost_usd": "52100304.19150000000000000",
            "avg_cost_per_delivery_usd": "177211.92"
        },
        {
            "month": "2024-12-31T18:30:00.000Z",
            "number_of_deliveries": "185",
            "total_cost_usd": "1763409.60625000000000000",
            "avg_cost_per_delivery_usd": "9531.94"
        },
        {
            "month": "2024-11-30T18:30:00.000Z",
            "number_of_deliveries": "156",
            "total_cost_usd": "179175.51185000000000000",
            "avg_cost_per_delivery_usd": "1148.56"
        },
        {
            "month": "2024-10-31T18:30:00.000Z",
            "number_of_deliveries": "5",
            "total_cost_usd": "3893.40000000000000000",
            "avg_cost_per_delivery_usd": "778.68"
        }
    ]
}



![image](https://github.com/user-attachments/assets/199fee88-5a08-46f2-a2a3-49ebd927d3db)













## **Average cost per KG (USD) - Monthly average cost for delivery per PO that have completed last mile delivery within the month**

WITH completed_deliveries AS (
  SELECT 
    DATE_TRUNC('month', d.latest_event_timestamp) as month,
    d.id as delivery_id
  FROM deliveries d
  JOIN shipments s ON d.id = s.delivery_id
  WHERE s.transport_mode = 'local delivery'
  AND d.latest_event_timestamp >= '2024-01-01'
  AND d.latest_event_timestamp < '2025-04-01'
),
delivery_weights AS (
  -- Get weights from quotation packages
  SELECT 
    cd.month,
    cd.delivery_id,
    SUM(qp.weight) as total_weight
  FROM completed_deliveries cd
  JOIN delivery_quotations dq ON cd.delivery_id = dq.delivery_id
  JOIN quotation_packages qp ON dq.quotation_id = qp.quotation_id
  GROUP BY cd.month, cd.delivery_id
),
delivery_costs AS (
  SELECT 
    cd.month,
    cd.delivery_id,
    -- Convert all costs to USD using approximate exchange rates
    SUM(
      CASE qsf.currency
        WHEN 'USD' THEN qsf.cost * qsf.quantity
        WHEN 'EUR' THEN qsf.cost * qsf.quantity * 1.08  -- EUR to USD
        WHEN 'JPY' THEN qsf.cost * qsf.quantity * 0.0067  -- JPY to USD
        WHEN 'KRW' THEN qsf.cost * qsf.quantity * 0.00075  -- KRW to USD
        ELSE qsf.cost * qsf.quantity  -- Default to original value if currency not handled
      END
    ) as total_cost_usd
  FROM completed_deliveries cd
  JOIN delivery_quotations dq ON cd.delivery_id = dq.delivery_id
  JOIN quotation_schedules qs ON dq.quotation_id = qs.quotation_id
  JOIN quotation_schedule_fees qsf ON qs.id = qsf.schedule_id
  GROUP BY cd.month, cd.delivery_id
)
SELECT 
  dw.month,
  COUNT(DISTINCT dw.delivery_id) as number_of_deliveries,
  ROUND(SUM(dw.total_weight), 2) as total_weight_kg,
  ROUND(SUM(dc.total_cost_usd), 2) as total_cost_usd,
  ROUND(SUM(dc.total_cost_usd) / NULLIF(SUM(dw.total_weight), 0), 2) as avg_cost_per_kg_usd
FROM delivery_weights dw
JOIN delivery_costs dc ON dw.delivery_id = dc.delivery_id AND dw.month = dc.month
WHERE dw.total_weight > 0  -- Exclude deliveries with no weight data
GROUP BY dw.month
ORDER BY dw.month DESC;


/*Output*/

{
    "success": true,
    "data": [
        {
            "month": "2025-01-31T18:30:00.000Z",
            "number_of_deliveries": "3117",
            "total_weight_kg": "43264055.42",
            "total_cost_usd": "52100304.19",
            "avg_cost_per_kg_usd": "1.20"
        },
        {
            "month": "2024-12-31T18:30:00.000Z",
            "number_of_deliveries": "1520",
            "total_weight_kg": "5405754.09",
            "total_cost_usd": "1763409.61",
            "avg_cost_per_kg_usd": "0.33"
        },
        {
            "month": "2024-11-30T18:30:00.000Z",
            "number_of_deliveries": "441",
            "total_weight_kg": "310377.20",
            "total_cost_usd": "179175.51",
            "avg_cost_per_kg_usd": "0.58"
        },
        {
            "month": "2024-10-31T18:30:00.000Z",
            "number_of_deliveries": "5",
            "total_weight_kg": "1341.60",
            "total_cost_usd": "3893.40",
            "avg_cost_per_kg_usd": "2.90"
        }
    ]
}

![image](https://github.com/user-attachments/assets/4360ad0c-f862-4ff5-8a54-c239f0947dec)


## **Cargo volumes in CBM - Monthly volume of inbound and outbound cargo in total and also by hub**

 If shipments reliably records both import and export activity → Use the second query.
✅ If imports and exports are managed independently and shipments doesn't always track both → Use the first query.
/*
SELECT 
    DATE_TRUNC('month', s.latest_event_timestamp) AS month,
    s.origin_city AS hub,
    SUM(COALESCE(ip.cbm, 0)) AS inbound_volume,
    SUM(COALESCE(ep.cbm, 0)) AS outbound_volume,
    SUM(COALESCE(ip.cbm, 0) + COALESCE(ep.cbm, 0)) AS total_volume
FROM shipments s
LEFT JOIN import_packages ip ON s.id = ip.import_id  -- Inbound shipments
LEFT JOIN export_packages ep ON s.id = ep.export_id  -- Outbound shipments
WHERE s.latest_event_timestamp IS NOT NULL  
GROUP BY month, hub
ORDER BY month DESC, hub;
*/
or


WITH monthly_exports AS (
  SELECT 
    DATE_TRUNC('month', e.export_timestamp) AS month,
    e.port_from as hub,
    SUM(ep.cbm) as total_cbm
  FROM exports e
  JOIN export_packages ep ON e.id = ep.export_id
  WHERE e.export_timestamp IS NOT NULL
  GROUP BY DATE_TRUNC('month', e.export_timestamp), e.port_from
),
monthly_imports AS (
  SELECT 
    DATE_TRUNC('month', i.import_timestamp) AS month,
    i.warehouse as hub,
    SUM(ip.cbm) as total_cbm
  FROM imports i
  JOIN import_packages ip ON i.id = ip.import_id
  WHERE i.import_timestamp IS NOT NULL
  GROUP BY DATE_TRUNC('month', i.import_timestamp), i.warehouse
)
SELECT 
  COALESCE(e.month, i.month) as month,
  COALESCE(e.hub, i.hub) as hub,
  COALESCE(e.total_cbm, 0) as export_cbm,
  COALESCE(i.total_cbm, 0) as import_cbm,
  COALESCE(e.total_cbm, 0) + COALESCE(i.total_cbm, 0) as total_cbm
FROM monthly_exports e
FULL OUTER JOIN monthly_imports i 
  ON e.month = i.month 
  AND e.hub = i.hub
WHERE COALESCE(e.month, i.month) IS NOT NULL
ORDER BY month DESC, hub;



/*Output*/
{
    "success": true,
    "data": [
        {
            "month": "2025-11-30T18:30:00.000Z",
            "hub": null,
            "export_cbm": "19.05800",
            "import_cbm": "0",
            "total_cbm": "19.05800"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "1F",
            "export_cbm": "0",
            "import_cbm": "1149.90900",
            "total_cbm": "1149.90900"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "2F",
            "export_cbm": "0",
            "import_cbm": "85.12400",
            "total_cbm": "85.12400"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "AMS",
            "export_cbm": "14.44900",
            "import_cbm": "0",
            "total_cbm": "14.44900"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "CNSGH",
            "export_cbm": "9.36500",
            "import_cbm": "0",
            "total_cbm": "9.36500"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "CNSHA",
            "export_cbm": "0.00100",
            "import_cbm": "0",
            "total_cbm": "0.00100"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "HAESUNG",
            "export_cbm": "0",
            "import_cbm": "4.02200",
            "total_cbm": "4.02200"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "ICN",
            "export_cbm": "158.67700",
            "import_cbm": "0",
            "total_cbm": "158.67700"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "JPCHI",
            "export_cbm": "0.02700",
            "import_cbm": "0",
            "total_cbm": "0.02700"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "JPUKB",
            "export_cbm": "91.52200",
            "import_cbm": "0",
            "total_cbm": "91.52200"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "KIX",
            "export_cbm": "36.63700",
            "import_cbm": "0",
            "total_cbm": "36.63700"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "KRPUS",
            "export_cbm": "339.25000",
            "import_cbm": "0",
            "total_cbm": "339.25000"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "NLRTM",
            "export_cbm": "51.75900",
            "import_cbm": "0",
            "total_cbm": "51.75900"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "OUTSIDE",
            "export_cbm": "0",
            "import_cbm": "0.49500",
            "total_cbm": "0.49500"
        },
        ...
    ]
    }

![image](https://github.com/user-attachments/assets/dcbec013-eb90-4711-9960-20628a694707)




## **List of High-cost shipment - Monthly list of high-cost delivery (only including shipment cost) that have completed onboard last mile delivery within the month**

WITH monthly_shipments AS (
    SELECT 
        s.id as shipment_id,
        s.customer,
        s.origin,
        s.destination,
        s.transport_mode,
        s.latest_event_timestamp,
        DATE_TRUNC('month', s.latest_event_timestamp) as delivery_month,
        dq.quotation_id,
        SUM(qsf.cost * qsf.quantity) as total_cost,
        qsf.currency
    FROM shipments s
    JOIN deliveries d ON s.delivery_id = d.id
    JOIN delivery_quotations dq ON d.id = dq.delivery_id
    JOIN quotation_schedules qs ON dq.quotation_id = qs.quotation_id
    JOIN quotation_schedule_fees qsf ON qs.id = qsf.schedule_id
    WHERE s.latest_event_timestamp >= (CURRENT_DATE - INTERVAL '3 months')
    AND s.latest_event_timestamp < CURRENT_DATE
    GROUP BY 
        s.id,
        s.customer,
        s.origin,
        s.destination,
        s.transport_mode,
        s.latest_event_timestamp,
        DATE_TRUNC('month', s.latest_event_timestamp),
        dq.quotation_id,
        qsf.currency
)
SELECT 
    TO_CHAR(delivery_month, 'YYYY-MM') as month,
    shipment_id,
    customer,
    origin,
    destination,
    transport_mode,
    TO_CHAR(latest_event_timestamp, 'YYYY-MM-DD') as delivery_date,
    ROUND(total_cost, 2) as total_cost,
    currency
FROM monthly_shipments
ORDER BY 
    delivery_month DESC,
    total_cost DESC
LIMIT 100;


/*Output*/

{
    "success": true,
    "data": [
        {
            "month": "2025-02",
            "shipment_id": "523809",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523791",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523802",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523804",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523796",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523801",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523805",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        {
            "month": "2025-02",
            "shipment_id": "523792",
            "customer": "60YC",
            "origin": "JPUKB",
            "destination": "JPKII",
            "transport_mode": "local delivery",
            "delivery_date": "2025-02-14",
            "total_cost": "12060690.00",
            "currency": "KRW"
        },
        ...]
}
![image](https://github.com/user-attachments/assets/60f10a1a-e9e4-43c9-a347-2ac5f89d6ba3)


## //Ratio on freight mode - Monthly ratio of freight modes based on cost, number of POs, and weight. Total ratio including all the shipments and ratio by hub is required.

try to write fully optimized query
WITH monthly_metrics AS (
  SELECT 
  DATE_TRUNC('month', s.latest_event_timestamp) as month,
  s.transport_mode,
  s.origin as hub,
  COUNT(DISTINCT po.number) as po_count,
  COUNT(DISTINCT s.id) as shipment_count,
  SUM(qp.weight) as total_weight,
  SUM(
    CASE qsf.currency
      WHEN 'USD' THEN qsf.cost * qsf.quantity
      WHEN 'EUR' THEN qsf.cost * qsf.quantity * 1.08  -- Approximate EUR to USD
      WHEN 'JPY' THEN qsf.cost * qsf.quantity * 0.0067  -- Approximate JPY to USD
      WHEN 'KRW' THEN qsf.cost * qsf.quantity * 0.00075  -- Approximate KRW to USD
    END
  ) as total_cost_usd
FROM shipments s
LEFT JOIN deliveries d ON s.delivery_id = d.id
LEFT JOIN delivery_quotations dq ON d.id = dq.delivery_id
LEFT JOIN quotations q ON dq.quotation_id = q.id
LEFT JOIN quotation_packages qp ON q.id = qp.quotation_id
LEFT JOIN quotation_schedules qs ON q.id = qs.quotation_id
LEFT JOIN quotation_schedule_fees qsf ON qs.id = qsf.schedule_id
LEFT JOIN export_package_po_numbers epn ON epn.export_package_id = qp.id
LEFT JOIN purchase_orders po ON epn.po_number = po.number
WHERE s.latest_event_timestamp >= '2024-01-01'
AND s.latest_event_timestamp < '2025-04-01'
GROUP BY 
  DATE_TRUNC('month', s.latest_event_timestamp),
  s.transport_mode,
  s.origin
),
monthly_totals AS (
SELECT 
  month,
  SUM(po_count) as total_po_count,
  SUM(shipment_count) as total_shipment_count,
  SUM(total_weight) as total_weight,
  SUM(total_cost_usd) as total_cost_usd
FROM monthly_metrics
GROUP BY month
),
hub_totals AS (
SELECT 
  month,
  hub,
  SUM(po_count) as hub_po_count,
  SUM(shipment_count) as hub_shipment_count,
  SUM(total_weight) as hub_weight,
  SUM(total_cost_usd) as hub_cost_usd
FROM monthly_metrics
GROUP BY month, hub
)

SELECT 
mm.month,
mm.hub,
mm.transport_mode,
mm.po_count,
ROUND(mm.po_count * 100.0 / NULLIF(mt.total_po_count, 0), 2) as po_ratio_total,
ROUND(mm.po_count * 100.0 / NULLIF(ht.hub_po_count, 0), 2) as po_ratio_hub,
mm.shipment_count,
ROUND(mm.shipment_count * 100.0 / NULLIF(mt.total_shipment_count, 0), 2) as shipment_ratio_total,
ROUND(mm.shipment_count * 100.0 / NULLIF(ht.hub_shipment_count, 0), 2) as shipment_ratio_hub,
ROUND(mm.total_weight, 2) as total_weight_kg,
ROUND(mm.total_weight * 100.0 / NULLIF(mt.total_weight, 0), 2) as weight_ratio_total,
ROUND(mm.total_weight * 100.0 / NULLIF(ht.hub_weight, 0), 2) as weight_ratio_hub,
ROUND(mm.total_cost_usd, 2) as total_cost_usd,
ROUND(mm.total_cost_usd * 100.0 / NULLIF(mt.total_cost_usd, 0), 2) as cost_ratio_total,
ROUND(mm.total_cost_usd * 100.0 / NULLIF(ht.hub_cost_usd, 0), 2) as cost_ratio_hub
FROM monthly_metrics mm
JOIN monthly_totals mt ON mm.month = mt.month
JOIN hub_totals ht ON mm.month = ht.month AND mm.hub = ht.hub
WHERE mm.transport_mode IS NOT NULL
ORDER BY 
mm.month DESC,
mm.hub,
mm.total_cost_usd DESC;




/*Output*/
{
    "success": true,
    "data": [
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "AMS",
            "transport_mode": "local delivery",
            "po_count": "0",
            "po_ratio_total": "0.00",
            "po_ratio_hub": "0.00",
            "shipment_count": "1",
            "shipment_ratio_total": "0.00",
            "shipment_ratio_hub": "2.13",
            "total_weight_kg": null,
            "weight_ratio_total": null,
            "weight_ratio_hub": null,
            "total_cost_usd": null,
            "cost_ratio_total": null,
            "cost_ratio_hub": null
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "AMS",
            "transport_mode": "air freight",
            "po_count": "111",
            "po_ratio_total": "2.81",
            "po_ratio_hub": "100.00",
            "shipment_count": "46",
            "shipment_ratio_total": "0.02",
            "shipment_ratio_hub": "97.87",
            "total_weight_kg": "4156.34",
            "weight_ratio_total": "0.00",
            "weight_ratio_hub": "100.00",
            "total_cost_usd": "120800.83",
            "cost_ratio_total": "0.01",
            "cost_ratio_hub": "100.00"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "CNSGH",
            "transport_mode": "courier",
            "po_count": "2",
            "po_ratio_total": "0.05",
            "po_ratio_hub": "33.33",
            "shipment_count": "2",
            "shipment_ratio_total": "0.00",
            "shipment_ratio_hub": "66.67",
            "total_weight_kg": "6.00",
            "weight_ratio_total": "0.00",
            "weight_ratio_hub": "0.84",
            "total_cost_usd": "299.00",
            "cost_ratio_total": "0.00",
            "cost_ratio_hub": "56.52"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "CNSGH",
            "transport_mode": "hand carry",
            "po_count": "4",
            "po_ratio_total": "0.10",
            "po_ratio_hub": "66.67",
            "shipment_count": "1",
            "shipment_ratio_total": "0.00",
            "shipment_ratio_hub": "33.33",
            "total_weight_kg": "710.00",
            "weight_ratio_total": "0.00",
            "weight_ratio_hub": "99.16",
            "total_cost_usd": "230.00",
            "cost_ratio_total": "0.00",
            "cost_ratio_hub": "43.48"
        },
        ...]
}
![image](https://github.com/user-attachments/assets/bf53a258-b098-4a04-93d1-2e40db3c8c11)


## // Ratio of key hub vs non-key hub (based on cost, number of POs, and weight) - Monthly ratio of completed last-mile delivery at key hubs vs non-key hubs based on cost, number of POs, and weight.
 WITH delivery_metrics AS (
  SELECT 
    DATE_TRUNC('month', s.latest_event_timestamp) as month,
    CASE 
      WHEN s.origin IN ('ICN', 'JPUKB', 'KRPUS') THEN 'Key Hub'
      ELSE 'Non-Key Hub'
    END as hub_type,
    s.origin as hub,
    -- Count distinct POs
    COUNT(DISTINCT po.number) as po_count,
    -- Count shipments
    COUNT(DISTINCT s.id) as shipment_count,
    -- Sum weights
    SUM(qp.weight) as total_weight,
    -- Calculate costs in USD
    SUM(
      CASE qsf.currency
        WHEN 'USD' THEN qsf.cost * qsf.quantity
        WHEN 'EUR' THEN qsf.cost * qsf.quantity * 1.08  -- Approximate EUR to USD
        WHEN 'JPY' THEN qsf.cost * qsf.quantity * 0.0067  -- Approximate JPY to USD
        WHEN 'KRW' THEN qsf.cost * qsf.quantity * 0.00075  -- Approximate KRW to USD
      END
    ) as total_cost_usd,
    -- Count completed last mile deliveries
    COUNT(DISTINCT CASE 
      WHEN s.transport_mode = 'local delivery' 
      AND d.latest_event_timestamp IS NOT NULL 
      THEN s.id 
    END) as completed_last_mile_count,
    -- Sum weight of completed last mile deliveries
    SUM(CASE 
      WHEN s.transport_mode = 'local delivery' 
      AND d.latest_event_timestamp IS NOT NULL 
      THEN qp.weight 
      ELSE 0 
    END) as completed_last_mile_weight,
    -- Sum cost of completed last mile deliveries
    SUM(CASE 
      WHEN s.transport_mode = 'local delivery' 
      AND d.latest_event_timestamp IS NOT NULL 
      THEN 
        CASE qsf.currency
          WHEN 'USD' THEN qsf.cost * qsf.quantity
          WHEN 'EUR' THEN qsf.cost * qsf.quantity * 1.08
          WHEN 'JPY' THEN qsf.cost * qsf.quantity * 0.0067
          WHEN 'KRW' THEN qsf.cost * qsf.quantity * 0.00075
        END
      ELSE 0 
    END) as completed_last_mile_cost_usd
  FROM shipments s
  LEFT JOIN deliveries d ON s.delivery_id = d.id
  LEFT JOIN delivery_quotations dq ON d.id = dq.delivery_id
  LEFT JOIN quotations q ON dq.quotation_id = q.id
  LEFT JOIN quotation_packages qp ON q.id = qp.quotation_id
  LEFT JOIN quotation_schedules qs ON q.id = qs.quotation_id
  LEFT JOIN quotation_schedule_fees qsf ON qs.id = qsf.schedule_id
  LEFT JOIN export_package_po_numbers epn ON epn.export_package_id = qp.id
  LEFT JOIN purchase_orders po ON epn.po_number = po.number
  WHERE s.latest_event_timestamp >= '2024-01-01'
  AND s.latest_event_timestamp < '2025-04-01'
  GROUP BY 
    DATE_TRUNC('month', s.latest_event_timestamp),
    hub_type,
    s.origin
),
monthly_totals AS (
  SELECT 
    month,
    SUM(po_count) as total_po_count,
    SUM(shipment_count) as total_shipment_count,
    SUM(total_weight) as total_weight,
    SUM(total_cost_usd) as total_cost_usd,
    SUM(completed_last_mile_count) as total_completed_last_mile_count,
    SUM(completed_last_mile_weight) as total_completed_last_mile_weight,
    SUM(completed_last_mile_cost_usd) as total_completed_last_mile_cost_usd
  FROM delivery_metrics
  GROUP BY month
)

SELECT 
  dm.month,
  dm.hub_type,
  dm.hub,
  -- Overall metrics
  dm.po_count,
  ROUND(dm.po_count * 100.0 / NULLIF(mt.total_po_count, 0), 2) as po_ratio,
  dm.shipment_count,
  ROUND(dm.shipment_count * 100.0 / NULLIF(mt.total_shipment_count, 0), 2) as shipment_ratio,
  ROUND(dm.total_weight, 2) as total_weight_kg,
  ROUND(dm.total_weight * 100.0 / NULLIF(mt.total_weight, 0), 2) as weight_ratio,
  ROUND(dm.total_cost_usd, 2) as total_cost_usd,
  ROUND(dm.total_cost_usd * 100.0 / NULLIF(mt.total_cost_usd, 0), 2) as cost_ratio,
  -- Last mile delivery metrics
  dm.completed_last_mile_count,
  ROUND(dm.completed_last_mile_count * 100.0 / NULLIF(mt.total_completed_last_mile_count, 0), 2) as last_mile_count_ratio,
  ROUND(dm.completed_last_mile_weight, 2) as completed_last_mile_weight_kg,
  ROUND(dm.completed_last_mile_weight * 100.0 / NULLIF(mt.total_completed_last_mile_weight, 0), 2) as last_mile_weight_ratio,
  ROUND(dm.completed_last_mile_cost_usd, 2) as completed_last_mile_cost_usd,
  ROUND(dm.completed_last_mile_cost_usd * 100.0 / NULLIF(mt.total_completed_last_mile_cost_usd, 0), 2) as last_mile_cost_ratio,
  -- Last mile completion rates within hub
  ROUND(dm.completed_last_mile_count * 100.0 / NULLIF(dm.shipment_count, 0), 2) as hub_last_mile_completion_rate,
  ROUND(dm.completed_last_mile_weight * 100.0 / NULLIF(dm.total_weight, 0), 2) as hub_weight_completion_rate,
  ROUND(dm.completed_last_mile_cost_usd * 100.0 / NULLIF(dm.total_cost_usd, 0), 2) as hub_cost_completion_rate
FROM delivery_metrics dm
JOIN monthly_totals mt ON dm.month = mt.month
WHERE dm.month >= '2025-01-01'  -- Focus on recent months
ORDER BY 
  dm.month DESC,
  dm.hub_type,
  dm.total_cost_usd DESC;


/*Output*/
{
    "success": true,
    "data": [
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Key Hub",
            "hub": "ICN",
            "po_count": "721",
            "po_ratio": "19.05",
            "shipment_count": "612",
            "shipment_ratio": "0.20",
            "total_weight_kg": "31616585.18",
            "weight_ratio": "20.15",
            "total_cost_usd": "189103560.36",
            "cost_ratio": "21.48",
            "completed_last_mile_count": "434",
            "last_mile_count_ratio": "0.14",
            "completed_last_mile_weight_kg": "31019974.92",
            "last_mile_weight_ratio": "20.29",
            "completed_last_mile_cost_usd": "184082963.61",
            "last_mile_cost_ratio": "21.43",
            "hub_last_mile_completion_rate": "70.92",
            "hub_weight_completion_rate": "98.11",
            "hub_cost_completion_rate": "97.35"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Key Hub",
            "hub": "KRPUS",
            "po_count": "422",
            "po_ratio": "11.15",
            "shipment_count": "2656",
            "shipment_ratio": "0.87",
            "total_weight_kg": "3299252.04",
            "weight_ratio": "2.10",
            "total_cost_usd": "14408073.50",
            "cost_ratio": "1.64",
            "completed_last_mile_count": "1",
            "last_mile_count_ratio": "0.00",
            "completed_last_mile_weight_kg": "154371.50",
            "last_mile_weight_ratio": "0.10",
            "completed_last_mile_cost_usd": "160325.00",
            "last_mile_cost_ratio": "0.02",
            "hub_last_mile_completion_rate": "0.04",
            "hub_weight_completion_rate": "4.68",
            "hub_cost_completion_rate": "1.11"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Key Hub",
            "hub": "JPUKB",
            "po_count": "713",
            "po_ratio": "18.84",
            "shipment_count": "294974",
            "shipment_ratio": "96.55",
            "total_weight_kg": "324476.81",
            "weight_ratio": "0.21",
            "total_cost_usd": "2507634.20",
            "cost_ratio": "0.28",
            "completed_last_mile_count": "294947",
            "last_mile_count_ratio": "98.28",
            "completed_last_mile_weight_kg": "312803.91",
            "last_mile_weight_ratio": "0.20",
            "completed_last_mile_cost_usd": "1967489.07",
            "last_mile_cost_ratio": "0.23",
            "hub_last_mile_completion_rate": "99.99",
            "hub_weight_completion_rate": "96.40",
            "hub_cost_completion_rate": "78.46"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Non-Key Hub",
            "hub": "JPCHI",
            "po_count": "0",
            "po_ratio": "0.00",
            "shipment_count": "1",
            "shipment_ratio": "0.00",
            "total_weight_kg": null,
            "weight_ratio": null,
            "total_cost_usd": null,
            "cost_ratio": null,
            "completed_last_mile_count": "0",
            "last_mile_count_ratio": "0.00",
            "completed_last_mile_weight_kg": "0.00",
            "last_mile_weight_ratio": "0.00",
            "completed_last_mile_cost_usd": "0.00",
            "last_mile_cost_ratio": "0.00",
            "hub_last_mile_completion_rate": "0.00",
            "hub_weight_completion_rate": null,
            "hub_cost_completion_rate": null
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Non-Key Hub",
            "hub": null,
            "po_count": "1584",
            "po_ratio": "41.85",
            "shipment_count": "7103",
            "shipment_ratio": "2.32",
            "total_weight_kg": "121541540.53",
            "weight_ratio": "77.47",
            "total_cost_usd": "673075596.72",
            "cost_ratio": "76.47",
            "completed_last_mile_count": "4702",
            "last_mile_count_ratio": "1.57",
            "completed_last_mile_weight_kg": "121414784.57",
            "last_mile_weight_ratio": "79.41",
            "completed_last_mile_cost_usd": "672835520.96",
            "last_mile_cost_ratio": "78.32",
            "hub_last_mile_completion_rate": "66.20",
            "hub_weight_completion_rate": "99.90",
            "hub_cost_completion_rate": "99.96"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Non-Key Hub",
            "hub": "KIX",
            "po_count": "190",
            "po_ratio": "5.02",
            "shipment_count": "83",
            "shipment_ratio": "0.03",
            "total_weight_kg": "77913.70",
            "weight_ratio": "0.05",
            "total_cost_usd": "872881.75",
            "cost_ratio": "0.10",
            "completed_last_mile_count": "0",
            "last_mile_count_ratio": "0.00",
            "completed_last_mile_weight_kg": "0.00",
            "last_mile_weight_ratio": "0.00",
            "completed_last_mile_cost_usd": "0.00",
            "last_mile_cost_ratio": "0.00",
            "hub_last_mile_completion_rate": "0.00",
            "hub_weight_completion_rate": "0.00",
            "hub_cost_completion_rate": "0.00"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub_type": "Non-Key Hub",
            "hub": "AMS",
            "po_count": "111",
            "po_ratio": "2.93",
            "shipment_count": "47",
            "shipment_ratio": "0.02",
            "total_weight_kg": "4156.34",
            "weight_ratio": "0.00",
            "total_cost_usd": "120800.83",
            "cost_ratio": "0.01",
            "completed_last_mile_count": "1",
            "last_mile_count_ratio": "0.00",
            "completed_last_mile_weight_kg": "0.00",
            "last_mile_weight_ratio": "0.00",
            "completed_last_mile_cost_usd": "0.00",
            "last_mile_cost_ratio": "0.00",
            "hub_last_mile_completion_rate": "2.13",
            "hub_weight_completion_rate": "0.00",
            "hub_cost_completion_rate": "0.00"
        },
        ...]
}

![image](https://github.com/user-attachments/assets/8432b1d3-9801-475c-8912-c34e25e4fa8e)


## //Consolidation Rate - Number of POs shipped / Number of shipments for air freight, sea freight, and trucking.
 WITH storage_durations AS (
    SELECT 
        DATE_TRUNC('month', de.event_timestamp) as month,
        s.origin as hub,
        s.origin_city,
        s.id as shipment_id,
        -- Get first arrival event
        MIN(de.event_timestamp) as arrival_date,
        -- Get last departure event
        MAX(de.event_timestamp) as departure_date,
        -- Calculate storage duration in days
        EXTRACT(EPOCH FROM (MAX(de.event_timestamp) - MIN(de.event_timestamp)))/86400.0 as storage_days
    FROM shipments s
    JOIN delivery_events de ON s.delivery_id = de.delivery_id
    WHERE 
        -- Focus on most recent data
        de.event_timestamp >= '2025-02-01'
        AND de.event_timestamp < '2025-03-01'
        -- Ensure valid storage duration
        AND s.latest_event_timestamp > s.earliest_event_timestamp
        AND s.origin IS NOT NULL
    GROUP BY 
        DATE_TRUNC('month', de.event_timestamp),
        s.origin,
        s.origin_city,
        s.id
    HAVING 
        MIN(de.event_timestamp) != MAX(de.event_timestamp)
)

SELECT 
    month,
    hub,
    origin_city,
    -- Volume metrics
    COUNT(DISTINCT shipment_id) as total_shipments,
    -- Storage duration metrics
    ROUND(AVG(storage_days)::numeric, 2) as avg_storage_days,
    ROUND(MIN(storage_days)::numeric, 2) as min_storage_days,
    ROUND(MAX(storage_days)::numeric, 2) as max_storage_days,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY storage_days)::numeric, 2) as median_storage_days,
    -- Storage duration distribution
    COUNT(CASE WHEN storage_days <= 7 THEN 1 END) as within_week,
    COUNT(CASE WHEN storage_days > 7 AND storage_days <= 14 THEN 1 END) as within_two_weeks,
    COUNT(CASE WHEN storage_days > 14 AND storage_days <= 30 THEN 1 END) as within_month,
    COUNT(CASE WHEN storage_days > 30 THEN 1 END) as over_month,
    -- Storage efficiency metrics
    ROUND(COUNT(CASE WHEN storage_days <= 7 THEN 1 END)::numeric * 100 / 
        NULLIF(COUNT(*), 0)::numeric, 2) as within_week_percentage,
    ROUND(COUNT(CASE WHEN storage_days <= 14 THEN 1 END)::numeric * 100 / 
        NULLIF(COUNT(*), 0)::numeric, 2) as within_two_weeks_percentage,
    -- Additional metrics for analysis
    ROUND(STDDEV(storage_days)::numeric, 2) as storage_days_stddev
FROM storage_durations
GROUP BY 
    month,
    hub,
    origin_city
HAVING COUNT(DISTINCT shipment_id) > 0
ORDER BY 
    avg_storage_days DESC
LIMIT 100;

![image](https://github.com/user-attachments/assets/5b6e05e1-cb55-4123-af34-0caf43c6e29b)
![image](https://github.com/user-attachments/assets/ae4b9bde-2a13-4691-9a31-6b05c880d496)

/*Output*/
{
    "success": true,
    "data": [
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "SIN",
            "origin_city": "Busan",
            "total_shipments": "1",
            "avg_storage_days": "20.74",
            "min_storage_days": "20.74",
            "max_storage_days": "20.74",
            "median_storage_days": "20.74",
            "within_week": "0",
            "within_two_weeks": "0",
            "within_month": "1",
            "over_month": "0",
            "within_week_percentage": "0.00",
            "within_two_weeks_percentage": "0.00",
            "storage_days_stddev": null
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "ICN",
            "origin_city": null,
            "total_shipments": "418",
            "avg_storage_days": "20.47",
            "min_storage_days": "4.81",
            "max_storage_days": "21.46",
            "median_storage_days": "20.46",
            "within_week": "1",
            "within_two_weeks": "0",
            "within_month": "417",
            "over_month": "0",
            "within_week_percentage": "0.24",
            "within_two_weeks_percentage": "0.24",
            "storage_days_stddev": "0.80"
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "AMS",
            "origin_city": null,
            "total_shipments": "1",
            "avg_storage_days": "17.62",
            "min_storage_days": "17.62",
            "max_storage_days": "17.62",
            "median_storage_days": "17.62",
            "within_week": "0",
            "within_two_weeks": "0",
            "within_month": "1",
            "over_month": "0",
            "within_week_percentage": "0.00",
            "within_two_weeks_percentage": "0.00",
            "storage_days_stddev": null
        },
        {
            "month": "2025-01-31T18:30:00.000Z",
            "hub": "RTM",
            "origin_city": "Rotterdam",
            "total_shipments": "1",
            "avg_storage_days": "16.76",
            "min_storage_days": "16.76",
            "max_storage_days": "16.76",
            "median_storage_days": "16.76",
            "within_week": "0",
            "within_two_weeks": "0",
            "within_month": "1",
            "over_month": "0",
            "within_week_percentage": "0.00",
            "within_two_weeks_percentage": "0.00",
            "storage_days_stddev": null
        },
        ...]
}



## //Last mile delivery
WITH last_mile_deliveries AS (
  SELECT 
    s.id as shipment_id,
    s.customer,
    s.origin,
    s.origin_city,
    s.destination,
    s.transport_mode,
    s.earliest_event_timestamp,
    s.latest_event_timestamp,
    EXTRACT(EPOCH FROM (s.latest_event_timestamp - s.earliest_event_timestamp))/3600/24 as delivery_days,
    s.forwarder
  FROM shipments s
  WHERE 
    s.transport_mode = 'local delivery'
    AND s.earliest_event_timestamp IS NOT NULL 
    AND s.latest_event_timestamp IS NOT NULL
    AND s.earliest_event_timestamp >= '2025-01-01'
)
SELECT 
  customer,
  forwarder,
  COUNT(*) as total_deliveries,
  ROUND(AVG(delivery_days), 1) as avg_delivery_days,
  ROUND(MIN(delivery_days), 1) as min_delivery_days,
  ROUND(MAX(delivery_days), 1) as max_delivery_days,
  ROUND(COUNT(CASE WHEN delivery_days <= 1 THEN 1 END) * 100.0 / COUNT(*), 1) as same_day_delivery_pct,
  ROUND(COUNT(CASE WHEN delivery_days <= 2 THEN 1 END) * 100.0 / COUNT(*), 1) as within_2days_pct,
  origin,
  destination
FROM last_mile_deliveries
GROUP BY customer, forwarder, origin, destination
ORDER BY total_deliveries DESC;


/*Output*/
{
    "success": true,
    "data": [
        {
            "customer": "6549",
            "forwarder": null,
            "total_deliveries": "1298",
            "avg_delivery_days": "0.6",
            "min_delivery_days": "-1.5",
            "max_delivery_days": "34.6",
            "same_day_delivery_pct": "99.9",
            "within_2days_pct": "99.9",
            "origin": null,
            "destination": "CNZOS"
        },
        {
            "customer": "6549",
            "forwarder": null,
            "total_deliveries": "432",
            "avg_delivery_days": "1.4",
            "min_delivery_days": "1.3",
            "max_delivery_days": "1.4",
            "same_day_delivery_pct": "0.0",
            "within_2days_pct": "100.0",
            "origin": "ICN",
            "destination": "KRPUS"
        },
        {
            "customer": "6549",
            "forwarder": null,
            "total_deliveries": "292",
            "avg_delivery_days": "9.5",
            "min_delivery_days": "1.3",
            "max_delivery_days": "39.7",
            "same_day_delivery_pct": "0.0",
            "within_2days_pct": "71.6",
            "origin": null,
            "destination": "KRPUS"
        },
        {
            "customer": "6549",
            "forwarder": null,
            "total_deliveries": "69",
            "avg_delivery_days": "36.5",
            "min_delivery_days": "12.9",
            "max_delivery_days": "38.0",
            "same_day_delivery_pct": "0.0",
            "within_2days_pct": "0.0",
            "origin": null,
            "destination": "KRYOS"
        },
        {
            "customer": "6549",
            "forwarder": "Fuji Global",
            "total_deliveries": "37",
            "avg_delivery_days": "10.7",
            "min_delivery_days": "0.0",
            "max_delivery_days": "47.2",
            "same_day_delivery_pct": "18.9",
            "within_2days_pct": "21.6",
            "origin": null,
            "destination": "KRPUS"
        },
        {
            "customer": "60CL",
            "forwarder": "Fuji Global",
            "total_deliveries": "20",
            "avg_delivery_days": "13.1",
            "min_delivery_days": "0.0",
            "max_delivery_days": "50.3",
            "same_day_delivery_pct": "40.0",
            "within_2days_pct": "45.0",
            "origin": null,
            "destination": "KRBNP"
        },
        {
            "customer": "60YC",
            "forwarder": null,
            "total_deliveries": "19",
            "avg_delivery_days": "-10.2",
            "min_delivery_days": "-10.4",
            "max_delivery_days": "-5.6",
            "same_day_delivery_pct": "100.0",
            "within_2days_pct": "100.0",
            "origin": "JPUKB",
            "destination": "JPKII"
        },
        ...]
}

![image](https://github.com/user-attachments/assets/cd8fe34a-ad76-40a6-b063-23c7a21f8413)



//Monthly average duration of storage in days at each hub
Applies to POs that have departed from the hub within the month.
Departure date is calculated as the difference between arrival date and departure from the hub.
