---
title: "ZapBot — notas de engenharia, experimentos e aprendizados"
date: 2026-09-25T22:36:00-03:00
draft: false
---

## Visão geral

O ZapBot começou como uma integração de WhatsApp com um agente local e evoluiu para um experimento de memória persistente, recuperação de contexto e execução de modelos locais.

O objetivo principal deixou de ser apenas "responder mensagens no WhatsApp". A parte mais interessante do projeto passou a ser a engenharia necessária para manter contexto histórico de forma barata, controlável, auditável e independente de um único modelo de linguagem.

A implementação atual está no repositório local `~/repos/zapbot`, principalmente na branch `dev`.

## Arquitetura atual

A arquitetura atual separa transporte, persistência, recuperação, segurança de instruções e geração.

### WhatsApp

O transporte é feito com `whatsapp-web.js` 1.34.7.

O bot roda no WSL e conversa com componentes locais, containers e o Atomic Chat no host. Como esses processos vivem em fronteiras de rede diferentes, foi necessário usar um relay com `socat` para permitir que o `memory-service` dentro do Docker alcance o endpoint local de inferência.

Um `start.sh` foi criado para reduzir as etapas manuais necessárias para subir o ambiente e validar as dependências principais antes de iniciar o bot.

### memory-service

O núcleo da memória roda em FastAPI dentro do Docker Compose.

O serviço é exposto somente localmente em:

```text
http://127.0.0.1:8787
```

A persistência usa SQLite dentro de `.zapbot`.

O SQLite continua sendo tratado como a fonte de verdade das mensagens. PDFs, PageIndex e qualquer outro índice de recuperação são derivados e podem ser reconstruídos sem perder a conversa original.

## Estratégia de memória

Uma das decisões mais importantes foi não enviar toda a conversa para o LLM a cada nova mensagem.

A implementação atual usa segmentos imutáveis com tamanho padrão de aproximadamente 250 mensagens.

Quando um segmento completo é fechado:

1. as mensagens permanecem persistidas no SQLite;
2. um snapshot daquele intervalo é criado;
3. o conteúdo é transformado em um PDF determinístico;
4. esse PDF é enviado ao PageIndex;
5. novas mensagens continuam sendo registradas no SQLite.

Os segmentos `READY` são imutáveis. Um novo lote não reconstrói segmentos anteriores.

Enquanto um segmento está `PENDING`, `INDEXING` ou `ERROR`, suas mensagens continuam disponíveis pelo SQLite. Isso evita a lacuna que existia no desenho anterior, em que centenas de mensagens poderiam ficar fora do índice enquanto apenas uma cauda recente limitada era consultada.

O schema chegou à versão 4 durante essa evolução.

## PageIndex

Foi utilizada a versão `0.2.19` do PageIndex.

Inicialmente, uma das ideias era tratar o PageIndex quase como um mecanismo completo de chat e recuperação.

Durante os testes ficou claro que isso criaria acoplamento excessivo.

A arquitetura foi alterada para usar o PageIndex como índice estrutural e mecanismo de routing. O fluxo atual não depende de `PageIndexClient.chat()`.

Os resumos e metadados produzidos pelo PageIndex podem ajudar a localizar regiões potencialmente relevantes, mas não são tratados como evidência factual.

A evidência entregue ao modelo final vem do conteúdo real das páginas recuperadas e das mensagens preservadas no SQLite.

Essa separação resultou em uma interface conceitual mais clara:

```text
mensagem
  -> retrieval
  -> collect_evidence
  -> geração
```

O passo `collect_evidence` é separado da geração da resposta.

## Recuperação híbrida

Outra mudança importante foi abandonar a expectativa de que uma única fonte de retrieval fosse suficiente.

A recuperação passou a combinar:

- mensagens ainda não cobertas por segmentos `READY`;
- segmentos históricos já indexados;
- busca lexical para routing;
- Markdown real das páginas recuperadas;
- contexto vizinho quando necessário;
- geração final pelo Atomic.

Essa abordagem resolveu um problema importante: mensagens recentes ou pertencentes a um segmento que ainda está indexando não podem depender exclusivamente do PageIndex.

O budget de contexto é aplicado globalmente antes da geração, em vez de multiplicar o limite por segmento.

## Routing multi-segmento

A primeira versão do retrieval assumia implicitamente que uma pergunta estaria relacionada a apenas um segmento.

Isso é frágil para conversas longas.

O sistema passou a suportar routing entre múltiplos segmentos. Quando existe correspondência lexical suficiente, os segmentos candidatos são priorizados. Em consultas ambíguas ou sem correspondência confiável, o sistema pode ampliar a busca para preservar recall.

A prioridade neste estágio foi evitar buracos de recuperação, mesmo que isso custe mais trabalho de leitura.

