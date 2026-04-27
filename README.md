# Point Hub — Base de Conhecimento

Diretório para centralizar contexto, análises e documentos do projeto Point Hub.
Tudo que for discutido sobre o tema é salvo aqui para manter contexto entre sessões.

## Arquivos

| Arquivo | Conteúdo |
|---------|----------|
| [painel_50_50.md](painel_50_50.md) | Análise do experimento 50/50 (28/02–14/03/2026): engajamento, audiência, fontes de tráfego, histórico mensal |
| [analise_conversao_shortcuts.md](analise_conversao_shortcuts.md) | Taxa de conversão Hub vs Legacy com metodologia de interseção: Comprar 12,1% / Ativar 2,7% / Vouchers 49,1% / Bobinas 37,2% — queries completas, paths testados, interseções adotadas |
| [analise_devolucoes_compras.md](analise_devolucoes_compras.md) | Efetividade de alertas (Devolução Pendente, Same Day, Device), funnels de conclusão Hub vs Legacy — schemas e queries |
| [queries_legacy_pendentes.md](queries_legacy_pendentes.md) | Queries Legacy calculadas mas não publicadas ou com flag ⚠️: Solicitar Bobinas (39%), Comprar Point (13,5%), Ativar Point (3,4%) |

## Resumo do Projeto

**Objetivo:** Reduzir Contact Rate (CX) e custos melhorando a seção "Minhas Maquininhas" no app ML para sellers de Point.

**Status atual:** Rollout do Point Hub em andamento (MLA + MLB), experimento 50/50 encerrado em 14/03/2026.

**Problema central:** Apesar do ganho de audiência (+6%), o Contact Rate não reduziu. Point Hub e Legacy apresentam CR quase idêntico.

## Constantes de Query

```sql
-- Período
ds >= '2026-01-29'

-- Sites
site_id IN ('MLA', 'MLB')

-- BU
bu = 'mercadopago'                          -- MELIDATA.TRACKS
PROCESS_BU_CR_REPORTING = 'MP'              -- BT_CX_HELP_RATE_V2

-- Paths
'/point_devices/point_hub'  -- Point Hub
'/point_devices/kinds'      -- Legacy

-- Excluir usuários de teste
CUS_CUST_ID NOT IN (3087698992, 3222801111)
```
