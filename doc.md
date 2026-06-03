# Sistema de Games - Arquitetura e Fluxos

## 📁 Estrutura de Arquivos

```
games/
├── handlers.py          # Comandos do usuário (/steam, /ps, /games)
├── search_orchestrator.py  # Orquestra busca + cache SQL
├── services.py          # SteamService (busca ITAD/Steam + CRUD monitores)
├── price_aggregator.py  # Mescla preços PC/PS de múltiplas fontes
├── formatter.py         # Renderiza mensagens (busca, lista, notificações)
├── cache.py             # Cache em memória (10min) dos resultados de busca
├── games_cache_service.py  # Cache SQL (24h) de jogos já buscados
├── repository.py        # CRUD monitores + games (PostgreSQL)
├── preferences.py       # Preferência de loja (steam_only, steam_plus_keys, all)
├── matching.py          # Fuzzy matching de nomes de jogos
├── platforms.py         # Enum Platform + parsing aliases
├── platform_config.py   # Config de plataformas habilitadas + limites
├── schemas.py           # Dataclasses (SearchResult, Deal, etc)
├── models.py            # SQLAlchemy models (SteamMonitor, Game)
├── franchises.py        # Expansão de siglas (gta → Grand Theft Auto)
├── images.py            # Download de covers (RAWG filtrado, PS Store)
│
├── loops/
│   ├── steam_loop.py    # Loop de monitoramento PC (30min)
│   └── ps_loop.py       # Loop de monitoramento PS (30min)
│
├── pricing/
│   ├── cooldown.py      # Política de notificação (3d, 15d)
│   ├── transitions.py   # Lógica de transição de estado (preços, promos)
│   └── throttle.py      # Rate limiting de API requests
│
└── providers/
    ├── itad.py          # ITAD API v1 (search) + v2 (appids) + v3 (prices)
    ├── steam_client.py  # Steam Store API (search, appdetails, packagedetails)
    ├── psstore_browser.py   # PS Store scraping (Playwright)
    └── psprices_browser.py  # PSPrices scraping (Playwright)
```

---

## 🔍 Fluxo 1: Busca de Jogos (PC)

```
┌─────────────────────────────────────────────────────────────────┐
│ Usuário: /steam cyberpunk                                       │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ handlers.py → handle_steam_command()                           │
│ ├─ Valida termo (min 2 chars)                                 │
│ ├─ Verifica plataforma habilitada                             │
│ └─ Delega para search_orchestrator.search()                   │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ search_orchestrator.py → search()                             │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 1. Verifica CACHE SQL (games table)             │          │
│ │    ├─ HIT (≥3 jogos, fresh <24h)?               │          │
│ │    │   └─ Enriquece com ITAD promos → RETORNA   │          │
│ │    └─ MISS: continua busca externa              │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 2. Busca EXTERNA (services.search())            │          │
│ │    ├─ ITAD v1 search (UUIDs)                    │          │
│ │    ├─ ITAD v3 prices (deals por UUID)           │          │
│ │    ├─ ITAD v2 appids (UUID → Steam appid)       │          │
│ │    └─ Mescla: ITAD >= 10? só ITAD : ITAD+Steam  │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 3. Persiste no CACHE SQL (games table)          │          │
│ │    └─ Próxima busca: cache hit (24h)            │          │
│ └─────────────────────────────────────────────────┘          │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ formatter.py → render_search_results()                         │
│ └─ Formata lista numerada com preços + ícones                 │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ Cache em MEMÓRIA (10min)                                       │
│ └─ Guarda resultados para callbacks ("1", "2", "3")           │
└────────────────────────────────────────────────────────────────┘
```

**Decisão ITAD vs Steam:**
```
IF ITAD retorna >= 10 jogos:
    └─ Retorna apenas ITAD (enriched com múltiplas lojas)

ELSE IF ITAD retorna 1-9 jogos:
    ├─ Busca Steam Store
    └─ Mescla inteligente:
       ├─ Matched (aparecem em ambas) → ITAD enriched
       ├─ ITAD-only → ITAD enriched
       └─ Steam-only → Steam (até completar 10)

ELSE (ITAD 0 jogos):
    └─ Fallback: apenas Steam Store
```

---

## 🔍 Fluxo 2: Busca de Jogos (PlayStation)

