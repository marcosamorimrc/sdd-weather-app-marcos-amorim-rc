# Especificação de Produto — Weather App

**Data:** 2026-08-12  
**Versão:** 1.0 (MVP)  
**Status:** ✅ Aprovado para implementação  

---

## 1. Overview

### Propósito

Fornecer aos usuários finais acesso rápido e confiável a informações meteorológicas em tempo real e previsões de 5 dias, com interface responsiva otimizada para mobile, sem custo operacional.

### Escopo do MVP

- Busca de cidades por nome (geocoding)
- Exibição de clima atual (temperatura, condição, umidade, vento)
- Previsão diária para 5 dias
- Toggle de unidade de temperatura (°C / °F)
- Cache local até 24h (offline mode)
- Interface mobile-first em português-BR

### Fora do Escopo (Phase 2+)

- Autenticação e accounts
- Histórico de servidor
- Notificações/alertas
- Previsão horária (granularidade diária apenas no MVP)
- Múltiplas cidades monitoradas simultaneamente
- Suporte a idiomas (apenas pt-BR no MVP)

### Público-Alvo

1. **Marina** — Usuários casuais/apressados que verificam clima 1-3x/dia em <30s (mobile-first)
2. **João** — Profissionais especializados que acompanham clima 5-10x/dia para decisões críticas (Phase 2)
3. **Sofia** — Viajantes que consultam múltiplas cidades e precisam de offline (Phase 2+)

---

## 2. Functional Requirements

| ID | Requisito | Descrição | Prioridade |
|---|---|---|---|
| **RF1** | Buscar cidades | Usuário pesquisa localidade pelo nome (ex: "São Paulo", "Rio de Janeiro") e seleciona da lista de geocoding | 🔴 P0 — Crítico |
| **RF2** | Visualizar clima atual | App exibe temperatura, condição meteorológica (ensolarado, nublado, chuva, etc.), umidade, velocidade do vento, sensação térmica e pressão atmosférica | 🔴 P0 — Crítico |
| **RF3** | Previsão de 5 dias | Exibir clima previsto para os próximos 4 dias + hoje, com temperatura máxima, mínima, condição e precipitação por dia | 🔴 P0 — Crítico |
| **RF4** | Alternar unidade de temperatura | Permitir mudança entre Celsius (°C) e Fahrenheit (°F); preferência persistida em localStorage | 🟠 P1 — Alta |
| **RF5** | Cache offline de 24h | Ao perder conexão, app exibe última consulta válida por até 24h com indicador visual de "dados podem estar desatualizados" | 🟡 P2 — Média |

---

## 3. User Stories

### Sprint 0: Navegação & Busca (MVP Foundation)

#### 🎯 US1: Buscar localidade — RF1

**Como** Marina (usuária casual),  
**Quero** pesquisar uma cidade pelo nome e selecionar da lista de sugestões,  
**Para** encontrar rapidamente o local onde quero consultar o clima (em <30s).

**Critérios de Aceite (BDD — Given/When/Then):**

**AC1.1 — Campo de busca visível e acessível**
```gherkin
Given: A página carregou
When: O usuário visualiza a interface
Then: Um campo de texto com label "Buscar cidade" ou aria-label equivalente é visível
  And: O campo tem placeholder "Ex: São Paulo, Rio de Janeiro"
  And: O campo é acessível via Tab (tabindex ≥ 0)
```

**AC1.2 — Autocompletar retorna sugestões**
```gherkin
Given: O campo de busca está ativo
When: O usuário digita "São Paulo" (exatamente ≥2 caracteres)
Then: Em <500ms a partir do keystroke final, app exibe sugestões:
  And: Máximo 5 sugestões retornadas (count=5 na API)
  And: Cada sugestão mostra formato: "Nome da Cidade, País"
  And: Exemplos: "São Paulo, Brazil" | "São Paulo, United States"
  And: Sugestões vêm diretamente de GET /v1/search (Open-Meteo API)
  And: Lista está visível em dropdown abaixo do campo
  And: Sugestões são navegáveis com setas ↓/↑ (teclado)
  And: Sugestões são clicáveis (mouse) ou Enter (teclado)
  And: Timing medido com DevTools Network Timing (P95 ≤ 500ms)
  And: Debounce de 300ms entre keystrokes (não requer para cada letra)
```

**AC1.3 — Sem resultados**
```gherkin
Given: O campo de busca está ativo
When: O usuário digita "XYZZZZ999" (texto inexistente)
Then: Em <500ms, mensagem "Nenhuma cidade encontrada" é exibida
  And: Nenhuma requisição desnecessária à API é feita
  And: Campo permanece focado para nova tentativa
```

**AC1.4 — Seleção de sugestão**
```gherkin
Given: Lista de sugestões é exibida
When: O usuário clica ou pressiona Enter em uma sugestão
Then: A sugestão é selecionada
  And: Requisição GET para /weather?lat=X&lon=Y é iniciada
  And: Campo de busca é limpo
  And: Clima atual começa a carregar (<1s)
```

**AC1.5 — Erro de rede**
```gherkin
Given: Usuário está sem conexão ou API está indisponível
When: O usuário digita e aguarda resposta de geocoding
Then: Em >1s, mensagem "Erro de conexão. Tente novamente." é exibida
  And: Botão "Tentar Novamente" é disponibilizado
  And: Nenhuma requisição é retentada automaticamente (user controla)
```

---

#### 🎯 US2: Visualizar clima atual — RF2

**Como** Marina (usuária apressada),  
**Quero** ver a temperatura e condição do clima agora com destaque visual,  
**Para** decidir rapidamente como me vestir ou se levo guarda-chuva (sem distrações).

**Critérios de Aceite (BDD — Given/When/Then):**

**AC2.1 — Carregamento bem-sucedido de clima**
```gherkin
Given: Usuário selecionou uma cidade (ex: São Paulo, lat=-23.5, lon=-46.6)
When: App faz requisição GET /forecast?latitude=-23.5&longitude=-46.6 à API Open-Meteo
Then: Resposta é recebida em <1s
  And: Seção de "Clima Atual" é renderizada com os dados
  And: Não há mensagem de erro visível
```

**AC2.2 — Temperatura em destaque**
```gherkin
Given: Dados de clima atual foram recebidos
When: A página renderiza o clima
Then: Temperatura é exibida em tamanho grande:
  And: Mobile (320px):   font-size 48px, bold (700)
  And: Tablet (768px):   font-size 64px, bold (700)
  And: Desktop (1920px): font-size 72px, bold (700)
  And: Cor dinamicamente ajustada conforme range:
    - <0°C → azul (#2563eb)
    - 0-15°C → azul claro (#60a5fa)
    - 15-25°C → verde (#10b981)
    - 25-35°C → laranja (#f97316)
    - >35°C → vermelho (#ef4444)
  And: Unidade (°C ou °F) é exibida ao lado (mesmo tamanho)
  And: Temperatura é o primeiro elemento focável (Tab order)
  And: Line-height ≤ 1.2 (sem espaço extra abaixo)
```

**AC2.3 — Condição meteorológica com ícone e descrição**
```gherkin
Given: Dados de clima incluem condição (ex: "Rainy", "Sunny", "Cloudy")
When: Página renderiza
Then: Ícone correspondente é exibido (ex: ☀️, 🌧️, ☁️)
  And: Descrição em português é exibida (ex: "Ensolarado", "Chuva", "Nublado")
  And: Ícone + descrição ocupam espaço destacado (abaixo da temperatura)
```

**AC2.4 — Indicadores secundários (umidade, vento, sensação térmica)**
```gherkin
Given: Dados de clima foram carregados
When: Página renderiza
Then: Card "Umidade" exibe: ícone + label "Umidade" + valor em % (ex: "65%")
  And: Card "Velocidade do Vento" exibe: ícone + label "Vento" + valor em km/h (ex: "12 km/h")
  And: Card "Sensação Térmica" exibe: ícone + label + valor em °C/°F (ex: "22°C")
  And: Font-size: 14px label + 16px valor (mobile), 16px + 18px (desktop)
  And: Padding entre cards: ≥12px (horizontal), ≥8px (vertical)
  And: Layout grid 3 colunas em mobile (se espaço), wrap em viewport <480px
  And: Linha-height 1.5 para legibilidade
  And: Se campo = null/undefined, renderizar "—" em lugar do valor
  And: Zero console errors
```

**AC2.5 — Pressão atmosférica (seção secundária)**
```gherkin
Given: Dados incluem pressão
When: Página renderiza
Then: Pressão é exibida em seção secundária ou "Mais detalhes"
  And: Valor é mostrado em hPa (ex: "1013 hPa")
  And: Não compite com temperatura + condição (informação principal)
```

**AC2.6 — Timestamp de atualização**
```gherkin
Given: Dados foram carregados da API
When: Página renderiza
Then: Timestamp é exibido (ex: "Atualizado há 5 min" ou "12 ago, 14:30")
  And: Formato é legível e em português
  And: Se dados vêm de cache, texto muda (ex: "Dados em cache")
```

