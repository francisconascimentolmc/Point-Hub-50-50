# Queries Legacy — Pendentes de Validação/Publicação

---

## Solicitar Bobinas — Legacy
> **Status:** Calculado, não publicado no painel
> **Resultado:** 39% (MLA: 33,6% | MLB: 40,9%)
> **Motivo:** Diferença metodológica vs Hub — Legacy mede a partir da entrada no flow (`/point_cx_paper_roll/flow`), que já é um passo mais avançado na jornada do que o clique no shortcut do Hub. Comparação direta não é precisa sem ajuste.

```sql
WITH
clicks AS (
  SELECT
    CAST(JSON_VALUE(event_data, '$.customer') AS STRING) AS user_id,
    site,
    ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS`
  WHERE ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND bu = 'mercadopago'
    AND path = '/point_cx_paper_roll/flow'
    AND site IN ('MLA', 'MLB')
    AND JSON_VALUE(event_data, '$.customer') IS NOT NULL
    AND CAST(JSON_VALUE(event_data, '$.customer') AS INT64)
        NOT IN (3087698992, 3222801111)
),
pedidos AS (
  SELECT
    CAST(SO_CUS_CUST_ID AS STRING) AS user_id,
    SO_SITE_ID                     AS site,
    DATE(SO_REQUEST_DATE)          AS data_pedido
  FROM `meli-bi-data.WHOWNER.BT_MP_POINT_SERVICE_ORDER`
  WHERE DATE(SO_REQUEST_DATE) BETWEEN '2026-02-28' AND '2026-03-14'
    AND SO_FLOW_TYPE = 'PAPER_ROLL'
    AND SO_SITE_ID IN ('MLA', 'MLB')
),
funil AS (
  SELECT
    c.site, c.data, c.user_id,
    CASE WHEN p.user_id IS NOT NULL THEN 1 ELSE 0 END AS pediu
  FROM clicks c
  LEFT JOIN pedidos p
    ON  c.user_id = p.user_id
    AND c.data    = p.data_pedido
    AND c.site    = p.site
)
SELECT
  site, data,
  COUNT(DISTINCT user_id)                                       AS usuarios_clicaram,
  COUNT(DISTINCT CASE WHEN pediu = 1 THEN user_id END)          AS usuarios_pediram,
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN pediu = 1 THEN user_id END),
    COUNT(DISTINCT user_id)
  ) * 100, 2)                                                   AS taxa_conversao_pct