```
┌─────────────────────────────────────────────────────────────────┐
│ Usuário: /ps god of war                                         │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ handlers.py → _handle_ps_search()                              │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ search_orchestrator.py → _handle_ps_search()                   │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 1. Verifica CACHE SQL (games table)             │          │
│ │    └─ HIT? Retorna (enriquece com PSN ID)       │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 2. Busca EXTERNA (price_aggregator.aggregate_ps) │          │
│ │    ├─ Primary: PS Store Browser (Playwright)     │          │
│ │    └─ Fallback: PSPrices Browser (timeout/falha) │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ └─ Persiste no cache SQL (platform_id = PSN ID)              │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ formatter.py → render_ps_search_results()                      │
└────────────────────────────────────────────────────────────────┘
```

---

## 📊 Fluxo 3: Adicionar Monitor

```
┌─────────────────────────────────────────────────────────────────┐
│ Usuário responde: 1                                             │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ handlers.py → complete_selection()                             │
│ ├─ Recupera resultados do CACHE MEMÓRIA (10min)               │
│ ├─ Valida seleção (1-N)                                       │
│ └─ Verifica limite de monitores (role-based)                  │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ Resolve ITAD UUID → Steam appid (se necessário)               │
│                                                                │
│ IF steam_id contém "-" (UUID) e NÃO é PSN_ID:                 │
│   ├─ Tenta resolver via ITAD v2 (UUID → appid)                │
│   │                                                            │
│   ├─ Resolveu appid?                                          │
│   │   └─ ✅ Usa appid (jogo normal na Steam)                  │
│   │                                                            │
│   └─ NÃO resolveu appid?                                      │
│       ├─ Tem preços? (key stores, non-Steam)                  │
│       │   └─ ✅ Permite monitorar com UUID                    │
│       │       (bundles que só vendem por key)                 │
│       │                                                        │
│       └─ Sem preços?                                          │
│           └─ ❌ Bloqueia: "não disponível na Steam"           │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ services.py → add_monitor()                                    │
│ └─ Insere em steam_monitors:                                  │
│    ├─ search_id (Steam appid, ITAD UUID, ou PSN ID)           │
│    ├─ name, user_id, from_app                                 │
│    └─ Estado inicial: was_in_promo=0, unavailable=0           │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ games table (cache SQL)                                        │
│ └─ Dados de preço/tipo já foram salvos na busca               │
└────────────────────────────────────────────────────────────────┘
```

**Tabelas:**
- `steam_monitors`: Estado de notificação **por usuário**
- `games`: Dados do jogo (preços, tipo, plataforma) **compartilhados**

**Tipos de ID aceitos no `search_id`:**
- Steam appid numérico (ex: `1091500`)
- ITAD UUID (ex: `018dcc3c-39cd-...`) — bundles/packages sem Steam appid
- PSN ID (ex: `UP9000-CUSA00552_00-...`)

---

## 🔄 Fluxo 4: Loop de Monitoramento (PC)

```
┌────────────────────────────────────────────────────────────────┐
│ Loop (a cada 30 min): steam_loop.py                           │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ 1. Busca monitores devido (last_check_at < agora - 30min)     │
│    ├─ Filtra: unavailable=0                                   │
│    └─ Ordena: last_check_at ASC (mais antigos primeiro)       │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ Para cada monitor:                                             │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 2. Verifica COOLDOWN (3 dias desde última notif) │          │
│ │    └─ Dentro do cooldown? SKIP (não busca preço) │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 3. Busca preço atual                             │          │
│ │    ├─ Via ITAD (lookup + prices)                 │          │
│ │    └─ Fallback: Steam Store API                  │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 4. Classifica transição (transitions.py)         │          │
│ │    ├─ Sucesso: classify_observation()            │          │
│ │    └─ Falha: classify_failure()                  │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ ┌─────────────────────────────────────────────────┐          │
│ │ 5. Decide notificação (cooldown.py)              │          │
│ │    ├─ Primeira vez: abaixo do padrão?            │          │
│ │    ├─ >= 3 dias E < último notificado?           │          │
│ │    └─ >= 15 dias E < preço padrão?               │          │
│ └─────────────────────────────────────────────────┘          │
│                                                                │
│ └─ IF should_notify: Envia notificação + atualiza monitor    │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│ 6. Atualiza games table (upsert)                              │
│    ├─ price_store, min_price_store (Steam Store)              │
│    ├─ min_price_store_key, store_key (key stores)             │
│    └─ min_price_store_nosteam, store_nosteam (Epic, GOG)      │
└────────────────────────────────────────────────────────────────┘
```