**AC2.7 — Erro de API**
```gherkin
Given: Requisição à API retorna erro (5XX, timeout, network fail)
When: App detecta o erro
Then: Mensagem "Erro ao carregar clima. Tente novamente." é exibida
  And: Botão "Tentar Novamente" é fornecido
  And: Se cache válido exists, fallback é oferecido
  And: Nenhuma informação parcial/incompleta é exibida
```

**AC2.8 — Dados nulos ou incompletos**
```gherkin
Given: API retorna resposta com campo null (ex: vento = null)
When: Página renderiza
Then: Campo exibe "—" ou "N/A" (não branco/vazio)
  And: Nenhum console error é gerado
  And: Outros campos continuam renderizando normalmente
```

---

#### 🎯 US3: Visualizar previsão de 5 dias — RF3

**Como** Sofia (viajante planificadora),  
**Quero** ver a previsão para os próximos 5 dias com ícone, temp max/min e chance de chuva,  
**Para** planejar minhas atividades ao ar livre sem surpresas climáticas.

**Critérios de Aceite (BDD — Given/When/Then):**

**AC3.1 — Cards de previsão diária**
```gherkin
Given: Dados de previsão foram carregados da API
When: Página renderiza seção "Previsão"
Then: Exatamente 5 cards são exibidos: "Hoje" + 4 próximos dias
  And: Cada card contém: data + ícone + condição + temp max/min + % chuva
  And: Todos os 5 cards têm altura idêntica (±2px tolerance)
  And: Layout: Flexbox com min-height definido (não auto-grow)
  And: Separação visual entre "Hoje" (grupo 1) e "Próximos 4 dias" (grupo 2):
    - Header "Hoje" acima do primeiro card OU
    - Primeira card com background/border distinto OU
    - Espaço visual maior entre grupos
  And: Mobile carousel scrollable, desktop grid visível simultaneamente
```

**AC3.2 — Conteúdo de cada card de dia**
```gherkin
Given: Um card de previsão é renderizado
When: Usuário visualiza o card
Then: Data é exibida no formato "Dia, DD mês" (ex: "Qua, 12 ago")
  And: Ícone da condição é mostrado (☀️, ☁️, 🌧️, etc.)
  And: Descrição em português é exibida (ex: "Ensolarado", "Chuva")
  And: Temperatura máxima é exibida (ex: "28°C")
  And: Temperatura mínima é exibida (ex: "18°C")
  And: Probabilidade de chuva é exibida em % (ex: "15%")
  And: Max/min podem ser em °F se unidade foi alternada
```

**AC3.3 — Layout responsivo (mobile vs desktop)**
```gherkin
Given: App é acessada em viewport 320px (mobile)
When: Seção de previsão é renderizada
Then: Cards estão em scroll horizontal (carousel com flex-box)
  And: Um card visível por vez (card width = 100% - padding)
  And: Scroll horizontal permite ver próximos cards
  And: Sem scroll vertical desnecessário (overflow-y hidden)
  And: Touch/mouse drag funciona fluidamente:
    - 60fps (16ms frame time máximo)
    - Sem lag perceptível ao arrastar
    - Momentum scroll em iOS (não trava ao soltar)
  And: Scroll snap snap-type: x mandatory (cards snappam após drag)
  And: Testado em device físico + DevTools Performance profiler
  And: Sem console errors durante scroll
```

**AC3.4 — Layout desktop**
```gherkin
Given: App é acessada em viewport 1024px+ (desktop)
When: Seção de previsão é renderizada
Then: Cards são exibidos em **grid** (5 colunas ou wrap responsivo)
  And: Todos os 5 dias são visíveis sem scroll
  And: Cards têm altura igual, distribuição uniforme
```

**AC3.5 — Carregamento junto com clima atual**
```gherkin
Given: Usuário selecionou uma cidade
When: App faz requisição à API
Then: Clima atual + Previsão 5 dias são **carregados juntos** (<1s total)
  And: Ambas as seções aparecem quase simultaneamente
  And: Nenhuma é renderizada separadamente (sem flickering)
  And: Loading spinner é mostrado antes de ambas chegarem
```

**AC3.6 — Dados nulos ou indisponíveis**
```gherkin
Given: API retorna previsão com campo null (ex: chuva = null)
When: Card é renderizado
Then: Campo exibe "—" (não branco/vazio)
  And: Outros campos do card continuam normais
  And: Nenhum console error
```

**AC3.7 — Datas sempre futuras ou presente**
```gherkin
Given: Previsão foi carregada
When: Cards são renderizados
Then: Primeiro card é sempre "hoje" (data = today)
  And: Próximos 4 cards são datas futuras (tomorrow, +2d, +3d, +4d)
  And: Nenhum card tem data no passado
  And: Ordem é cronológica (hoje → futuro)
```

---

#### 🎯 US4: Alternar unidade de temperatura — RF4

**Como** João (profissional internacional),  
**Quero** alternar entre Celsius e Fahrenheit com um toggle,  
**Para** compreender intuitivamente as temperaturas conforme minha preferência ou contexto geográfico.

**Critérios de Aceite (BDD — Given/When/Then):**

**AC4.1 — Toggle visível e acessível**
```gherkin
Given: Página carregou
When: Usuário visualiza a interface
Then: Toggle/seletor "°C / °F" é visível (ex: top-right ou header)
  And: Toggle tem label claro ou aria-label = "Alternar unidade de temperatura"
  And: Toggle é acessível via Tab e clicável via Enter/Space
  And: Estado atual (°C ou °F) é indicado visualmente
```

**AC4.2 — Alternância de unidade**
```gherkin
Given: App mostra temperatura em Celsius (padrão: 20°C)
When: Usuário clica toggle para Fahrenheit
Then: Temperatura atual é convertida: 20°C → 68°F (fórmula: C × 9/5 + 32)
  And: Número é atualizado em <50ms (visualmente instantâneo)
  And: **TODAS** as temperaturas mudam simultaneamente:
    - Clima atual
    - Previsão 5 dias (max/min)
    - Sensação térmica
  And: Sem flickering ou múltiplas renderizações
  And: Unidade (°C ou °F) aparece ao lado de cada valor
  And: localStorage atualizado imediatamente
  And: No reload de página necessário
```

**AC4.3 — Conversão precisa**
```gherkin
Given: Múltiplas temperaturas são exibidas
When: Usuário alterna unidade
Then: Cada temperatura é convertida corretamente:
  And: 0°C = 32°F
  And: -10°C = 14°F
  And: 25°C = 77°F
  And: 30°C = 86°F
  And: Nenhuma temperatura é arredondada incorretamente (usar Math.round ou similar)
```

**AC4.4 — Persistência em localStorage**
```gherkin
Given: Usuário alternou unidade para Fahrenheit
When: Usuário fecha app (tab/browser)
Then: localStorage key "temperatureUnit" é atualizado com valor "fahrenheit"
  And: Valor está persistido (pode ser verificado em DevTools)
  And: Nenhuma informação sensível é armazenada
```

**AC4.5 — Restauração ao reabrir**
```gherkin
Given: Usuário abriu app antes com unidade em °F
When: Usuário reabre app (nova sessão, mesma aba)
Then: Clima é exibido em °F (não retorna a °C padrão)
  And: Toggle mostra °F como selecionado
  And: localStorage foi lido corretamente na inicialização
```

**AC4.6 — Primeira visita (valor padrão)**
```gherkin
Given: Novo usuário (localStorage vazio)
When: App carrega pela primeira vez
Then: Unidade padrão é Celsius (°C)
  And: Toggle mostra °C selecionado
  And: Todas as temperaturas exibem em °C
  And: localStorage é criado/atualizado após interação com toggle
```

**AC4.7 — Alternância múltipla**
```gherkin
Given: App está com Celsius
When: Usuário clica toggle 3 vezes (°C → °F → °C → °F)
Then: Cada alternância funciona corretamente
  And: Conversão continua precisa após cada mudança
  And: localStorage é atualizado a cada clique
  And: Nenhuma duplicação ou erro acumula
```

---

#### 🎯 US5: Modo offline com cache de 24h — RF5

**Como** Sofia (viajante em roaming),  
**Quero** ver o último clima consultado quando sem conexão de internet,  
**Para** ter acesso mínimo à informação mesmo offline ou com dados caros (roaming).

**Critérios de Aceite (BDD — Given/When/Then):**

**AC5.1 — Carregamento de cache quando online**
```gherkin
Given: Usuário consultou uma cidade (ex: São Paulo) e dados foram carregados
When: App recebe dados da API
Then: Cache é armazenado em localStorage com chave "weatherCache"
  And: Cache contém: { city, latitude, longitude, current, forecast, timestamp }
  And: Timestamp é registrado em Unix milliseconds ou ISO string
  And: Apenas **última consulta** é salva (anterior é sobrescrita)
```

**AC5.2 — Detecção de modo offline**
```gherkin
Given: Usuário está sem conexão (offline detectado)
When: Usuário abre app ou tenta atualizar
Then: App detecta offline (via navigator.onLine ou fetch error)
  And: Banner/indicador "🔴 Sem conexão" aparece no topo
  And: Botão "Atualizar" é desabilitado/cinzento
```

