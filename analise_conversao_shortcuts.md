# Análise de Taxa de Conversão por Shortcut — Point Hub vs Legacy
> Período: 28/02/2026 – 14/03/2026 | Sites: MLA, MLB | BU: mercadopago
> Última atualização: 27/04/2026

---

## Resultado Final no Painel

| Shortcut | Point Hub | Legacy | Alinhado? | Δ |
|----------|:---------:|:------:|:---------:|:-:|
| 🛒 Comprar Point | **12,1%** | **13,5% ⚠️** | Parcial | -1,4 p.p. |
| ⚡ Ativar Point | **2,7%** | **3,4% ⚠️** | Parcial | -0,7 p.p. |
| 🎟️ Vouchers | **49,1%** `MLB` | **46,2%** `MLB` | ✅ Sim | +2,9 p.p. |
| 📦 Solicitar Bobinas | **37,2%** | **39% ⚠️** | Parcial | -1,8 p.p. |

> **Conclusão:** Empate técnico — variações mínimas que não permitem apontar avanço ou retrocesso definitivo.

---

## Metodologia de Interseção (ajuste de alinhamento de funil)

Para reduzir a diferença de nível de funil entre Hub e Legacy, foram aplicadas interseções de path. Hub garante que o usuário passou por um step específico **E** clicou no shortcut no mesmo dia.

### Estrutura base da interseção

```sql
WITH clicks AS (
  SELECT DISTINCT
    CAST(t1.usr.user_id AS STRING) AS user_id,
    t1.site, t1.ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS` t1
  INNER JOIN `meli-bi-data.MELIDATA.TRACKS` t2
    ON  CAST(t1.usr.user_id AS STRING) = CAST(t2.usr.user_id AS STRING)
    AND t1.ds = t2.ds AND t1.site = t2.site
  WHERE t1.ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND t1.bu = 'mercadopago'
    AND t1.path = '/point_devices/point_hub/shortcuts'
    AND JSON_VALUE(t1.event_data, '$.action') = '[ACTION]'   -- shortcut específico
    AND t1.site IN ('MLA','MLB')
    AND t1.usr.user_id IS NOT NULL
    AND SAFE_CAST(t1.usr.user_id AS INT64) NOT IN (3087698992,3222801111)
    AND t2.path = '[PATH_INTERSECAO]'                        -- path de alinhamento
    AND t2.bu   = 'mercadopago'
    AND t2.usr.user_id IS NOT NULL
)
```

---

## 1. Comprar Point

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_devices/point_hub/shortcuts` + `action=buy_point` | `/point_devices/buy_new_point` |
| **Path interseção (Hub)** | `/calm/landing/point` | — |
| **Conversão** | `BT_PRODUCT_ORDERS` — `PURCHASE` + `LOWER(ORDER_DESCRIPTION) LIKE '%point%'` | Mesmo |
| **Taxa** | **12,1%** (antes: 11,6% sem interseção) | **13,5% ⚠️** |
| **User ID** | `usr.user_id` | `usr.user_id` |

**Impacto da interseção:** 16.435 → 15.753 cliques (-682). Taxa subiu 11,6% → 12,1% — removidos usuários que clicaram o shortcut sem ter passado pela landing de compra.

**Query Hub (com interseção):**
```sql
WITH clicks AS (
  SELECT DISTINCT CAST(t1.usr.user_id AS STRING) AS user_id, t1.site, t1.ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS` t1
  INNER JOIN `meli-bi-data.MELIDATA.TRACKS` t2
    ON CAST(t1.usr.user_id AS STRING)=CAST(t2.usr.user_id AS STRING)
    AND t1.ds=t2.ds AND t1.site=t2.site
  WHERE t1.ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND t1.bu='mercadopago'
    AND t1.path='/point_devices/point_hub/shortcuts'
    AND JSON_VALUE(t1.event_data,'$.action')='buy_point'
    AND t1.site IN ('MLA','MLB') AND t1.usr.user_id IS NOT NULL
    AND SAFE_CAST(t1.usr.user_id AS INT64) NOT IN (3087698992,3222801111)
    AND t2.path='/calm/landing/point' AND t2.bu='mercadopago'
    AND t2.usr.user_id IS NOT NULL
),
purchases AS (
  SELECT CAST(CUS_CUST_ID_BUY AS STRING) AS user_id, SIT_SITE_ID, DATE(PO_CREATED_AT) AS data
  FROM `meli-bi-data.WHOWNER.BT_PRODUCT_ORDERS`
  WHERE OPERATION_TYPE_DESC='PURCHASE' AND SIT_SITE_ID IN ('MLA','MLB')
    AND PO_CREATED_AT BETWEEN '2026-02-28' AND '2026-03-14'
    AND LOWER(ORDER_DESCRIPTION) LIKE '%point%'
)
SELECT c.site, c.data,
  COUNT(DISTINCT c.user_id) AS clicaram,
  COUNT(DISTINCT p.user_id) AS compraram,
  ROUND(COUNT(DISTINCT p.user_id)*100.0/NULLIF(COUNT(DISTINCT c.user_id),0),2) AS taxa_pct
