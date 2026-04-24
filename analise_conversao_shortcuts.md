# Análise de Taxa de Conversão por Shortcut — Point Hub vs Legacy
> Período: 28/02/2026 – 14/03/2026 | Sites: MLA, MLB | BU: mercadopago

---

## Resultado Consolidado (estado atual do painel)

| Shortcut | Point Hub | Legacy | Metodologia alinhada? | Nota |
|----------|:---------:|:------:|:---------------------:|------|
| 🛒 Comprar Point | **11,6%** | **13,5% ⚠️** | Não | Legacy usa `/buy_new_point` (mais avançado no funil) |
| ⚡ Ativar Point | **2,7%** | **3,4% ⚠️** | Parcial | Mesma conversão (LK_PTM), entry point diferente |
| 🎟️ Vouchers | **49,1%** `MLB` | **46,2%** `MLB` | ✅ Sim | Mesmo evento de conversão (`/point_vouchers/detail`) |
| 📦 Solicitar Bobinas | **29%** | **39% ⚠️** | Não | Legacy usa flow page (mais avançado no funil) |

> ⚠️ = metodologia diferente (entry point mais avançado no funil para Legacy — comparação deve ser revisada)
> 💡 Vouchers mede **intenção pós-entrada na seção** (engajamento), não conversão transacional

---

## 1. Comprar Point

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_devices/point_hub/shortcuts` + `action=buy_point` | `/point_devices/buy_new_point` |
| **Conversão** | `BT_PRODUCT_ORDERS` — `OPERATION_TYPE_DESC='PURCHASE'` + `LOWER(ORDER_DESCRIPTION) LIKE '%point%'` | Mesmo |
| **Taxa** | **11,6%** (MLA: 15,1% / MLB: 11,3%) | **13,5%** ⚠️ (MLA: 17,5% / MLB: 13,1%) |
| **User ID** | `usr.user_id` | `usr.user_id` |

---

## 2. Ativar Point

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_devices/point_hub/shortcuts` + `action=vinculate_point` | `/point_devices/activate_point` |
| **Conversão** | `LK_PTM_POINT_DEVICE` — `CREATED_AT` mesmo dia | Mesmo |
| **Taxa** | **2,7%** (MLA: ~1,2% / MLB: ~2,8%) | **3,4%** ⚠️ (MLA: ~0,8% / MLB: ~3,6%) |
| **User ID** | `usr.user_id` | `usr.user_id` |
| **Caveat MLA** | LK_PTM_POINT_DEVICE cobertura parcial para Argentina (muitos dias zeros) |

---

## 3. Vouchers

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_vouchers/main` | `/point_devices/vouchers` |
| **Conversão** | `/point_vouchers/detail` (acesso ao detalhe do voucher) | `/point_vouchers/detail` (mesmo) |
| **Taxa** | **49,1%** `MLB` | **46,2%** `MLB` |
| **User ID** | `usr.user_id` | `usr.user_id` |
| **Nota** | MLB only (MLA sem tráfego de Vouchers) | MLB only |
| **Metodologia** | Mede **intenção** (engajamento dentro da seção), não conversão transacional |

> ✅ **Comparação mais alinhada** — mesmo evento de conversão nos dois ambientes. Hub +2,9 p.p.

---

## 4. Solicitar Bobinas

| | Hub | Legacy |
|-|-----|--------|
| **Path clicks** | `/point_devices/point_hub/shortcuts` + `action=manage_paper_rolls` | `/point_cx_paper_roll/flow` |
| **Conversão** | `BT_MP_POINT_SERVICE_ORDER` — `SO_FLOW_TYPE='PAPER_ROLL'` mesmo dia | Mesmo |
| **Taxa** | **29%** (MLA: 31,8% / MLB: 28,6%) | **39%** ⚠️ (MLA: 33,6% / MLB: 40,9%) |
| **User ID Hub** | `usr.user_id` | `JSON_VALUE(event_data, '$.customer')` (confirmado) |

---

## Filtros Padrão (todos os shortcuts)

```sql
ds BETWEEN '2026-02-28' AND '2026-03-14'
site IN ('MLA', 'MLB')
bu = 'mercadopago'
-- Excluir usuários de teste:
SAFE_CAST(usr.user_id AS INT64) NOT IN (3087698992, 3222801111)
```

## Status das Queries Legacy

| Shortcut | Publicado | Arquivo |
|----------|:---------:|---------|
| Comprar Point | ✅ com ⚠️ | `queries_legacy_pendentes.md` |
| Ativar Point | ✅ com ⚠️ | `queries_legacy_pendentes.md` |
| Vouchers | ✅ alinhado | — |
| Solicitar Bobinas | ❌ | `queries_legacy_pendentes.md` |