**AC5.3 — Cache válido (<24h) offline**
```gherkin
Given: Usuário está offline
  And: Cache existe com timestamp de 2 horas atrás
  And: (Agora - timestamp) < 86400000ms (24 horas = 86400 segundos)
When: App carrega
Then: Dados em cache são exibidos (cidade, clima, previsão)
  And: Banner amarelo aparece no topo:
    "⚠️ Sem conexão — dados podem estar desatualizados"
  And: Timestamp é mostrado em português: "Dados de 12 ago, 14:30"
  And: Seção de clima mostra badge: "(Em cache - offline)"
  And: Botão "Atualizar" é desabilitado (opacity 50%, cursor: not-allowed)
  And: User sabe que dados são antigos mas válidos
  And: Nenhuma requisição à API é tentada
```

**AC5.4 — Cache expirado (>24h) offline**
```gherkin
Given: Usuário está offline
  And: Cache existe com timestamp de 25 horas atrás
  And: (Agora - timestamp) > 86400000ms (24h)
When: App tenta carregar
Then: Mensagem é exibida: "🔴 Dados expirados. Conecte à internet para atualizar."
  And: Botão "Tentar Conectar" ou "Atualizar" é fornecido
  And: Nenhum dado em cache é mostrado (página em branco/vazia)
  And: Usuário é instruído a conectar-se
```

**AC5.5 — Sem cache e offline**
```gherkin
Given: Usuário está offline
  And: localStorage está vazio (primeira visita)
  And: Nenhum cache disponível
When: App carrega
Then: Mensagem é exibida: "📡 Sem conexão de internet. Nenhum dado disponível."
  And: Sugestão: "Conecte à internet e busque uma cidade."
  And: Campo de busca permanece acessível (para quando online)
```

**AC5.6 — Retorno online (reconexão)**
```gherkin
Given: Usuário estava offline, dados em cache exibindo
When: Usuário reconecta à internet
  And: Clica botão "Atualizar" ou app detecta reconexão
Then: Requisição é feita à API
  And: Dados frescos são carregados (<1s)
  And: Cache é atualizado com novo timestamp
  And: Banner offline desaparece
  And: Página mostra dados atualizados (sem indicador offline)
```

**AC5.7 — localStorage indisponível (quota exceeded)**
```gherkin
Given: Browser nega escrita em localStorage (quota exceeded)
When: App tenta salvar cache
Then: Erro é capturado (try-catch)
  And: Mensagem não-intrusiva: "⚠️ Cache local cheio. App continua funcionando."
  And: App continua operando normalmente (sem crash)
  And: Próxima consulta tenta salvar de novo
```

**AC5.8 — Limpeza de cache corrompido**
```gherkin
Given: localStorage contém JSON inválido em "weatherCache"
When: App tenta ler cache
Then: Erro JSON.parse() é capturado
  And: Cache é deletado automaticamente
  And: App exibe mensagem: "Cache corrompido. Carregando dados novos..."
  And: Se offline, usuário é instruído a conectar
```

**AC5.9 — Estrutura de cache**
```gherkin
Given: Cache foi armazenado
When: localStorage é inspecionado (DevTools)
Then: Cache tem estrutura: {
  city: "São Paulo",
  latitude: -23.5,
  longitude: -46.6,
  current: { temp, condition, humidity, wind, feelsLike, pressure },
  forecast: [ { date, condition, tempMax, tempMin, precipitationProbability }, ... ],
  timestamp: 1691857200000
}
  And: Nenhum campo sensível é incluído (sem API keys)
```

---

### Sprint 1: Responsividade & Acessibilidade

#### 🎯 US6: Interface mobile-first responsiva — RNF (Responsividade)

**Como** Marina (usuária mobile-first),  
**Quero** usar a app confortavelmente em meu smartphone (320px) e tablet (768px),  
**Para** ter experiência fluida sem scroll horizontal, botões pequenos ou layout quebrado.

**Critérios de Aceite (BDD — Given/When/Then):**

**AC6.1 — Funcionalidade em mobile 320px**
```gherkin
Given: App é visualizada em viewport 320px (iPhone SE)
When: Usuário navega pela interface
Then: Todos os elementos são renderizados corretamente
  And: Sem scroll horizontal (apenas vertical se necessário)
  And: Campo de busca ocupa >80% da largura
  And: Temperatura é legível e destacada
  And: Botões/cards não saem da viewport
```

**AC6.2 — Funcionalidade em tablet 768px**
```gherkin
Given: App é visualizada em viewport 768px (iPad)
When: Usuário navega pela interface
Then: Layout aproveita espaço adicional
  And: Previsão mostra 2-3 cards por linha
  And: Sem desperdício de espaço à direita/esquerda
  And: Elementos mantêm proporção visual
```

**AC6.3 — Funcionalidade em desktop 1920px**
```gherkin
Given: App é visualizada em viewport 1920px (desktop grande)
When: Usuário navega pela interface
Then: Layout é centralizado (não ocupa toda a largura)
  And: Max-width é respeitado (ex: 1200px)
  And: Previsão mostra 5 cards em grid
  And: Informações secundárias acessíveis
```

**AC6.4 — Touch targets (44×44px)**
```gherkin
Given: App é visualizada em dispositivo mobile
When: Usuário inspeciona elementos interativos (botões, toggle, links)
Then: Todos têm dimensões mínimas 44×44px
  And: Padding interno garante zona clicável
  And: Sem elementos muito pequenos (12×12px, por ex.)
  And: Espaçamento entre targets ≥8px
```

**AC6.5 — Legibilidade de texto**
```gherkin
Given: App é visualizada em mobile
When: Usuário lê texto (labels, descrições, valores)
Then: Font-size mínimo é 14px em mobile (320-767px viewport)
  And: Font-size em desktop é ≥16px (768px+)
  And: Temperatura: 48px mobile, 64px tablet, 72px desktop (AC2.2)
  And: Line-height ≥1.5 para body text, ≤1.2 para headings
  And: Contraste mínimo 4.5:1 para text normal (WCAG AA)
  And: Contraste mínimo 3:1 para text large (≥18pt ou ≥14pt bold)
  And: Letter-spacing não-negativo (sem compressão)
  And: Testado com axe DevTools ou WebAIM Contrast Checker
  And: Dark mode gloss respeita todos os requisitos acima
```

**AC6.6 — Imagens e ícones escalam**
```gherkin
Given: App exibe ícones de clima (☀️, 🌧️, etc.)
When: Viewport muda (320px → 768px → 1920px)
Then: Ícones escalam proporcionalmente
  And: Qualidade não é perdida (sem distorção)
  And: Imagens (se houver) usam max-width: 100%
  And: SVG escala sem degradação
```

**AC6.7 — Carousel em mobile**
```gherkin
Given: Previsão de 5 dias em viewport 320px
When: Usuário visualiza cards
Then: Cards estão em horizontal scroll/carousel
  And: Primeiro card tem indicador visual (ex: "Hoje")
  And: Touch drag funciona fluidamente
  And: Scroll snap (opcional, mas desejável)
```

**AC6.8 — Grid em desktop**
```gherkin
Given: Previsão de 5 dias em viewport 1024px+
When: Usuário visualiza cards
Then: Cards em grid 5 colunas (ou wrap responsivo)
  And: Sem scroll horizontal
  And: Cards têm altura uniforme
```

---

#### 🎯 US7: Acessibilidade com teclado e leitor de tela — RNF (Acessibilidade)

**Como** João (usuário com deficiência motora ou visual),  
**Quero** navegar e usar a app completamente via teclado e leitor de tela,  
**Para** ter acesso inclusivo e independente ao serviço de previsão.

**Critérios de Aceite (BDD — Given/When/Then):**

**AC7.1 — Navegação por Tab (teclado)**
```gherkin
Given: App está focada no navegador
When: Usuário pressiona Tab repetidamente
Then: Foco percorre nesta ordem exata:
  1. Campo "Buscar cidade" (search input)
  2. Botão "Buscar" (ou Enter em campo)
  3. Toggle "°C / °F" (temperature unit)
  4. Seção Clima Atual (cards de umidade/vento se focáveis)
  5. Cards de Previsão (carousel, 5 cards)
  6. Botão "Atualizar" (ou similar)
  And: Ordem visual = ordem de Tab (left-to-right, top-to-bottom)
  And: Nenhum elemento focável é pulado
  And: Foco é visível: outline ≥2px, contraste ≥3:1 com background
  And: Testado manualmente com Tab key
  And: Accessibility Inspector (DevTools) mostra Tab order correto
```

**AC7.2 — Interação via teclado (Enter/Space/Esc)**
```gherkin
Given: Um botão tem foco
When: Usuário pressiona Enter
Then: Botão é ativado (sem mouse necessário)
  And: "Atualizar" carrega dados
  And: "Tentar Novamente" retenta a requisição
```

```gherkin
Given: Toggle °C/°F tem foco
When: Usuário pressiona Space
Then: Unidade alterna (sem mouse)
  And: Mudança é imediata e visível
```