FROM clicks c LEFT JOIN purchases p
  ON c.user_id=p.user_id AND c.data=p.data AND c.site=p.SIT_SITE_ID
GROUP BY 1,2 ORDER BY 2,1
```

---

## 2. Ativar Point

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_devices/point_hub/shortcuts` + `action=vinculate_point` | `/point_devices/activate_point` |
| **Path interseção testada** | `/instore/scan_qr` (não alterou resultado — descartada) | — |
| **Conversão** | `LK_PTM_POINT_DEVICE` — `CREATED_AT` mesmo dia | Mesmo |
| **Taxa Hub (shortcut only)** | **2,7%** | **3,4% ⚠️** (MLB: 3,6%) |
| **Obs. MLA** | LK_PTM_POINT_DEVICE cobertura parcial — muitos zeros em MLA |
| **Obs. MLB only** | Hub 2,8% vs Legacy 3,6% — gap real de ~0,8 p.p. |

**Teste de interseção `/instore/scan_qr`:** 30.190 → 29.848 cliques (-342). Taxa 2,7% → 2,7% — sem impacto. Quase todos que clicam o shortcut já passaram pelo scan QR no mesmo dia. Path não resolve o problema.

**Query Hub (shortcut only — publicado):**
```sql
WITH clicks AS (
  SELECT CAST(usr.user_id AS STRING) AS user_id, site, ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS`
  WHERE ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND bu='mercadopago'
    AND path='/point_devices/point_hub/shortcuts'
    AND JSON_VALUE(event_data,'$.action')='vinculate_point'
    AND site IN ('MLA','MLB') AND usr.user_id IS NOT NULL
    AND CAST(usr.user_id AS INT64) NOT IN (3087698992,3222801111)
),
ativacoes AS (
  SELECT LAST_USER_ID AS user_id, DATE(CREATED_AT) AS data_ativacao, SIT_SITE_ID AS site
  FROM `meli-bi-data.WHOWNER.LK_PTM_POINT_DEVICE`
  WHERE DATE(CREATED_AT) BETWEEN '2026-02-28' AND '2026-03-14'
    AND SIT_SITE_ID IN ('MLA','MLB') AND LAST_USER_ID IS NOT NULL
  GROUP BY 1,2,3
)
SELECT site, data,
  COUNT(DISTINCT user_id) AS clicaram,
  COUNT(DISTINCT CASE WHEN ativou=1 THEN user_id END) AS ativaram,
  ROUND(SAFE_DIVIDE(...)*100,2) AS taxa_pct
FROM (SELECT c.*, CASE WHEN a.user_id IS NOT NULL THEN 1 ELSE 0 END AS ativou
      FROM clicks c LEFT JOIN ativacoes a ON c.user_id=a.user_id AND c.data=a.data_ativacao AND c.site=a.site)
GROUP BY 1,2 ORDER BY 2,1
```

---

## 3. Vouchers

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_vouchers/main` | `/point_devices/vouchers` |
| **Conversão** | `/point_vouchers/detail` (acesso ao detalhe) | `/point_vouchers/detail` (mesmo ✅) |
| **Taxa** | **49,1%** `MLB` | **46,2%** `MLB` |
| **Obs.** | MLB only (MLA sem tráfego). Mede **intenção pós-entrada**, não conversão transacional |

> ✅ **Melhor alinhamento metodológico** — mesmo evento de conversão nos dois ambientes. Hub +2,9 p.p.

---