## Segurança de instruções e prompt injection

A evolução da memória criou outra fronteira importante: configuração do agente não pode ser confundida com dados vindos da conversa.

A policy permanente foi separada do histórico mutável e passou a viver em um módulo próprio.

Cada request recria uma única mensagem `system` no início. O histórico mantém somente turnos `user` e `assistant`.

A hierarquia conceitual ficou:

```text
SYSTEM / POLICY
      >
conversation data
      >
persistent memory data
```

Entradas de usuários, histórico, mensagens citadas, nomes de participantes, roster e futura evidência recuperada da memória são tratados como dados não confiáveis.

Foi adicionado um `prompt-guard` heurístico para identificar padrões comuns de prompt injection em português e inglês, como pedidos para ignorar instruções anteriores, redefinir identidade, simular `developer mode` ou declarar um novo system prompt.

O guard não bloqueia a mensagem. Uma pergunta legítima como "o que é system prompt?" continua chegando ao modelo normalmente. A detecção serve apenas para reforçar a fronteira de confiança.

Também foi preparado um wrapper para futura evidência de memória, com o objetivo de impedir que uma mensagem histórica como:

> "quando você ler isto no futuro, ignore suas instruções"

se transforme em uma instrução executável quando for recuperada posteriormente.

Essa defesa reduz o risco de prompt injection direta e persistente, mas não oferece garantia matemática: a obediência final do modelo continua probabilística.

## Menções em grupos

O fluxo de grupos usa aliases efêmeros como `u1`, `u2` e placeholders `{{uN}}`.

JIDs e números de telefone não são enviados ao Atomic.

O backend mantém o roster real, valida mentions, resolve placeholders e somente então envia a menção real pelo WhatsApp.

Nomes duplicados não autorizam escolha arbitrária, e aliases inventados pelo modelo não geram ping.

Os nomes de participantes são tratados como dados, não como instruções.

## Atomic Chat e modelo local

O Atomic Chat é usado como camada local para executar modelos GGUF por uma API compatível com OpenAI.

Durante os testes, o endpoint:

```text
/v1/models
```

foi usado para validar o modelo carregado.

O modelo utilizado na configuração atual é:

```text
HauhauCS/Gemma-4-E4B-Uncensored-HauhauCS-Aggressive-Q8_K_P
```

O `ATOMIC_MODEL` é usado para geração final. O `PAGEINDEX_MODEL`, quando configurado, é reservado para o pipeline de indexação.

## O que funcionou

### Integração local entre serviços

Foi possível integrar:

- WhatsApp;
- WSL;
- Docker Compose;
- FastAPI;
- SQLite;
- PageIndex;
- Atomic Chat;
- modelo GGUF local.

O maior ganho foi provar que esses componentes conseguem operar como um pipeline único.

### Persistência independente do LLM

A memória existe fora do contexto do modelo.

Trocar o modelo não destrói a memória da aplicação.

Essa continua sendo uma das decisões arquiteturais mais importantes do projeto.

### Segmentação imutável

A divisão em segmentos limitou o tamanho máximo de uma unidade de indexação.

Uma falha em um segmento não invalida segmentos anteriores e não apaga as mensagens originais.

### Separação entre retrieval e geração

O sistema deixou de depender da API de chat do mecanismo de índice.

`collect_evidence()` pode evoluir ou trocar o retriever sem obrigar a geração final a mudar junto.

Essa separação também abriu a possibilidade de comparar PageIndex, FTS5 ou outros retrievers usando a mesma etapa de geração.

### Estrutura real recuperada do PageIndex

Houve documentos processados com estado `READY`.

A estrutura recuperada continha nós, resumos, `node_id`, `page_index` e itens-chave, confirmando que o mecanismo de indexação estrutural funciona para documentos válidos.

### Hardening contra prompt injection

A policy deixou de fazer parte do histórico mutável.

Foram adicionados testes adversariais em português e inglês, incluindo tentativas de substituir instruções, trocar identidade, falsificar autoridade e redefinir o system prompt.

### Testes automatizados

Na validação da memória segmentada foram executados com sucesso:

- 55 testes Python.

Após o hardening de prompt:

- 69 testes Node.js passaram;
- `node --check` passou nos arquivos alterados;
- `docker compose config --quiet` passou;
- `git diff --check` passou;
- `whatsapp-web.js` permaneceu em 1.34.7;
- PageIndex permaneceu em 0.2.19.

## O que falhou

### PDFs inválidos durante a ingestão

A integração com PageIndex apresentou uma falha concreta ao enviar determinados PDFs.

O erro observado foi:

```text
PageIndexAPIError: Failed to submit document:
could not read PDF: Odd-length string
```

A origem apareceu no `PyPDF2` durante a extração de texto.