```gherkin
Given: Um modal/popup está aberto (se aplicável)
When: Usuário pressiona Esc
Then: Modal/popup fecha
  And: Foco retorna ao elemento anterior
```

**AC7.3 — Labels e ARIA para campos**
```gherkin
Given: Campo de busca é renderizado
When: User inspeciona HTML
Then: Elemento tem `<label>` com `for="search"` OU
  And: Input tem `aria-label="Buscar cidade"`
  And: Placeholder não substitui label
```

```gherkin
Given: Toggle °C/°F é renderizado
When: User inspeciona HTML
Then: Elemento tem `aria-label="Alternar unidade de temperatura"` OU
  And: Label visível que descreve função
  And: `aria-checked` indica estado (true/false)
```

**AC7.4 — ARIA para mensagens de erro/sucesso**
```gherkin
Given: Erro de API é exibido (ex: "Nenhuma cidade encontrada")
When: Usuário com leitor de tela navega
Then: Mensagem é `aria-live="polite"` ou `role="alert"`
  And: Leitor anuncia: "Erro: Nenhuma cidade encontrada"
  And: Mensagem é associada ao campo com `aria-describedby="error-msg"`
```

**AC7.5 — Estrutura semântica HTML**
```gherkin
Given: App é renderizada
When: User inspeciona estrutura HTML
Then: Página tem `<main>` tag (conteúdo principal)
  And: Página tem `<nav>` ou `<header>` (navegação, se houver)
  And: Título principal é `<h1>`
  And: Seções usam `<section>` ou `<article>` (não divs cegos)
  And: Listas usam `<ul>` ou `<ol>` (não divs)
```

**AC7.6 — Ícones com alt/aria-label**
```gherkin
Given: Ícone de clima (☀️, 🌧️) é exibido
When: User inspeciona
Then: Ícone tem `aria-label="Ensolarado"` OU
  And: Imagem tem `alt="Ensolarado"`
  And: Descrição não é redundante se texto visível existe
```

```gherkin
Given: Ícone de toggle (ex: 🌡️) é exibido
When: User inspeciona
Then: Elemento pai (button/label) tem descrição via aria-label
  And: Ícone não é envolvido em span vazio
```

**AC7.7 — Contraste de cores (WCAG AA)**
```gherkin
Given: Texto e fundo são renderizados
When: Contraste é medido
Then: Razão de contraste ≥4.5:1 (normal text)
  And: Razão ≥3:1 (large text ≥18pt ou ≥14pt bold)
  And: Dark mode glass (padrão do projeto) respeita isto
  And: Cores são testadas com ferramenta (ex: axe, WAVE)
```

**AC7.8 — Tabindex e ordem de navegação**
```gherkin
Given: App tem múltiplos elementos focáveis
When: User pressiona Tab
Then: Ordem visual = ordem de tabindex
  And: Tabindex > 0 é evitado (usar ordem natural da DOM)
  And: Tabindex = -1 em elementos ocultos (não focáveis)
  And: Tabindex = 0 em elementos customizados que precisam foco
```

**AC7.9 — Leitor de tela (NVDA, JAWS, VoiceOver)**
```gherkin
Given: App é aberta com leitor ativo
When: User navega linearmente (seta para baixo)
Then: Leitor anuncia: "Buscar cidade, caixa de texto"
  And: "Alternar unidade, botão, não selecionado"
  And: "Temperatura, 20 graus Celsius, grande"
  And: "Condição, Ensolarado"
  And: "Previsão, região, 5 cards: hoje, amanhã, ..."
```

**AC7.10 — Sem captura de teclado**
```gherkin
Given: User está navegando com teclado
When: User pressiona Tab
Then: Foco não fica preso em nenhum elemento
  And: User consegue sair via Esc ou Tab (cycle correto)
  And: Sem traps de teclado
```

---

## 4. Acceptance Criteria

### Por Feature (Resumo Verificável)

#### ✅ Feature: Busca de Cidades (RF1)

| Cenário | Entrada | Ação | Resultado Esperado | Verificável |
|---|---|---|---|---|
| Busca com resultados | "São Paulo" | Digita, aguarda 500ms | Lista com ≥1 resultado; seleciona primeiro | GET `/geocoding?name=São Paulo` < 500ms retorna array |
| Busca sem resultados | "XYZZZZ" | Digita, aguarda 500ms | Mensagem "Nenhuma cidade encontrada" | UI exibe mensagem; sem crash |
| Busca com erro de rede | "Rio" | Digita, sem internet | Mensagem "Erro de conexão. Tente novamente." | UI mostra erro; botão retry funciona |

#### ✅ Feature: Clima Atual (RF2)

| Cenário | Entrada | Ação | Resultado Esperado | Verificável |
|---|---|---|---|---|
| Carregamento bem-sucedido | Cidade: São Paulo | Seleciona cidade | Clima carrega <1s; exibe temp, condição, umidade, vento, sensação, pressão | GET `/forecast?lat=X&lon=Y` < 1s; valores renderizados; timestamp atualizado |
| Erro de API | Cidade: São Paulo | Seleciona cidade, API retorna 5XX | Mensagem: "Erro ao carregar clima." com botão retry | UI mostra erro; retry funciona |
| Dado nulo (ex: sem vento) | Cidade com vento nulo | API retorna vento = null | Campo exibe "—" ou "N/A"; sem erro visual | UI renderiza valor padrão; sem console errors |

#### ✅ Feature: Previsão 5 Dias (RF3)

| Cenário | Entrada | Ação | Resultado Esperado | Verificável |
|---|---|---|---|---|
| Carregamento bem-sucedido | Cidade: São Paulo | Seleciona cidade | 5 dias aparecem (hoje + 4 próximos); cada um com temp max/min, ícone, % chuva | GET `/forecast?...` retorna 5 dias; renderizados com formatação correta |
| Scroll horizontal mobile | 5 dias em carousel | Swipe/scroll left/right | Próximos dias aparecem; sem quebra de layout | CSS/JS carousel funciona; responsivo |

#### ✅ Feature: Toggle °C / °F (RF4)

| Cenário | Entrada | Ação | Resultado Esperado | Verificável |
|---|---|---|---|---|
| Alternar unidade | Temp 20°C | Clica toggle | Exibe 68°F em todos os campos; localStorage atualizado | temp = 20; 20 × 9/5 + 32 = 68 ✓; localStorage.getItem('temperatureUnit') = 'fahrenheit' |
| Persistência | User: visitou em °F | Fecha/reabre app | App abre em °F (não volta a °C padrão) | localStorage persiste; UI inicializa com preferência salva |

#### ✅ Feature: Cache Offline 24h (RF5)

| Cenário | Entrada | Ação | Resultado Esperado | Verificável |
|---|---|---|---|---|
| Offline com dados válidos | Sem internet; dados <24h | Abre app | Exibe clima + banner "Sem conexão"; timestamp visível | localStorage.getItem('weatherCache') retorna dados; timestamp diff < 24h; banner renderizado |
| Offline com dados expirados | Sem internet; dados >24h | Abre app | Exibe "Dados expirados. Conecte para atualizar." | localStorage timestamp diff > 24h; mensagem renderizada |
| Retorno online | Estava offline | Reconecta (button retry) | App recarrega dados da API; remove banner offline | API chamada; cache atualizado; banner desaparece |

---

## 5. Non-Functional Requirements

### Performance

| ID | Requisito | Métrica | Justificativa |
|---|---|---|---|
| **RNF-PERF-1** | Carregamento inicial | <2s em mobile 4G (LTE) | Marina precisa resultado rápido; taxa de abandono >2s é alta |
| **RNF-PERF-2** | Carregamento de clima | <1s (P50 latência de rede) | Feedback visual rápido; UX responsiva |
| **RNF-PERF-3** | Autocompletar de cidades | <500ms entre keystroke e resultado | Debounce + cache local garante fluidez |
| **RNF-PERF-4** | Tamanho total (gzipped) | <150 KB (HTML + CSS + JS) | Economia de banda mobile; carregamento rápido |
| **RNF-PERF-5** | Latência de interação (TTI) | <3s em 4G | Time-to-interactive; user pode interagir rápido |

### Disponibilidade & Confiabilidade

| ID | Requisito | Métrica | Justificativa |
|---|---|---|---|
| **RNF-AVAIL-1** | Uptime do serviço | ≥99% mensal | SLA padrão; 7h de downtime/mês aceitável no MVP |
| **RNF-AVAIL-2** | SLA da API meteorológica | ≥99% (Open-Meteo contratualmente) | Dependência crítica; fallback via cache se cair |
| **RNF-AVAIL-3** | MTTF (Mean Time To Failure) | >1.000 horas | Estabilidade em ambiente de produção |
| **RNF-AVAIL-4** | MTTR (Mean Time To Recovery) | <30 min para restauração manual | Rápida resposta a incidentes |

### Acessibilidade