**Política de notificação:**
```
IF nunca notificou:
    └─ Notifica SE preço < padrão

ELSE IF >= 3 dias desde última notificação:
    └─ Notifica SE preço < último notificado

ELSE IF >= 15 dias desde última notificação:
    └─ Notifica SE preço < padrão

ELSE:
    └─ Não notifica (dentro do cooldown)
```

---

## 🗄️ Sistema de Cache (3 Camadas)

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. CACHE MEMÓRIA (10 min)                                      │
│    Arquivo: cache.py                                            │
│    ├─ Chave: (user_id, from_app)                               │
│    ├─ Valor: list[SearchResult]                                │
│    ├─ TTL: 10 minutos                                           │
│    └─ Uso: Callbacks de seleção ("1", "2", "3")                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 2. CACHE SQL (24 horas)                                        │
│    Arquivo: games_cache_service.py                             │
│    ├─ Tabela: games                                            │
│    ├─ Chave: (name ILIKE %term%, platform)                     │
│    ├─ Freshness: updated_at < 24h                              │
│    └─ Uso: Evitar requests ITAD/Steam repetidos                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 3. CACHE PERSISTENTE (games table)                             │
│    Arquivo: repository.py → GamesRepository                    │
│    ├─ Tabela: games                                            │
│    ├─ Sem TTL (persiste indefinidamente)                       │
│    ├─ Atualiza: busca do usuário + loop de monitoramento       │
│    └─ Uso: Baseline de preços (normal, lowest, key, nosteam)   │
└─────────────────────────────────────────────────────────────────┘
```

**Relação entre tabelas:**
```
steam_monitors (por usuário)
├─ search_id (FK → games.steam_id ou games.platform_id)
└─ Estado de notificação (last_promo_notified_at, was_in_promo)

games (compartilhado)
├─ steam_id (Steam appid) ou platform_id (UUID/PSN ID)
├─ Preços baseline (price_store, min_price_store_key, min_price_store_nosteam)
└─ Metadados (name, game_type, platform)
```

---

## 🔑 Conceitos Chave

### Preferências de Loja (PC)
```
STEAM_ONLY        → Só Steam Store
STEAM_PLUS_KEYS   → Steam Store + key stores (Nuuvem, GMG) [PADRÃO]
ALL               → Steam + key stores + non-Steam (Epic, GOG)
```

### Classificação de Lojas (3 buckets)
```
💙 Steam Store       → price_store, min_price_store
🔑 Key Stores        → min_price_store_key, store_key (ativação Steam)
☁️ Non-Steam Stores  → min_price_store_nosteam, store_nosteam (Epic, GOG)
```

### Matching de Jogos
- **Por ID**: `steam_id` exato (mais confiável)
- **Por nome**: Fuzzy matching (similarity > 0.85)
- **Deduplicação**: ITAD UUID → resolve Steam appid via ITAD v2

### Sistema de Imagens (images.py)
**Fontes de capa (ordem de tentativa):**
1. Cache local (busca por `steam_id` ou nome normalizado)
2. `package_image_url` (se fornecida)
3. ITAD CDN (banner 400/300/600)
4. Steam CDN (múltiplos formatos)
5. Resolve appid via ITAD API + Steam CDN
6. PS Store image (type=12 do chihiro)
7. **RAWG (background_image)** — fallback universal

**Filtro de plataforma RAWG:**
- **PlayStation (PS Store/PSN)**: Aplica filtro `platforms=[18, 187, 16]` (PS4, PS5, PS3)
- **PC/Steam**: Sem filtro (busca genérica)
- **Detecção automática**: Via PSN_ID, imagens PS Store, ou contexto de busca

### Transições de Estado (Monitor)
```
normal_price       → Preço padrão (histerese de 2 checks)
lowest_price       → Menor preço já visto (só desce, nunca sobe)
was_in_promo       → Flag: jogo está em promoção (1) ou não (0)
consecutive_failures → Contador de falhas (>=3 → unavailable=1)
```

---

## 🎯 Fluxo Resumido End-to-End

```
1. Usuário: /steam cyberpunk
2. Cache SQL hit? Retorna + enriquece ITAD
3. Cache miss: ITAD search + prices + Steam (se <10)
4. Salva no cache SQL (games table)
5. Renderiza lista numerada
6. Usuário: 1
7. Valida + cria monitor (steam_monitors table)
8. Loop (30min): verifica preços
9. Cooldown ok? Busca ITAD/Steam
10. Classifica transição + política de notificação
11. Notifica? Envia mensagem + atualiza monitor
12. Atualiza games table (baseline de preços)
```

---

**Última atualização:** 2026-06-03