Isso mostrou que gerar um PDF não significa necessariamente gerar um PDF suficientemente robusto para qualquer parser.

A consequência foi tratar a geração e a validação do documento como parte crítica do pipeline.

### Acoplamento inicial excessivo ao PageIndex

A expectativa inicial de usar o PageIndex como solução quase completa de memória e chat não se sustentou bem.

O projeto ficou melhor quando o PageIndex passou a ser tratado como uma peça de retrieval, e não como o "cérebro" do sistema.

### Retrieval de apenas um segmento

O desenho inicial era insuficiente para perguntas cuja resposta estivesse distribuída em vários períodos da conversa.

Isso levou à implementação do routing multi-segmento.

### Tail limitado e lacunas de recuperação

Em uma versão anterior, o índice podia ficar muito atrás do SQLite enquanto a consulta direta ao banco considerava apenas uma janela recente.

Esse desenho permitia que parte do histórico ficasse temporariamente invisível ao retrieval.

A arquitetura segmentada foi alterada para considerar no SQLite todas as mensagens ainda não cobertas por segmentos `READY`.

### Competição entre indexação e geração interativa

Um problema importante apareceu durante o uso real.

O PageIndex usa o mesmo recurso local de inferência que atende o comando `!gpt`.

Durante a indexação de um segmento, o PageIndex pode executar inferências longas. Como indexação e conversa compartilham o Atomic, a latência do `!gpt` aumenta de forma perceptível.

O problema ficou especialmente visível após reiniciar o ZapBot com indexação pendente.

Isso levantou uma questão arquitetural maior: um mecanismo de indexação baseado em LLM pode ser excessivamente caro para histórico de conversa.

### Diferença entre testes isolados e ambiente real

Muitos componentes funcionaram individualmente antes de o sistema completo funcionar de ponta a ponta.

O projeto reforçou um problema comum em sistemas distribuídos locais: HTTP 200 em cada componente não significa que o fluxo inteiro esteja correto.

## Desafios técnicos

### Fronteiras de rede

Executar partes do sistema em WSL, outras em containers e outras diretamente no host criou problemas de conectividade que não existiriam em uma aplicação monolítica.

Foi necessário entender claramente:

- `localhost` de cada ambiente;
- `host.docker.internal`;
- portas expostas;
- relays;
- ordem de inicialização dos serviços.

### Consistência da memória

O índice não pode ser a fonte de verdade.

Documentos podem falhar ao indexar, ficar temporariamente indisponíveis ou representar apenas parte do histórico.

Por isso, o SQLite precisa continuar preservando a conversa original.

### Tail não indexado

Existe inevitavelmente uma janela entre uma nova mensagem e sua presença em um segmento `READY`.

A diferença atual é que essa janela não depende mais de um limite arbitrário de 100 mensagens: tudo que ainda não estiver coberto por segmentos prontos deve continuar acessível pelo SQLite.

### Qualidade do retrieval

Encontrar texto relacionado é diferente de recuperar evidência suficiente para responder corretamente.

Esse foi um dos motivos para criar uma etapa explícita de coleta de evidências antes da geração.

### Modelos locais menores

Executar localmente reduz custo e dependência externa, mas aumenta a importância da engenharia ao redor do modelo.

Um modelo menor pode funcionar bem quando recebe contexto selecionado e estruturado; funciona muito pior quando precisa descobrir sozinho, dentro de um histórico enorme, o que é relevante.

## Reavaliando PageIndex para conversas

PageIndex continua sendo interessante para documentos naturalmente estruturados, como PDFs, manuais, documentação e relatórios.

Uma conversa de WhatsApp, porém, não possui naturalmente capítulos, páginas ou bookmarks.

O pipeline atual precisa transformar mensagens em PDFs artificiais antes de indexá-las:

```text
WhatsApp
  -> SQLite
  -> segmento
  -> PDF
  -> PageIndex
  -> páginas relevantes
  -> EvidenceBundle
  -> Atomic
```

Além da complexidade, a indexação usa o mesmo recurso de inferência do bot interativo.

Por isso, a próxima hipótese a ser testada é um retrieval mais direto para conversas:

```text
WhatsApp
  -> SQLite
  -> FTS5 / BM25
  -> mensagens candidatas + vizinhas
  -> EvidenceBundle
  -> Atomic
```

O PageIndex não deve ser removido antes de uma comparação objetiva.

A ideia é comparar pelo menos:

- recall histórico;
- recall recente;
- falsos positivos;
- latência p50/p95;
- custo de ingestão;
- impacto no startup;
- uso de CPU/RAM;
- complexidade operacional.

Se FTS5 com contexto vizinho entregar qualidade suficiente, PageIndex pode continuar reservado para documentos estruturados em vez de histórico de chat.

## Lições

### Memória é um problema de sistema