| ID | Requisito | Padrão | Justificativa |
|---|---|---|---|
| **RNF-A11Y-1** | Conformidade com WCAG | WCAG 2.1 AA | Legal (Lei de Acessibilidade Digital se aplicável); inclusão |
| **RNF-A11Y-2** | Suporte a leitor de tela | NVDA, JAWS, VoiceOver | 15% da população tem deficiência visual |
| **RNF-A11Y-3** | Navegação por teclado | 100% da UI navegável sem mouse | Deficiência motora; produtividade |
| **RNF-A11Y-4** | Contraste de cores | Mínimo 4.5:1 | Dark mode gloss; legibilidade para baixa visão |

### Responsividade

| ID | Requisito | Métrica | Justificativa |
|---|---|---|---|
| **RNF-RESP-1** | Breakpoints | 320px, 640px, 768px, 1024px, 1280px | Cobertura mobile-first: iPhone SE, mid-range, tablet, desktop |
| **RNF-RESP-2** | Touch targets | Mínimo 44×44px | Recomendação Apple/Google; evita misclicks |
| **RNF-RESP-3** | Viewport meta | `width=device-width, initial-scale=1` | Sem zoom automático; legibilidade em dispositivos |

### Segurança

| ID | Requisito | Controle | Justificativa |
|---|---|---|---|
| **RNF-SEC-1** | HTTPS obrigatório | SSL/TLS em produção | Proteção de dados em trânsito |
| **RNF-SEC-2** | Sem armazenamento sensível | localStorage apenas para preferências (unit, city, cache) | Privacidade; sem LGPD/GDPR violations (sem auth) |
| **RNF-SEC-3** | CORS configurado | Apenas domínios confiáveis | Proteção contra XSS/CSRF |
| **RNF-SEC-4** | CSP (Content Security Policy) | Header CSP padrão | Mitigação de injeção |

### Usabilidade

| ID | Requisito | Métrica | Justificativa |
|---|---|---|---|
| **RNF-USE-1** | Curva de aprendizado | Usuário novo completa fluxo em <30s | Interface auto-explicativa; Onboarding mínimo (Marina) |
| **RNF-USE-2** | Feedback visual | Indicadores de loading, erro, sucesso | Usuário sempre sabe o estado da app |
| **RNF-USE-3** | Idioma | Português-BR único no MVP | Familiaridade com público-alvo |

### Manutenibilidade

| ID | Requisito | Métrica | Justificativa |
|---|---|---|---|
| **RNF-MAINT-1** | Cobertura de testes | ≥80% (unit + E2E) | Confiança em refatorações; reduz bugs em produção |
| **RNF-MAINT-2** | TypeScript strict mode | `strict: true` no tsconfig.json | Type safety; menos erros em runtime |
| **RNF-MAINT-3** | Documentação inline | JSDoc para funções públicas | Manutenção futura facilitada |
| **RNF-MAINT-4** | Linting & formatação | Biome (lint + format) em CI/CD | Consistência de código |

---

## 6. Edge Cases

### Entrada Inválida

| Caso | Entrada | Comportamento Esperado |
|---|---|---|
| **Busca vazia** | User clica search sem digitar | Campo recebe focus; sem requisição API; hint text visível ("Busque por cidade...") |
| **Busca com caracteres especiais** | "São Paulo", "Dão", "Québec" | API geocoding suporta UTF-8; resultados corretos |
| **Busca com números** | "123", "NYC2" | API retorna zero resultados; mensagem clara |
| **Busca muito longa** | "A" × 500 caracteres | Limitar a 100 caracteres; warn user com aria-live |

### Conectividade

| Caso | Cenário | Comportamento Esperado |
|---|---|---|
| **Falha de rede intermitente** | GET /forecast falha a 1ª vez, sucesso a 2ª | Retry automático após 1s; máximo 3 tentativas; depois manual |
| **Timeout de API** | Geocoding demora >2s | Exibir spinner; após 3s, mensagem "Servidor lento. Tente novamente." |
| **API retorna erro 5XX** | GET /forecast retorna 503 | Exibir "Serviço temporariamente indisponível. Carregando cache..."; fallback offline |
| **IP ou DNS bloqueado** | Sem resolução DNS | Detectar offline local; exibir "Sem conexão com internet" |

### Dados Inválidos

| Caso | Dado | Comportamento Esperado |
|---|---|---|
| **Latitude/longitude nula** | `lat = null, lon = null` | Validar antes de API; exibir erro "Localização inválida" |
| **Temperatura impossível** | `temp = -300` (Kelvin error) | Validação de range: -50°C a 50°C no MVP; fora disso, exibir "N/A" |
| **Campo faltando na API** | Resposta sem `humidity` | Renderizar "—" em vez de crash; console.warn |
| **Data no passado** | Previsão com data anterior a hoje | Filtrar; exibir apenas datas ≥ hoje |

### Estados Extremos

| Caso | Cenário | Comportamento Esperado |
|---|---|---|
| **Sem conexão ao iniciar** | User abre app offline | Exibir "Sem conexão" + cache (se existe) |
| **Cache vazio + offline** | 1º acesso sem internet | Exibir "Sem dados. Conecte à internet para começar." |
| **Múltiplas buscas rápidas** | User clica 5 cidades em 2s | Debounce/throttle requisições; carregar última; cancelar anteriores |
| **localStorage cheio** | Browser nega escrita (quota exceeded) | Exibir warn: "Cache local cheio"; app continua funcionando (sem persistência) |
| **Formato de localStorage corrompido** | JSON inválido em cache | Try-catch; limpar cache; reload |

### Timezone e Horário

| Caso | Cenário | Comportamento Esperado |
|---|---|---|
| **Fuso horário incorreto** | Usuário em São Paulo vê horário de NY | Exibir hora local do usuário (via `new Date()` do browser) |
| **DST (Daylight Saving Time)** | Transição de horário de verão | Rely em JS nativo; browser cuida automaticamente |
| **Relógio do dispositivo atrasado** | Device clock = 2 meses atrás | Validar timestamp; avisar se cache > real time (rare) |

---

## 6.1. Cenários Detalhados: Busca, API e Resposta

Esta subseção detalha comportamentos esperados para edge cases críticos não totalmente cobertos acima.

### EC1: Input Vazio

**Cenário:** Usuário ativa o campo de busca mas não digita nada, ou digita apenas espaços.

| Aspecto | Comportamento Esperado | Validação |
|---|---|---|
| **Ação do usuário** | Clica campo; vê placeholder "Busque por cidade..." | Campo tem focus, outline visível |
| **Submissão sem conteúdo** | Pressiona Enter com input vazio ou só espaços | Nenhuma requisição à API é feita |
| **Feedback visual** | Mensagem: "Por favor, digite o nome de uma cidade" (aria-live) | Mensagem visível, acessível |
| **Persistência de estado** | Campo mantém foco; usuário pode digitar imediatamente | Sem limpeza prematura do campo |
| **Timeout** | Usuário espera >5s sem digitar | Mensagem desaparece; campo permanece ativo |

### EC2: Caracteres Especiais e UTF-8

**Cenário:** Entrada com acentos, caracteres internacionais, símbolos ou emojis.

| Entrada | Esperado | Validação |
|---|---|---|
| "São Paulo" | ✅ Retorna resultados ("São Paulo, Brasil") | Open-Meteo suporta UTF-8; API test confirm |
| "Dão" | ✅ Retorna sugestões com o caractere preservado | Codificação UTF-8 mantida em requisição + resposta |
| "Québec" | ✅ Retorna "Québec, Canadá" | Caracteres diacríticos processados corretamente |
| "Москва" (Moscou em Russo) | ✅ Retorna "Moscow, Russia" (transliteração) | Open-Meteo trata alfabetos diferentes |
| "北京" (Beijing em Chinês) | ✅ Retorna "Beijing, China" (se API suporta) | Testado; fallback: zero resultados se não suportado |
| "São Paulo 🌞" | ❌ Sem resultados (emoji não é cidade válida) | Mensagem: "Nenhuma cidade encontrada" |
| "São Paulo!!!!" | ⚠️ API ignora símbolos extras; retorna "São Paulo" | Limpeza de input recomendada (trim, remove special chars?) |

**Validação:**
- Requisição URL codifica UTF-8: `encodeURIComponent("São Paulo")` = `%C3%A3o%20Paulo`
- Resposta JSON mantém UTF-8: `"name": "São Paulo"`
- Rendering no DOM não corrompe caracteres: `<div>São Paulo</div>` renderiza corretamente

### EC3: Cidade Inexistente ou Geocoding Sem Resultados

**Cenário:** Usuário busca por localidade que não existe, é misspelled, ou é muito vaga.

| Entrada | Resposta API | Comportamento Esperado | UX |
|---|---|---|---|
| "XYZZZZ" | `{ results: [] }` | Mensagem: "Nenhuma cidade encontrada" | Sem crash; sem requisição de clima |
| "Blahblah123" | `{ results: [] }` | Mesmo acima | Campo mantém foco; botão "Tentar novamente" (opcional) |
| "São" (busca curta) | `{ results: [] }` ou truncated | App aguarda ≥2 caracteres antes de requisitar; se requisita, trata vazio | Aviso: "Digite mais caracteres" (se <2 chars) |
| "Paris" (sem país) | `{ results: [{ name: "Paris", country: "France" }, ...] }` | Retorna múltiplas cidades chamadas Paris (França, EUA, etc.) | User seleciona uma; mostrar país na sugestão |
| "123456789" (números) | `{ results: [] }` | Sem sugestões; mensagem clara | Sem requisição desnecessária |
| "" (string vazia após trim) | Validação local previne requisição | Mensagem: "Por favor, digite uma cidade" | Sem requisição à API |

