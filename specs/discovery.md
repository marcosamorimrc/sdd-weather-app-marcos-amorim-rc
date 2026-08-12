# Análise de Discovery — Weather App

## 1. **Contexto**

A empresa necessita de uma aplicação de previsão do tempo (Weather App) para fornecer aos usuários acesso rápido e confiável a informações meteorológicas. A aplicação deve ser acessível em diferentes dispositivos, priorizando a experiência mobile, e permitir consultas por localização geográfica.

**Público-alvo:** Usuários finais que buscam informações de clima em tempo real e previsões curtas.

---

## 2. **Requisitos Funcionais**

| # | Requisito | Descrição |
|---|-----------|-----------|
| RF1 | Buscar cidades | Usuário deve poder pesquisar e selecionar cidades pelo nome |
| RF2 | Visualizar clima atual | Exibir temperatura, condição meteorológica, umidade, velocidade do vento e outros indicadores presentes |
| RF3 | Previsão de 5 dias | Mostrar clima previsto para os próximos 5 dias com detalhes diários |
| RF4 | Alternar unidade de temperatura | Permitir mudança entre Celsius e Fahrenheit (com persistência de preferência) |


---

## 3. **Requisitos Não-Funcionais**

| # | Requisito | Critério |
|---|-----------|----------|
| RNF1 | Performance | Carregamento de dados em <3s (rede 4G) |
| RNF2 | Disponibilidade | 99% uptime da aplicação |
| RNF3 | Escalabilidade | Suportar 10k usuários simultâneos inicialmente |
| RNF4 | Segurança | HTTPS obrigatório; sem armazenamento de dados sensíveis |
| RNF5 | Acessibilidade | Conformidade WCAG 2.1 AA (semântica, cores, contraste) |
| RNF6 | Compatibilidade | Funcional em navegadores modernos (últimas 2 versões) |
| RNF7 | Confiabilidade de dados | API de dados com SLA comprovado |
| RNF8 | Responsividade mobile | Interface funcional e otimizada para smartphones, tablets e desktops (mobile-first) |
| RNF9 | Usabilidade | Interface intuitiva; usuário novo completa busca e visualização em <30s |
| RNF10 | Capacidade Offline | Exibir última consulta válida por até 24h sem conexão (cache local) |
| RNF11 | Latência de Busca | Autocompletar de cidades em <500ms; previsão em <1s |
| RNF12 | Tempo de Carregamento Inicial | App pronta para usar em <2s (mobile 4G) |
| RNF13 | Manutenibilidade | Cobertura de testes ≥80%; TypeScript strict mode; código documentado |
| RNF14 | Recuperação de Falhas | Mensagens claras ao usuário em erros; retry automático em timeouts |

---

## 4. **Riscos**

| # | Risco | Impacto | Probabilidade | Mitigação |
|---|-------|--------|--------------|-----------|
| R1 | Indisponibilidade da API meteorológica | Alto | Média | Contrato SLA com provedor; cache local de dados |
| R2 | Latência de busca de cidades | Médio | Média | Implementar debounce; autocompletar com geocoding local |
| R3 | Consumo excessivo de banda em mobile | Médio | Alta | Compressão de imagens; lazy loading; dados otimizados |
| R4 | Localização não funciona em alguns dispositivos | Médio | Baixa | Fallback para busca manual de cidade |
| R5 | Conversão incorreta de unidades | Alto | Baixa | Testes automatizados; validação cruzada |

---

## 5. **Perguntas em Aberto**

- [ ] **Dados da API**: Qual será a fonte de dados meteorológicos? (ex: Open-Meteo, OpenWeatherMap, INMET)
- [ ] **Geolocalização**: A app deve detectar a localização do usuário automaticamente via GPS/IP?
- [ ] **Histórico**: Será necessário manter histórico de buscas anteriores?
- [ ] **Notificações**: Há requisito de alertas (ex: chuva em horário específico)?
- [ ] **Offline**: Qual comportamento esperado sem conexão?
- [ ] **Múltiplas cidades**: Usuário pode monitorar mais de uma cidade?
- [ ] **Detalhes adicionais**: Qual o nível de granularidade horária? (ex: a cada 1h, 6h ou só diário)
- [ ] **Autenticação**: Login obrigatório ou app anônima?
- [ ] **Monetização**: Há modelo de receita (ads, premium features)?