FROM funil
GROUP BY 1, 2
ORDER BY 2, 1
```

**Campos:**
- `clicks.user_id` = `JSON_VALUE(event_data, '$.customer')` — confirmado via exploração (usr.user_id vem NULL na maioria dos eventos desse path)
- `pedidos` = `BT_MP_POINT_SERVICE_ORDER` com `SO_FLOW_TYPE = 'PAPER_ROLL'`
- JOIN: mesmo dia (`c.data = p.data_pedido`) + mesmo site

**Próximo passo sugerido:** encontrar o equivalente do path de shortcut no Legacy para que a comparação seja feita no mesmo ponto da jornada.

---

## Comprar Point — Legacy
> **Status:** Calculado e publicado no painel com flag ⚠️
> **Resultado:** 13,5% (MLA: 17,5% | MLB: 13,1%)
> **Motivo flag:** Hub usa shortcut `/point_devices/point_hub/shortcuts` + `action=buy_point`. Legacy usa `/point_devices/buy_new_point` (página de intenção de compra). Pontos de entrada diferentes na jornada — metodologia será revisada para alinhar.

```sql
WITH
clicks AS (
  SELECT
    CAST(usr.user_id AS STRING) AS user_id,
    site,
    ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS`
  WHERE ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND bu = 'mercadopago'
    AND path = '/point_devices/buy_new_point'
    AND site IN ('MLA', 'MLB')
    AND usr.user_id IS NOT NULL
    AND SAFE_CAST(usr.user_id AS INT64) NOT IN (3087698992, 3222801111)
),
purchases AS (
  SELECT
    CAST(CUS_CUST_ID_BUY AS STRING) AS user_id,
    SIT_SITE_ID,
    DATE(PO_CREATED_AT)             AS data,
    ORDER_DESCRIPTION
  FROM `meli-bi-data.WHOWNER.BT_PRODUCT_ORDERS`
  WHERE OPERATION_TYPE_DESC = 'PURCHASE'
    AND SIT_SITE_ID IN ('MLA', 'MLB')
    AND PO_CREATED_AT BETWEEN '2026-02-28' AND '2026-03-14'
    AND LOWER(ORDER_DESCRIPTION) LIKE '%point%'
),
funil AS (
  SELECT
    c.site, c.data, c.user_id,
    CASE WHEN p.user_id IS NOT NULL THEN 1 ELSE 0 END AS comprou
  FROM clicks c
  LEFT JOIN purchases p
    ON  c.user_id = p.user_id
    AND c.data    = p.data
    AND c.site    = p.SIT_SITE_ID
)
SELECT
  site, data,
  COUNT(DISTINCT user_id)                                        AS usuarios_clicaram,
  COUNT(DISTINCT CASE WHEN comprou = 1 THEN user_id END)         AS usuarios_compraram,
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN comprou = 1 THEN user_id END),
    COUNT(DISTINCT user_id)
  ) * 100, 2)                                                    AS taxa_conversao_pct
FROM funil
GROUP BY 1, 2
ORDER BY 2, 1
```

**Campos:**
- `clicks.user_id` = `usr.user_id` (confirmado — campo preenchido para esse path)
- `purchases` = `BT_PRODUCT_ORDERS` com `LOWER(ORDER_DESCRIPTION) LIKE '%point%'`
- JOIN: mesmo dia + mesmo site
- 14/03 excluído da média (lag de ingestão = 0 compras)

---

## Ativar Point — Legacy
> **Status:** Calculado, NÃO publicado — problema metodológico identificado
> **Resultado:** ~3,4% (MLA: ~0,9% com muitos zeros | MLB: ~3,4%)
> **Motivo não publicar:** Mesmo desalinhamento de funil que Comprar Point e Solicitar Bobinas.
> - **Hub** mede a partir do clique no shortcut `vinculate_point` (`/point_devices/point_hub/shortcuts`) — entrada no funil, usuário pode ser apenas curioso
> - **Legacy** mede a partir do acesso à página `/point_devices/activate_point` — usuário já navegou ativamente até a tela de ativação, mais engajado/qualificado
> - Comparação direta infla o Legacy artificialmente (começa mais tarde no funil)

```sql
WITH
clicks AS (
  SELECT
    CAST(usr.user_id AS STRING) AS user_id,
    site,
    ds AS data
  FROM `meli-bi-data.MELIDATA.TRACKS`
  WHERE ds BETWEEN '2026-02-28' AND '2026-03-14'
    AND bu = 'mercadopago'
    AND path = '/point_devices/activate_point'
    AND site IN ('MLA', 'MLB')
    AND usr.user_id IS NOT NULL
    AND CAST(usr.user_id AS INT64) NOT IN (3087698992, 3222801111)
),
ativacoes AS (
  SELECT LAST_USER_ID AS user_id, DATE(CREATED_AT) AS data_ativacao, SIT_SITE_ID AS site
  FROM `meli-bi-data.WHOWNER.LK_PTM_POINT_DEVICE`
  WHERE DATE(CREATED_AT) BETWEEN '2026-02-28' AND '2026-03-14'
    AND SIT_SITE_ID IN ('MLA', 'MLB')
    AND LAST_USER_ID IS NOT NULL
  GROUP BY 1, 2, 3
),
funil AS (
  SELECT c.site, c.data, c.user_id,
    CASE WHEN a.user_id IS NOT NULL THEN 1 ELSE 0 END AS ativou
  FROM clicks c
  LEFT JOIN ativacoes a ON c.user_id=a.user_id AND c.data=a.data_ativacao AND c.site=a.site
)
SELECT site, data,
  COUNT(DISTINCT user_id)                                        AS usuarios_clicaram,
  COUNT(DISTINCT CASE WHEN ativou=1 THEN user_id END)            AS usuarios_ativaram,
  ROUND(SAFE_DIVIDE(COUNT(DISTINCT CASE WHEN ativou=1 THEN user_id END),
    COUNT(DISTINCT user_id))*100,2)                             AS taxa_conversao_pct
FROM funil GROUP BY 1,2 ORDER BY 2,1
```

**Próximo passo:** encontrar o equivalente do shortcut de ativação no Legacy — um evento que capture o mesmo ponto da jornada que `vinculate_point` captura no Hub.