**Validação:**
- GET `/geocoding?name=XYZZZZ` retorna `results: []` (array vazio, não error)
- App detecta `results.length === 0` e exibe mensagem
- Console não mostra erros (graceful handling)
- Campo permanece focado para retry

### EC4: Falha de API (Erro 5XX, Network Error)

**Cenário:** Servidor Open-Meteo está indisponível (503, 500, etc.) ou conexão de rede falha.

| Erro | HTTP | Cenário | Comportamento Esperado | Retry |
|---|---|---|---|---|
| **Service Unavailable** | 503 | Open-Meteo maintenance | Mensagem: "Serviço temporariamente indisponível. Tente novamente em alguns minutos." | Retry automático 1× após 2s; depois manual |
| **Server Error** | 500 | Bug no servidor API | Mensagem: "Erro no servidor. Contate o suporte." (genérica) | Manual retry via botão |
| **Bad Gateway** | 502 | Proxy/load balancer falhou | Mensagem: "Conexão com servidor falhou." | Retry automático 1× após 1s; depois manual |
| **Network Timeout** | 0 (fetch timeout) | Request demora >5s, browser aborta | Mensagem: "Conexão lenta. Tente novamente." | Manual retry |
| **CORS error** | 0 (blocked by browser) | Origem não permitida | Mensagem: "Erro de acesso. Contate o admin." | Não há retry (problema de configuração) |
| **DNS resolution fail** | 0 (offline local) | Hostname não resolve | Mensagem: "Não foi possível alcançar o servidor." | Retry automático; se persiste, sugerir "Verifique sua conexão" |

**Validação:**
- Response status verificado: `if (response.status >= 500)`
- Mensagem de erro é **específica** (não genérica) quando possível
- Retry logic: `max_retries = 3`, `delay = exponential backoff (1s, 2s, 4s)`
- User é informado do retry: "Tentando novamente (1/3)..."
- Cache é oferecido como fallback: "Usando dados em cache enquanto isso."

### EC5: Timeout de API

**Cenário:** Requisição geocoding ou forecast demora muito tempo (>2s para geocoding, >3s para forecast).

| Fase | Timeout | Comportamento | UX | Fallback |
|---|---|---|---|---|
| **Geocoding** | >500ms | Mostrar spinner com msg "Buscando cidades..." | User vê progresso | Aguardar até 2s; depois erro "Servidor lento" |
| **Geocoding** | >2s | Cancelar requisição; exibir erro | Msg: "Busca demorou muito. Tente novamente." | User pode clicar botão "Tentar" |
| **Forecast (clima)** | >1s | Mostrar skeleton/spinner | "Carregando clima..." | Aguardar até 3s |
| **Forecast** | >3s | Cancelar; exibir erro | Msg: "Clima demorou. Tente novamente." | Oferecer cache se existe |

**Implementação:**
```javascript
// Pseudocode
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 2000); // 2s timeout

try {
  const response = await fetch(url, { signal: controller.signal });
  // ...
} catch (error) {
  if (error.name === 'AbortError') {
    showError("Requisição expirou. Tente novamente.");
  }
} finally {
  clearTimeout(timeoutId);
}
```

**Validação:**
- Timeout é respeitado (não dispara após cancelamento)
- Spinner é exibido enquanto aguarda
- Mensagem de erro aparece em <100ms após timeout
- User pode clicar "Tentar Novamente" imediatamente

### EC6: Resposta Parcial ou Dados Inválidos da API

**Cenário:** Open-Meteo retorna resposta com campos faltando, valores inválidos, ou estrutura corrompida.

| Resposta da API | Campo(s) | Comportamento Esperado | Renderização |
|---|---|---|---|
| **Resposta incompleta** | Falta `humidity` | Renderizar "—" em lugar do valor | Outros campos normais; sem crash |
| **Campo nulo** | `temp: null` | Exibir "—" ou "Indisponível" | Label visível, valor placeholder |
| **Campo vazio** | `condition: ""` (string vazia) | Renderizar "—" ou ícone genérico | Não deixar branco; usar fallback |
| **Tipo incorreto** | `temp: "20" (string)` | Parsear para número; se falhar, "—" | Tentar conversão; fallback para "—" |
| **Valores fora de range** | `temp: -300` (Kelvin?) | Validar range -50°C a 50°C; fora disso, "N/A" | Não exibir valor impossível |
| **Data inválida** | `date: "99-99-9999"` | Não renderizar; exibir "Data indisponível" | Skip card ou mostrar placeholder |
| **Array vazio** | `forecast: []` | Mensagem: "Previsão não disponível no momento" | Seção "Previsão" fica vazia ou hidden |
| **JSON malformado** | Response é `{ invalid: ` (truncado) | Capturar erro JSON.parse(); refazer requisição | Botão retry disponível |

**Validação:**
```javascript
// Exemplos de validação
const validateTemp = (temp) => {
  if (typeof temp !== 'number' || temp < -50 || temp > 50) return '—';
  return temp;
};

const validateHumidity = (humidity) => {
  if (!humidity || isNaN(humidity) || humidity < 0 || humidity > 100) return '—';
  return humidity;
};

const safeRender = (value, fallback = '—') => value ?? fallback;
```

**Teste:**
- API mock retorna campos nulos
- App não lança console errors
- UI renderiza gracefully sem crash
- Valores válidos são exibidos; inválidos mostram fallback

### EC7: Resposta Bem-sucedida mas com Dados Mínimos

**Cenário:** API retorna status 200, mas com dados muito reduzidos (ex: apenas temperatura, sem umidade/vento).

| Resposta | Campos Presentes | Comportamento | UX |
|---|---|---|---|
| **Mínimo absoluto** | `{ temp, condition }` | Renderizar o que existe; outros como "—" | Seção primária (temp + condição) funciona |
| **Sem previsão** | Clima atual OK, `forecast: []` | Exibir clima; seção previsão mostra "Indisponível" | User vê algo útil; sabe que previsão falta |
| **Sem indicadores secundários** | Sem `humidity`, `wind`, `pressure` | Cards de umidade/vento exibem "—" | Grid mantém estrutura; espaçamento uniforme |

**Validação:**
- App prioriza dados essenciais (temperatura, condição)
- Sem crash se dados opcionais faltam
- User vê feedback claro (—, N/A, "Indisponível")

---

## 7. Assumptions

### Plataforma & Dispositivos

- ✅ **Navegador moderno:** App funciona em Chrome, Firefox, Safari, Edge (últimas 2 versões)
- ✅ **Web-based (não nativa):** Desktop PWA e web app; sem apps iOS/Android no MVP
- ✅ **JavaScript habilitado:** User liga JS no browser (fallback sem JS não é suportado)

### API & Dados

- ✅ **Fonte: Open-Meteo (gratuita, sem API key):** Geocoding + Weather Forecast em mesmo endpoint
- ✅ **Geolocalização: Manual (não automática):** Sem GPS/IP geolocation; user busca cidade
- ✅ **Dados atualizados a cada consulta:** Sem polling automático no MVP
- ✅ **Suporta coordenadas globais:** Qualquer latitude/longitude (sem restrição regional)
- ✅ **Idioma da resposta API:** Open-Meteo retorna em inglês; conversão local (ex: "Rainy" → "Chuva") no frontend

### Persistência & Conta

- ✅ **Sem autenticação:** App anônima; nada vinculado a usuario
- ✅ **Sem servidor de backend:** Frontend-only; dados em browser (localStorage, sessionStorage)
- ✅ **Preferências locais:** Unidade de temp + último city em localStorage; limpar = reset
- ✅ **Sem sincronização em cloud:** Dados não sincronizam entre dispositivos

### UI & Experiência

- ✅ **Idioma único: Português-BR** no MVP; i18n é Phase 2+
- ✅ **Dark mode com gloss (glassmorphism):** Design system padrão do projeto Tailwind
- ✅ **Sem notificações push:** Não há service worker registration no MVP
- ✅ **Sem histórico de servidor:** Apenas cache local (última consulta)

### Stack & Infraestrutura

- ✅ **React + TypeScript + Tailwind:** Stack do projeto (conforme AGENTS.md)
- ✅ **Vite como bundler:** Build rápido; deploy em Vercel/netlify/GitHub Pages
- ✅ **Vitest + Playwright para testes:** Cobertura ≥80%
- ✅ **Biome para lint + formato:** Ferramental padronizado

---

## 8. Risks

### Riscos Técnicos