---

## 6. **Suposições**

- ✓ A aplicação será web-based (não nativa mobile)
- ✓ Será usada uma API pública de dados meteorológicos gratuita ou de baixo custo
- ✓ Conversão entre escalas será feita no frontend
- ✓ Design será responsivo (mobile-first)
- ✓ Usuário acessa via navegador padrão (sem necessidade de PWA instalável inicialmente)
- ✓ Localização padrão será por busca manual de cidade
- ✓ Dados serão atualizados a cada acesso ou periodicamente (ex: a cada 10min)
- ✓ Stack será moderna: TypeScript, React, Tailwind CSS (conforme guia do projeto)
- ✓ Testes serão inclusos (unit + E2E)

---

## 7. **Decisões Arquiteturais**

Decisões-chave que destravam a especificação e resolvem perguntas em aberto:

| # | Decisão | Justificativa | Resolve | Impacto |
|---|---|---|---|---|
| **D1** | **Fonte de dados: Open-Meteo** (sem API key, gratuito) | Sem custo operacional; sem autenticação; API confiável (99% SLA); suporta geocoding + forecast em mesmo endpoint | "Dados da API" | ✅ Reduz custo a $0; ✅ Simplifica arquitetura (sem secrets); ✅ Pronto para MVP |
| **D2** | **Granularidade: Hoje + 4 dias = "5 dias"** (diário, não horário) | Reduz complexidade de UI; suficiente para Marina/Sofia; João pode escalar em Phase 2 | "Detalhes adicionais - granularidade horária" | ✅ MVP mais simples (sem gráficos); ✅ Rápido de carregar; ⚠️ João aguarda Phase 2 |
| **D3** | **Unidade padrão: Celsius** (com toggle para Fahrenheit) | Brasil usa Celsius; compatível com Open-Meteo; preferência persistida em localStorage | "Alternar unidade de temperatura" | ✅ UX natural para público-alvo; ✅ Sem dependência de BD |
| **D4** | **Sem autenticação, sem persistência de servidor** (frontend-only) | Reduz infraestrutura (apenas Vercel/static); sem LGPD de BD; usuário anônimo | "Autenticação", "Histórico de servidor" | ✅ Deploy simples; ✅ Privacidade garantida; ✅ Sem custo de BD |
| **D5** | **Idioma da UI: português-BR (pt-BR)** | Público-alvo nacional; reduz i18n complexity no MVP | "Suporte a idiomas" | ✅ MVP lançado rápido; ✅ i18n como feature futura (Phase 2+) |

### Perguntas em Aberto Resolvidas ✅

| Pergunta | Resposta | Vinculada a |
|---|---|---|
| "Dados da API?" | Open-Meteo (gratuita, sem key) | D1 |
| "Geolocalização automática?" | Não (busca manual via Open-Meteo geocoding); sem GPS | D1 |
| "Histórico de buscas?" | Não em servidor; localStorage pode guardar 5 últimas no Phase 2 | D4 |
| "Notificações/alertas?" | Fora do MVP; considerar Phase 3 | — |
| "Offline?" | Cache local até 24h (já em RNF10) | — |
| "Múltiplas cidades?" | MVP com 1 cidade; multi-city em Phase 2 (ver Persona Sofia) | D2 |
| "Granularidade horária?" | MVP diária (5 dias); horária em Phase 2 (ver Persona João) | D2 |
| "Autenticação?" | Não; app anônima | D4 |
| "Monetização?" | MVP gratuita; modelo de receita decidir em Q2 | — |

---

## 8. **Personas**

### 👤 Persona 1: Marina — O Usuário Casual/Apressado