## 4. Solicitar Bobinas

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_devices/point_hub/shortcuts` + `action=manage_paper_rolls` | `/point_cx_paper_roll/flow` |
| **Path interseção (Hub)** | `/point_cx_paper_roll/inventory` | — |
| **Conversão** | `BT_MP_POINT_SERVICE_ORDER` — `SO_FLOW_TYPE='PAPER_ROLL'` mesmo dia | Mesmo |
| **Taxa** | **37,2%** (antes: 29% sem interseção) | **39% ⚠️** |
| **User ID Hub** | `usr.user_id` | `JSON_VALUE(event_data,'$.customer')` |

**Impacto da interseção `/inventory`:** 9.775 → 7.647 cliques (-2.128). Taxa subiu 29% → 37,2% — gap com Legacy reduziu de -10 p.p. → -1,8 p.p.

**Teste `/flow`:** 9.775 → 9.607 (-168). Taxa 29% → 29,6% — sem impacto relevante. Descartado.

**Query Hub (com interseção /inventory — publicado):**
```sql
WITH clicks AS (
  SELECT DISTINCT CAST(t1.usr.user_id AS STRING) AS user_id, t1.site, t1.ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS` t1
  INNER JOIN `meli-bi-data.MELIDATA.TRACKS` t2
    ON CAST(t1.usr.user_id AS STRING)=CAST(t2.usr.user_id AS STRING)
    AND t1.ds=t2.ds AND t1.site=t2.site
  WHERE t1.ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND t1.bu='mercadopago'
    AND t1.path='/point_devices/point_hub/shortcuts'
    AND JSON_VALUE(t1.event_data,'$.action')='manage_paper_rolls'
    AND t1.site IN ('MLA','MLB') AND t1.usr.user_id IS NOT NULL
    AND SAFE_CAST(t1.usr.user_id AS INT64) NOT IN (3087698992,3222801111)
    AND t2.path='/point_cx_paper_roll/inventory'
    AND t2.bu='mercadopago' AND t2.usr.user_id IS NOT NULL
),
pedidos AS (
  SELECT CAST(SO_CUS_CUST_ID AS STRING) AS user_id,
         SO_SITE_ID AS site, DATE(SO_REQUEST_DATE) AS data_pedido
  FROM `meli-bi-data.WHOWNER.BT_MP_POINT_SERVICE_ORDER`
  WHERE DATE(SO_REQUEST_DATE) BETWEEN '2026-02-28' AND '2026-03-14'
    AND SO_FLOW_TYPE='PAPER_ROLL' AND SO_SITE_ID IN ('MLA','MLB')
)
SELECT site, data,
  COUNT(DISTINCT user_id) AS clicaram,
  COUNT(DISTINCT CASE WHEN pediu=1 THEN user_id END) AS pediram,
  ROUND(SAFE_DIVIDE(...)*100,2) AS taxa_pct
FROM (SELECT c.*, CASE WHEN p.user_id IS NOT NULL THEN 1 ELSE 0 END AS pediu
      FROM clicks c LEFT JOIN pedidos p ON c.user_id=p.user_id AND c.data=p.data_pedido AND c.site=p.site)
GROUP BY 1,2 ORDER BY 2,1
```

---

## Paths de Interseção Testados

| Shortcut | Path testado | Impacto na taxa | Adotado? |
|----------|-------------|----------------|----------|
| Comprar Point | `/calm/landing/point` | 11,6% → **12,1%** (+0,5 p.p.) | ✅ Sim |
| Ativar Point | `/instore/scan_qr` | 2,7% → **2,7%** (sem impacto) | ❌ Não |
| Solicitar Bobinas | `/point_cx_paper_roll/inventory` | 29% → **37,2%** (+8,2 p.p.) | ✅ Sim |
| Solicitar Bobinas | `/point_cx_paper_roll/flow` | 29% → **29,6%** (sem impacto) | ❌ Não |

---

## Campos de Usuário por Tabela

| Tabela | Campo | Nota |
|--------|-------|------|
| `MELIDATA.TRACKS` (geral) | `usr.user_id` | STRING |
| `MELIDATA.TRACKS` (bobinas legacy) | `JSON_VALUE(event_data,'$.customer')` | Confirmado via exploração |
| `BT_PRODUCT_ORDERS` | `CUS_CUST_ID_BUY` | INTEGER |
| `LK_PTM_POINT_DEVICE` | `LAST_USER_ID` | STRING |
| `BT_MP_POINT_SERVICE_ORDER` | `SO_CUS_CUST_ID` | BIGNUMERIC |

## Filtros Padrão

```sql
ds BETWEEN '2026-02-28' AND '2026-03-14'
site IN ('MLA', 'MLB')
bu = 'mercadopago'
SAFE_CAST(usr.user_id AS INT64) NOT IN (3087698992, 3222801111)
```