| Risco | Impacto | Probabilidade | Mitigação | Owner |
|---|---|---|---|---|
| **R1: Indisponibilidade de Open-Meteo API** | 🔴 Alto — App não funciona | 🟡 Média (99% SLA) | Implementar retry (3x); exibir cache offline por 24h | Tech Lead |
| **R2: Latência da API geocoding** | 🟠 Médio — UX degradada | 🟠 Média | Debounce 300ms; cache local de cidades buscadas; mostrar spinner |  Frontend |
| **R3: localStorage não disponível** | 🟠 Médio — Sem cache | 🔵 Baixa (moderno browsers) | Try-catch em write; aviso user "Sem persistência local"; app continua sem cache |  Frontend |
| **R4: Conversão de temperatura incorreta** | 🔴 Alto — Confiança data | 🔵 Baixa | Testes unit da fórmula; validação cruzada em E2E (ex: 0°C = 32°F) | QA |
| **R5: Performance <2s não atingida** | 🟠 Médio — Marina abandona | 🟠 Média | Lazy load; code splitting; image optimization; monitor com Lighthouse CI | DevOps |

### Riscos de Produto

| Risco | Impacto | Probabilidade | Mitigação | Owner |
|---|---|---|---|---|
| **R6: Precisão de previsão decepcionante** | 🟠 Médio — NPS baixo | 🟡 Média | Disclaimer: "Previsão informativa; validar com fonte oficial"; roadmap Phase 2 melhoria | PM |
| **R7: Usuários esperam múltiplas cidades** | 🟡 Baixo → Médio | 🟡 Média | Documentar no MVP como "Coming Soon"; roadmap Phase 2 claro; feedback user |  PM |
| **R8: Expectativa de notificações (alertas)** | 🟡 Baixo | 🟠 Média | Out of scope no MVP; registrar feature request; Phase 3 candidata | PM |

### Riscos de Segurança

| Risco | Impacto | Probabilidade | Mitigação | Owner |
|---|---|---|---|---|
| **R9: XSS via entrada de busca** | 🔴 Alto | 🔵 Muito Baixa (React escapa) | Usar `dangerouslySetInnerHTML` apenas se inevitável; DOMPurify se necessário | Security |
| **R10: CORS misuse (data leaks)** | 🟡 Baixo | 🔵 Baixa | CORS headers only from trusted origins; validate API responses |  Security |

### Riscos Operacionais

| Risco | Impacto | Probabilidade | Mitigação | Owner |
|---|---|---|---|---|
| **R11: Não atingir 80% cobertura de testes** | 🟠 Médio — Manutenção futura | 🟡 Média | Exigir CR review; CI bloqueia build <80%; mentoring do time | QA/Tech Lead |
| **R12: Deploy quebra produção** | 🔴 Alto | 🔵 Muito Baixa | Staging env; smoke tests; rollback script; monitoria pós-deploy | DevOps |

---

## 8.1 Implementation Reference — Especificações Técnicas para Desenvolvimento

### API Endpoints & Timing Specifications

#### Geocoding (RF1)

```
Endpoint: GET https://api.open-meteo.com/v1/search
Query Params:
  - name: string (≥2 chars, UTF-8 encoded)
  - count: 5 (máximo de resultados)
  - language: en (sempre English, traduzir no frontend)

Response Schema:
{
  "results": [
    {
      "id": number,
      "name": string,
      "latitude": number,
      "longitude": number,
      "country": string
    }
  ]
}

Timing SLA:
- P50 (median):       <300ms
- P95:                <500ms (AC1.2 constraint)
- P99:                <1000ms
- Max timeout:        2000ms (after which cancel request)
- Debounce:           300ms after keystroke before making request
- Cache local:        Last 20 searches in memory (session-only)
```

#### Forecast (RF2, RF3)

```
Endpoint: GET https://api.open-meteo.com/v1/forecast
Query Params:
  - latitude: number
  - longitude: number
  - current: "temperature_2m,weather_code,humidity,wind_speed_10m,apparent_temperature,pressure_msl"
  - daily: "weather_code,temperature_2m_max,temperature_2m_min,precipitation_sum,precipitation_probability"
  - timezone: string (e.g., "America/Sao_Paulo")

Response Schema:
{
  "latitude": number,
  "longitude": number,
  "timezone": string,
  "current": {
    "temperature_2m": number,
    "weather_code": number (WMO code),
    "humidity": number (0-100),
    "wind_speed_10m": number (km/h),
    "apparent_temperature": number,
    "pressure_msl": number (hPa)
  },
  "daily": {
    "time": string[] (ISO 8601 dates),
    "weather_code": number[],
    "temperature_2m_max": number[],
    "temperature_2m_min": number[],
    "precipitation_probability": number[] (0-100)
  }
}

Timing SLA:
- P50 (median):       <800ms
- P95:                <1000ms (AC2.1 constraint)
- P99:                <2000ms
- Max timeout:        3000ms (after which cancel, fallback to cache)
```

### Retry Policy (Formal RF7)

```javascript
// Geocoding
const geocodingRetry = {
  maxAttempts: 2,
  delays: [500, 1000],  // ms between retries
  backoffMultiplier: 1  // linear (not exponential for fast requests)
}

// Forecast
const forecastRetry = {
  maxAttempts: 3,
  delays: [1000, 2000, 4000],  // exponential backoff
  backoffMultiplier: 2
}

// Retry only on: 5XX, timeout, network error
// Do NOT retry on: 4XX (bad request), CORS error, validation error
```

### localStorage Schema

#### Key: `temperatureUnit`
```javascript
Type: string
Values: "celsius" | "fahrenheit"
Default: "celsius"
Lifecycle: Persistent (no expiration)
Sync: Updated immediately when user toggles
```

#### Key: `weatherCache`
```javascript
Type: JSON string
Structure: {
  city: string,               // e.g., "São Paulo, Brazil"
  latitude: number,           // e.g., -23.5505
  longitude: number,          // e.g., -46.6333
  current: {
    temperature_2m: number,
    weather_code: number,
    humidity: number,
    wind_speed_10m: number,
    apparent_temperature: number,
    pressure_msl: number
  },
  forecast: {
    time: string[],           // ISO 8601 dates (5 elements)
    weather_code: number[],
    temperature_2m_max: number[],
    temperature_2m_min: number[],
    precipitation_probability: number[]
  },
  timestamp: number           // milliseconds since epoch
}
Default: null (no cache initially)
Lifecycle: Expires after 86400000ms (24h)
Max size: ~10 KB per entry (no compression)
Sync: Updated whenever climate data is loaded successfully
```

### WMO Weather Code Mapping (PT-BR)

```
Code Range | Condição PT-BR        | Ícone | Priority (Display Order)
───────────┼───────────────────────┼──────┼─────────────────────────
0          | Céu limpo             | ☀️   | 1
1          | Parcialmente nublado  | ⛅   | 2
2          | Nublado               | ☁️   | 2
3          | Muito nublado         | 🌫️   | 2
45         | Nebuloso              | 🌫️   | 2
48         | Neblina de deposição  | 🌫️   | 2
51-55      | Chuva leve            | 🌧️   | 1
56-57      | Chuva congelante leve | 🌧️   | 1
61-63      | Chuva                 | 🌧️   | 1
64-67      | Chuva congelante      | 🌧️   | 1
71-77      | Neve                  | ❄️   | 1
80-82      | Pancadas              | ⛈️   | 1
85-86      | Pancadas de neve      | ❄️   | 1
95-99      | Tempestade            | ⛈️   | 1
DEFAULT    | Indisponível          | ❓   | 999

Implementação:
- Criar Object `const weatherCodeMap = { 0: { pt: 'Céu limpo', icon: '☀️' }, ... }`
- Fallback se código not in map: "Indisponível" + "?"
- Never leave blank or null
```

### Data Validation Ranges

```javascript
// Temperature validation (after API response)
const validateTemp = (temp) => {
  if (typeof temp !== 'number' || temp < -50 || temp > 50) {
    console.warn(`Temperature out of range: ${temp}°C, using N/A`);
    return null;  // Render as "N/A" in UI
  }
  return temp;
};

// Humidity validation (0-100%)
const validateHumidity = (humidity) => {
  if (!Number.isInteger(humidity) || humidity < 0 || humidity > 100) {
    console.warn(`Invalid humidity: ${humidity}, using N/A`);
    return null;
  }
  return humidity;
};

// Wind speed validation (0-50 km/h typical, up to 100+ for hurricanes)
const validateWindSpeed = (speed) => {
  if (typeof speed !== 'number' || speed < 0 || speed > 150) {
    console.warn(`Wind speed out of range: ${speed} km/h, using N/A`);
    return null;
  }
  return speed;
};

// Precipitation probability (0-100%)
const validatePrecipitation = (probability) => {
  if (!Number.isInteger(probability) || probability < 0 || probability > 100) {
    console.warn(`Invalid precipitation: ${probability}, using N/A`);
    return null;
  }
  return probability;
};

// Render logic: value ?? "—" (null or undefined → "—")
```

### Temperature Conversion Formula

```javascript
// Celsius to Fahrenheit (EXACT formula, no rounding during storage)
function celsiusToFahrenheit(celsius) {
  return (celsius * 9 / 5) + 32;
}

// Test cases (must all pass):
// 0°C    → 32°F  (ice point)
// -10°C  → 14°F  (very cold)
// 20°C   → 68°F  (room temp)
// 25°C   → 77°F  (warm)
// 37°C   → 98.6°F (body temp)

// Rounding: use Math.round() only for display, never during calculation
// Display: 1 decimal place (e.g., 68.0°F) or no decimals if integer
```