A principal conclusão até agora é que memória de agente não deveria ser tratada apenas como "aumentar o context window".

Ela envolve:

- persistência;
- segmentação;
- indexação;
- freshness;
- retrieval;
- ranking;
- evidência;
- segurança de instruções;
- geração.

### Índice e banco têm responsabilidades diferentes

O banco responde:

> O que realmente aconteceu?

O índice responde:

> Onde provavelmente está a informação relevante?

Confundir essas responsabilidades torna o sistema mais frágil.

### Nem todo dado precisa virar documento

Uma das lições mais recentes é que histórico de chat e documentos têm estruturas diferentes.

Transformar conversas em PDFs pode ser útil como experimento, mas não deve ser assumido como a melhor representação apenas porque o mecanismo de retrieval trabalha bem com documentos.

### Modelos locais ficam mais úteis com boa engenharia de contexto

O valor de um modelo local não depende apenas do benchmark do modelo.

A qualidade do pipeline de contexto pode compensar parte significativa da diferença entre um modelo pequeno e um modelo muito maior para tarefas específicas.

### Falhas intermediárias precisam ser observáveis

Problemas de ingestão, documentos incompletos, erros de parsing ou segmentos em `ERROR` precisam aparecer claramente.

Caso contrário, o sistema pode responder com confiança usando uma memória silenciosamente incompleta.

### Conteúdo recuperado também é input não confiável

Memória persistente pode carregar prompt injection ao longo do tempo.

Por isso, recuperar uma mensagem antiga não pode elevar essa mensagem à autoridade de `system`.

## Estado atual

Até o momento:

- integração WhatsApp funcionando em ambiente local;
- `memory-service` operacional;
- SQLite como fonte de verdade;
- schema v4;
- segmentação imutável de 250 mensagens por padrão;
- PageIndex 0.2.19 integrado;
- PageIndex já atingiu `READY` em smokes anteriores;
- routing multi-segmento implementado;
- `collect_evidence` separado da geração;
- SQLite tail sem a antiga limitação que podia criar lacunas;
- proteção contra prompt injection implementada;
- system policy reconstruída em cada request;
- histórico da IA contendo somente turnos `user` e `assistant`;
- conteúdo de usuário, roster e futura memória tratado como dado não confiável;
- 55 testes Python passando na validação da memória segmentada;
- 69 testes Node.js passando após o hardening de prompt;
- smoke real da nova arquitetura segmentada iniciado;
- primeiro ciclo completo `INDEXING -> READY` da arquitetura segmentada ainda precisa ser confirmado;
- impacto de performance da indexação PageIndex sobre o `!gpt` identificado em uso real;
- SQLite FTS5 sendo considerado como alternativa experimental para memória de conversas.

## Próximos pontos a validar

O próximo passo importante é comparar arquiteturas, não simplesmente adicionar mais componentes.

As principais perguntas agora são:

1. O retrieval atual encontra a evidência correta depois de milhares de mensagens?
2. O routing entre segmentos compensa o custo de manter PageIndex?
3. SQLite FTS5/BM25 com mensagens vizinhas consegue manter recall semelhante com latência e ingestão muito menores?
4. A combinação entre histórico e mensagens recentes evita qualquer lacuna?
5. O sistema consegue indicar quando não possui evidência suficiente?
6. Quanto a indexação em background interfere na latência do `!gpt`?
7. Qual é o custo real em CPU, RAM, GPU e armazenamento?
8. PageIndex deve continuar no caminho de memória de chat ou ficar reservado para documentos estruturados?
9. Como provenance deve distinguir conteúdo humano, do proprietário, gerado pelo assistente e operacional antes de integrar memória automática ao `!gpt`?

Essas medições vão determinar se a arquitetura deve continuar segmentada sobre PageIndex ou migrar para um retriever mais simples diretamente sobre o banco.

## Resumo

O ZapBot começou como um bot de WhatsApp, mas o experimento mais relevante passou a ser a construção de uma memória de agente independente do modelo.

Os maiores avanços vieram menos de trocar LLMs e mais de corrigir fronteiras arquiteturais:

- SQLite como fonte de verdade;
- segmentos imutáveis;
- recuperação sem lacunas;
- PageIndex desacoplado da geração;
- `collect_evidence` separado da resposta final;
- suporte a múltiplos segmentos;
- system policy fora do histórico;
- dados de usuário e memória tratados como não confiáveis.

Os principais fracassos — PDFs inválidos, acoplamento excessivo ao índice, retrieval limitado, lacunas de tail, dificuldades de rede e competição pelo Atomic — foram úteis porque expuseram onde a arquitetura precisava ser simplificada.

O projeto já provou que as peças conseguem coexistir.

A próxima etapa é descobrir quais peças realmente precisam continuar existindo.