| Aspecto | Descrição |
|---|---|
| **Perfil** | Profissional urbano (28-45 anos), centros urbanos |
| **Dispositivo primário** | Smartphone (85% do tempo) |
| **Frequência de uso** | 1-3 vezes/dia, em bursts rápidos (<1 min/sessão) |
| **Objetivo principal** | "Verificar rapidamente se preciso levar guarda-chuva ou jaqueta" |
| **Contexto de uso** | Manhã (antes de sair), em movimento, 4G/WiFi, 10-15s de atenção |
| **Pontos de dor** | App lento (>2s) = abandono; muita informação; confusão C/F; "agora" vs "previsão" |
| **Métrica de sucesso** | ⚡ Tempo <2s até temperatura + condição; 🎯 Acurácia 2-3h; 🔄 NPS ≥7 |

### 👤 Persona 2: João — O Profissional Especializado

| Aspecto | Descrição |
|---|---|
| **Perfil** | Agricultor/meteorologista/piloto/guia (35-55 anos), áreas rurais/costais/montanhosas |
| **Dispositivo primário** | Desktop (40%) + Tablet (60%) |
| **Frequência de uso** | 5-10 vezes/dia, sessões prolongadas (5-10 min) |
| **Objetivo principal** | "Acompanhar evolução do clima para tomar decisões críticas de negócio" |
| **Contexto de uso** | Períodos de trabalho (6h-18h), ambiente controlado, conectividade variável |
| **Pontos de dor** | Falta de detalhes (vento, pressão); dados horários insuficientes; imprecisão = perda $; dificuldade em comparar múltiplas cidades |
| **Métrica de sucesso** | 📊 Previsão horária 7 dias; 🎯 Acurácia <±1°C; 🔍 Gráficos (vento, umidade, UV); 💾 Histórico; 🏆 3+ usos/dia, sessões >5 min |

### 👤 Persona 3: Sofia — A Viajante/Aventureira

| Aspecto | Descrição |
|---|---|
| **Perfil** | Viajante/backpacker/influencer (22-35 anos), múltiplas cidades/países |
| **Dispositivo primário** | Smartphone (90%) |
| **Frequência de uso** | 2-4 vezes/dia, picos antes de atividades ao ar livre |
| **Objetivo principal** | "Planejar atividades ao ar livre (hiking, beach) sem surpresas climáticas" |
| **Contexto de uso** | Noites (planejamento), manhãs (confirmação), sem WiFi/4G confiável, dados caros (roaming) |
| **Pontos de dor** | Mudar cidade = buscar nova localização; sem histórico; dados internacionais caros; timezone confuso; sem alertas de chuva extrema |
| **Métrica de sucesso** | 🗺️ Multi-city (5-10 favoritas); 📡 Offline 12-24h; 🕐 Timezone claro; 📱 <2MB/consulta; 🎭 Sugestões de cidades |

### 📊 Comparação

| Aspecto | Marina | João | Sofia |
|---|---|---|---|
| **Frequência** | 1-3x/dia | 5-10x/dia | 2-4x/dia |
| **Duração** | <1 min | 5-10 min | 2-5 min |
| **Dispositivo** | Mobile 85% | Desktop/Tablet 60% | Mobile 90% |
| **Info crítica** | Temp, condição | Detalhes horários, gráficos | Multi-city, offline, alerts |
| **Métrica #1** | Velocidade (<2s) | Acurácia (<±1°C) | Offline + multi-city |
| **Métrica #2** | Confiança 15s | Histórico | Economia dados |

### 🎯 Implicações para Roadmap

| Phase | Foco | Features | Personas |
|---|---|---|---|
| **MVP (W1-2)** | Marina | <2s load, essencial (temp/condição/5d) | Marina (maior volume) |
| **Phase 2 (W3-6)** | João | Dados horários, gráficos, multi-city, SLA | João (retenção/diferencial) |
| **Phase 3 (W7+)** | Sofia | Offline 24h, histórico, timezone, alertas | Sofia (engagement/viagens) |

---

**Próxima etapa:** Transformar este discovery em uma **Especificação Técnica** detalhando critérios de aceite e cenários de teste.