### Font Sizes & Typography (Responsive)

```css
/* AC2.2, AC6.5 — Temperature Display */
Temperature:
  Mobile (320px):   font-size: 48px, font-weight: 700
  Tablet (768px):   font-size: 64px, font-weight: 700
  Desktop (1920px): font-size: 72px, font-weight: 700
  Line-height:      1.2

/* Condition (Sunny, Rainy, etc.) */
  Mobile:   font-size: 18px
  Desktop:  font-size: 24px

/* Indicators (Humidity, Wind, etc.) Labels */
  Mobile:   font-size: 14px
  Desktop:  font-size: 16px
  Min:      14px (never smaller on mobile)
  Line-height: 1.5 (legibility)

/* Touch target minimum */
  All interactive: 44×44px (including padding)
```

### Color Scheme by Temperature (AC2.2 + Design System)

```
Temperature Range | Hex Color | Tailwind Class | Purpose
──────────────────┼───────────┼────────────────┼──────────────
< 0°C             | #2563eb   | blue-600       | Cold (icy)
0–15°C            | #60a5fa   | blue-400       | Cool
15–25°C           | #10b981   | emerald-600    | Comfortable (default)
25–35°C           | #f97316   | orange-500     | Hot
> 35°C            | #ef4444   | red-500        | Very hot (alert)

Implementation:
- Store in Tailwind config or CSS variable map
- Update dynamically based on current temp
- Apply to text color OR background (glassmorphism)
```

### Error Message Standards

```
Error Category | Message Template           | Icon | Tone
───────────────┼───────────────────────────┼──────┼──────────
API 5XX        | "Erro no servidor..."     | 🔴   | Technical
API 4XX        | "Requisição inválida..."  | 🟠   | Technical
Timeout        | "Conexão lenta..."        | ⏱️   | Friendly
Network        | "Sem conexão com internet"| 📡   | Friendly
No Results     | "Nenhuma cidade encontrada" | ℹ️   | Informational
Cache Expired  | "Dados expirados..."      | ⚠️   | Warning
Other          | "Algo deu errado..."      | ❓   | Generic

Rules:
- Always include action (retry button, try again, connect)
- Never show raw error codes to user (log to console instead)
- Show aria-live="polite" for screen readers
```

---

## 9. Out of Scope

### Explicitamente Fora do MVP

#### ❌ Funcionalidades Adiadas para Phase 2+

- **Previsão horária:** Apenas diária no MVP (5 × 1 dia); hourly data em Phase 2
- **Múltiplas cidades monitoradas:** Single city por sessão no MVP; favoritos em Phase 2
- **Histórico de buscas:** Apenas última consulta em cache; histórico de servidor adiado
- **Alertas/Notificações:** Push notifications, SMS alerts, e-mail forecasts → Phase 3
- **Dados avançados:** Índice UV, qualidade do ar, raios solares, marés → Phase 2+
- **Gráficos interativos:** Chart.js/D3.js com séries temporais → Phase 2 (João demand)
- **Comparação de cidades:** Lado-a-lado; multi-select → Phase 2
- **Suporte multilíngue:** Apenas pt-BR no MVP; i18n framework em Phase 2
- **Autenticação & accounts:** Sem login; sem sync em cloud
- **Integração com calendários:** Sync com Google Calendar, Apple Calendar → Phase 3+
- **Monetização:** Sem ads, sem paywall no MVP; modelo decidir em Q2

#### ❌ Requisitos Técnicos Fora do Escopo

- **PWA instalável:** Sem service worker; sem `manifest.json` no MVP (Phase 2 candidate)
- **Offline-first sync:** Sem suporte a mudanças locais que sincronizam online
- **Analytics avançada:** Sem Google Analytics 4, Sentry, etc. (apenas logs básicos)
- **A/B testing:** Sem framework (Growthbook, LaunchDarkly)
- **DevOps avançada:** Sem CD/CI complexa; deploy manual via Vercel é suficiente
- **Logging centralizador:** Sem ELK, CloudWatch; console logs locais apenas

#### ❌ Considerações Futuras (Não Começar Discussão)

- Aplicativos nativos (iOS, Android)
- API backend própria (usar Open-Meteo agregado)
- Monetização premium
- Geolocalização automática por GPS/IP
- Integração com wearables (smartwatch)

---

## 10. Open Questions

### Perguntas Respondidas ✅ (Decisões Arquiteturais)

| Pergunta | Resposta | Decisão | Impacto |
|---|---|---|---|
| "Qual será a fonte de dados meteorológicos?" | Open-Meteo (gratuita, sem API key, 99% SLA) | **D1** | ✅ $0 custo; ✅ Sem autenticação; ✅ MVP pronto |
| "A app detecta localização automaticamente via GPS/IP?" | Não — busca manual de cidade via geocoding | **D1** | ✅ Simplicidade; ✅ Sem permissões de browser |
| "Há requisito de histórico de buscas de servidor?" | Não — cache local apenas (última consulta) | **D4** | ✅ Frontend-only; ✅ Sem BD |
| "Qual o nível de granularidade horária?" | Diária apenas no MVP (5 dias); horária em Phase 2 | **D2** | ✅ MVP simples/rápido; ✅ João aguarda Phase 2 |
| "Login obrigatório?" | Não — app anônima | **D4** | ✅ Zero atrito; ✅ Sem LGPD/GDPR |
| "Unidade de temperatura padrão?" | Celsius (Brasil); toggle para Fahrenheit | **D3** | ✅ UX natural; ✅ localStorage persiste |
| "Suporte a múltiplas cidades?" | MVP com 1 cidade; multi-city Phase 2 (Sofia demand) | **D2** | ✅ Escopo reduzido; ✅ Roadmap claro |
| "Idioma da UI?" | Português-BR; i18n em Phase 2 | **D5** | ✅ MVP rápido; ✅ Público-alvo atendido |

### Perguntas Pendentes (Requer Resolução Antes de Code)

| # | Pergunta | Tipo | Resposta Esperada | Owner | Prazo |
|---|---|---|---|---|---|
| **Q1** | Qual é o SLA esperado (% uptime)? | Operacional | 99%, 99.5%, ou 99.9%? | PM | Sprint 0 |
| **Q2** | Quantos usuários simultâneos no MVP (para capacity planning)? | Arquitetura | 100, 1k, 10k? | PM/DevOps | Sprint 0 |
| **Q3** | Budget/custo máximo permitido? | Negócio | $0 (open-source only), <$100/mês, <$500/mês? | Finance | Sprint 0 |
| **Q4** | Data de go-live esperada? | Projeto | Sprint 1 end, 2 weeks, 1 month? | PM | Sprint 0 |
| **Q5** | É obrigatório suportar IE11 ou browser legado? | Compat | Sim ou não? (impacta polyfills) | PM | Sprint 0 |
| **Q6** | Há requisito de SEO (indexação em buscadores)? | Produto | Sim (SSR/SSG?) ou não (SPA é ok)? | PM | Sprint 0 |
| **Q7** | Qual é a fonte de dados de geocoding (OpenStreetMap, Google Maps, Open-Meteo)? | Dados | Explícita? (assumindo Open-Meteo) | Tech Lead | Sprint 0 |
| **Q8** | Suporte a formulários acessíveis (WAI-ARIA)? | A11y | Full WCAG 2.1 AA ou mínimo? | QA | Sprint 0 |

---

## Approval & Sign-off

| Papel | Nome | Aprovação | Data |
|---|---|---|---|
| **Product Manager** | *(nome)* | ⬜ Pendente | — |
| **Tech Lead** | *(nome)* | ⬜ Pendente | — |
| **Design/UX Lead** | *(nome)* | ⬜ Pendente | — |
| **QA Lead** | *(nome)* | ⬜ Pendente | — |

---

## Histórico de Versões

| Versão | Data | Autor | Mudanças |
|---|---|---|---|
| 1.0 | 2026-08-12 | Spec Agent | Spec inicial baseada em discovery.md; 5 stories MVP |
| — | — | — | — |

---

## Anexos

### A. Referências

- [Discovery.md](./discovery.md) — Análise de contexto, personas, riscos iniciais
- [Open-Meteo API Docs](https://open-meteo.com/en/docs) — Geocoding + Weather Forecast
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/) — Acessibilidade

### B. Glossário

| Termo | Definição |
|---|---|
| **API** | Application Programming Interface; neste contexto, Open-Meteo |
| **Geocoding** | Converter nome de cidade (texto) → latitude, longitude |
| **SLA** | Service Level Agreement; % de uptime garantido |
| **MVP** | Minimum Viable Product; versão 1.0 com o mínimo para entregar valor |
| **Phase 2/3** | Roadmap futuro; funcionalidades adiadas (multi-city, hourly, alerts, etc.) |
| **Dark mode / Glassmorphism** | Design system do projeto Tailwind; interface semi-transparente |

