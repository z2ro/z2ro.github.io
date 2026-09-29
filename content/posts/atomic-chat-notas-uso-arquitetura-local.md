---
title: "Atomic Chat — notas de uso, arquitetura local e aprendizados"
date: 2026-09-28T21:57:00-03:00
draft: false
---

## Visão geral

O Atomic Chat entrou no meu ambiente local de IA inicialmente como uma forma prática de executar modelos GGUF e conversar com eles.

Com o tempo, ele passou a ocupar um papel mais importante: uma camada local de inferência que pode servir modelos por API, receber contexto já selecionado por outros componentes, participar de pipelines de memória e agir como ponto de entrada para ferramentas e agentes mais especializados.

A principal mudança de percepção foi esta:

> o Atomic não precisa ser o modelo mais inteligente do sistema para ser uma peça útil da arquitetura.

Na prática, comecei a tratá-lo menos como um "ChatGPT local" isolado e mais como um runtime para uma arquitetura híbrida.

O desenho que faz mais sentido hoje é:

```text
usuário
  -> Atomic Chat
  -> contexto externo / retrieval / tools
  -> modelo local
  -> decisão de escalonamento
       -> resposta local
       -> Codex / agente especializado
       -> modelo cloud, quando necessário
```

O objetivo não é obrigar um único modelo a fazer tudo.

É usar o componente mais barato e suficientemente confiável para cada etapa.

## Ambiente local

O ambiente em que venho testando o Atomic inclui:

- notebook com i9-13900HX;
- RTX 4070 Laptop com 8 GB de VRAM;
- 32 GB de RAM;
- Windows 11;
- WSL;
- modelos GGUF;
- llama.cpp / llama-server;
- Ollama em alguns experimentos;
- Codex CLI;
- PageIndex;
- serviços FastAPI;
- Docker Compose;
- memória persistente;
- retrieval lexical e estrutural.

O Atomic expõe uma API compatível com o formato da OpenAI, o que permite integrar a inferência local sem obrigar cada projeto a conhecer detalhes do backend que está executando o modelo.

Em experimentos anteriores, usei a API local em:

```text
http://localhost:1337/v1
```

Isso foi importante porque transformou o Atomic em infraestrutura reutilizável.

Um projeto pode simplesmente apontar para o endpoint local e trocar o modelo carregado sem reconstruir todo o restante da aplicação.

## Modelos que testei

Uma parte considerável dos experimentos foi entender quanto modelo realmente cabe no hardware e qual é o custo prático de contexto, quantização e offload.

Entre os testes estiveram variantes de Qwen e Gemma.

Em uma configuração com Qwen3 Hermes 8B em `Q4_K_M`, configurei contexto de 65.536 tokens, mas o modelo reportava contexto nativo menor. O comportamento também foi mais lento do que o Gemma que eu estava comparando no mesmo ambiente.

Isso reforçou uma coisa que parece óbvia, mas é fácil esquecer:

> tamanho nominal do modelo não determina sozinho a experiência de uso.

Entram na conta:

- arquitetura do modelo;
- quantização;
- quantidade de pesos na VRAM;
- offload para RAM;
- KV cache;
- tamanho do contexto;
- quantidade de tokens de raciocínio;
- velocidade de CPU;
- velocidade da memória;
- backend utilizado.

Em outra configuração mais recente, o Atomic estava executando um Gemma 4 12B quantizado em GGUF por meio do `llama-server`.

O ponto mais importante desses testes não foi descobrir "o melhor modelo".

Foi descobrir qual modelo é suficientemente bom para determinada função.

## Quantização: o ganho é capacidade, não milagre

O Atomic me levou a explorar mais seriamente GGUF e quantizações como `Q4_K_M`, `Q4_K_S` e variantes maiores.

A quantização reduz bastante o custo de memória e pode tornar modelos viáveis em hardware que não conseguiria executar os pesos em precisão maior.

Mas ela não elimina os demais custos.

Mesmo quando os pesos cabem, ainda existem:

- KV cache;
- contexto longo;
- buffers;
- CPU/RAM offload;
- transferência entre RAM e VRAM;
- latência por token.

Isso mudou a forma como avalio modelos locais.

Hoje eu prefiro perguntar:

> qual é o menor modelo que consegue executar esta tarefa com confiabilidade aceitável?

em vez de:

> qual é o maior modelo que eu consigo fazer abrir?

Essas duas perguntas produzem arquiteturas muito diferentes.

## Contexto longo não substitui memória

Outro aprendizado importante foi separar context window de memória persistente.

No começo é tentador pensar que um modelo com 32K, 64K ou mais contexto resolve o problema de memória.

Não resolve.

Contexto grande ajuda, mas ainda é necessário decidir:

- o que deve entrar;
- o que deve ficar de fora;
- o que é evidência;
- o que é apenas resumo;
- o que é recente;
- o que é histórico;
- o que pode ser recuperado sob demanda.

Além disso, contextos maiores aumentam custo de KV cache e podem degradar velocidade.

A arquitetura que passei a perseguir é:

```text
memória persistente
  -> retrieval
  -> seleção de evidência
  -> budget de contexto
  -> Atomic
```

O modelo recebe apenas o contexto que precisa para responder.

## System prompt: menos é melhor

Também explorei até onde fazia sentido colocar perfil, regras, histórico e instruções permanentes diretamente no system prompt de um Assistant do Atomic.

A conclusão foi que transformar o system prompt em um banco de dados é uma má ideia.

Mesmo quando não existe um limite pequeno explícito de caracteres, todo esse conteúdo compete pela mesma janela de contexto usada pela conversa e pelas evidências recuperadas.

Hoje considero uma arquitetura melhor:

```text
system prompt curto
  + perfil estável
  + memória externa
  + retrieval
  + tools
```

O system prompt deve definir principalmente:

- identidade;
- prioridades;
- fronteiras de segurança;
- forma de raciocínio;
- regras permanentes.

Informações extensas ou mutáveis devem ficar fora dele.

## Atomic no ZapBot

O projeto em que o Atomic mais deixou de ser apenas interface e virou infraestrutura foi o ZapBot.

O pipeline evoluiu para algo próximo de:

```text
WhatsApp
  -> persistência em SQLite
  -> retrieval
  -> coleta de evidência
  -> budget de contexto
  -> Atomic
  -> resposta
```

Nesse desenho, o Atomic não precisa guardar a memória da conversa.

A memória continua existindo mesmo que eu troque completamente o modelo local.

Essa separação foi essencial.

Ela permite testar:

- Gemma;
- Qwen;
- outro GGUF;
- outro backend;

sem destruir o histórico ou reindexar tudo apenas porque o modelo mudou.

## PageIndex

O PageIndex entrou nos experimentos como uma possível camada de memória e recuperação.

Ele funcionou bem o suficiente para provar que documentos estruturados podem ser indexados, resumidos e roteados.

O problema apareceu quando tentei aplicar a mesma lógica a histórico de conversa.

Conversa não nasce como documento.

Para usar o mesmo pipeline, foi necessário transformar mensagens em algo parecido com:

```text
mensagens
  -> segmento
  -> PDF
  -> PageIndex
  -> páginas
  -> evidência
```

Isso funciona, mas cria uma quantidade grande de infraestrutura intermediária.

Também apareceu um problema operacional: indexação e resposta interativa podiam disputar o mesmo recurso de inferência local no Atomic.

Durante uma indexação pesada, a latência da conversa aumentava.

Esse foi um dos sinais de que um retriever mais simples, como FTS5/BM25 com mensagens vizinhas, pode ser melhor para conversa, enquanto PageIndex continua fazendo mais sentido para PDFs, documentação e relatórios.

## Laya

Também avaliei onde o Laya poderia entrar nessa arquitetura.

A principal distinção que passei a fazer é entre:

- ferramenta que possui API ou protocolo estruturado;
- aplicação que só pode ser controlada pela interface gráfica.

Sempre que existe API, shell, MCP ou outra interface estruturada, prefiro esse caminho.

Automação visual é útil quando não existe uma interface melhor, mas não deveria virar a base de uma infraestrutura que pode ser controlada deterministicamente.

Por isso, vejo Laya mais como uma ferramenta complementar do que como o mecanismo principal de orquestração.

## Atomic + Codex

Uma das ideias que mais me interessaram foi usar o modelo local antes de recorrer a um agente mais caro e mais capaz.

A divisão de responsabilidades que faz sentido é aproximadamente:

### Modelo local

Usar para:

- classificação;
- extração;
- sumarização;
- compressão de contexto;
- routing;
- conversa simples;
- transformação de texto;
- seleção preliminar de ferramentas.

### Codex ou agente especializado

Usar para:

- investigação de código;
- mudanças em repositórios;
- raciocínio técnico mais difícil;
- execução de tarefas multi-etapas;
- validação;
- debugging;
- operações que exigem maior confiabilidade.

O Atomic, nesse desenho, pode funcionar como front-end e dispatcher.

Ele não precisa vencer o Codex em raciocínio.

Precisa saber quando uma tarefa não deveria terminar nele.

## Januz e o handoff local

Essa ideia chegou a ser testada em um fluxo de controle local.

O objetivo era permitir que uma conversa disparasse uma tarefa para o Codex dentro do WSL e depois trouxesse o resultado de volta.

Uma tentativa de chamar diretamente o WSL pelo broker encontrou erro de permissão.

A solução que funcionou foi desacoplar os lados com arquivos de handoff:

```text
Atomic / navegador
  -> broker
  -> arquivos de handoff
  -> watcher no WSL
  -> Codex
  -> resultado
  -> broker
  -> chat
```

Foram usados estados como:

```text
READY
RUNNING
DONE
```

Esse experimento foi importante porque mostrou que o modelo local não precisa ter acesso irrestrito ao sistema inteiro.

O controle pode ser mediado por contratos explícitos.

## O que funcionou

### API local reutilizável

A API compatível com OpenAI permitiu encaixar o Atomic em aplicações sem criar uma integração proprietária para cada projeto.

### Troca de modelos sem trocar a aplicação

Manter memória, retrieval e regras fora do modelo tornou possível experimentar diferentes GGUF sem reescrever o restante do sistema.

### Modelos locais como camada de triagem

Para tarefas simples, não existe motivo para começar sempre pelo componente mais caro do pipeline.

O modelo local consegue assumir boa parte do trabalho mecânico.

### Integração com memória externa

O ZapBot provou que o Atomic pode gerar a resposta final usando evidências recuperadas de um sistema independente do LLM.

### Handoff para agentes mais fortes

O experimento do Januz mostrou um caminho viável para deixar o chat como interface enquanto tarefas mais difíceis são delegadas a outro executor.

## O que não funcionou tão bem

### Tentar resolver tudo com contexto

Aumentar o context window não elimina o problema de selecionar informação.

Em alguns casos apenas aumenta custo e latência.

### Usar um modelo local pequeno como se fosse um modelo cloud grande

O resultado piora quando o modelo precisa:

- descobrir sozinho todo o contexto relevante;
- manter muitas instruções concorrentes;
- executar raciocínio longo;
- escolher entre muitas ferramentas;
- preservar precisão em tarefas extensas.

A solução não é necessariamente um modelo maior.

Muitas vezes é reduzir a responsabilidade do modelo.

### Compartilhar inferência entre indexação e chat

Quando PageIndex e conversa usam o mesmo recurso local, tarefas de ingestão podem degradar a experiência interativa.

Isso sugere separar workers, limitar concorrência ou usar mecanismos de retrieval que não dependam de LLM para cada ingestão.

### Tratar resumo como evidência

Resumo é útil para routing.

Não deveria substituir o conteúdo original quando precisão factual é importante.

Essa diferença passou a orientar os pipelines de memória.

### Automação visual quando existe interface estruturada

Controlar GUI é mais frágil do que usar API, MCP, shell ou protocolos explícitos.

Laya pode ser útil, mas não deve substituir interfaces melhores apenas por ser mais visual.

## Principais aprendizados

### 1. O modelo não é o sistema

O resultado depende tanto de retrieval, memória, ferramentas e contratos quanto do LLM escolhido.

### 2. Capacidade local deve ser usada com propósito

Fazer um modelo enorme caber é tecnicamente interessante.

Mas um 8B ou 12B bem alimentado pode ser mais útil do que um modelo muito maior operando lentamente e recebendo contexto ruim.

### 3. Memória precisa existir fora do contexto

Histórico importante deve sobreviver a:

- reinicialização;
- troca de modelo;
- troca de backend;
- redução de context window.

### 4. Retrieval deve preservar evidência

Índices, embeddings e resumos ajudam a encontrar informação.

A resposta final deveria, sempre que possível, ser baseada no conteúdo original recuperado.

### 5. Escalonamento é parte da inteligência do sistema

Uma arquitetura local eficiente não precisa resolver tudo localmente.

Ela precisa identificar corretamente quando vale a pena escalar.

### 6. A melhor arquitetura é híbrida

O desenho que mais faz sentido para mim hoje é:

```text
determinístico
  -> retrieval estruturado
  -> modelo local
  -> agente especializado
  -> cloud forte
```

Cada etapa só entra quando a anterior não é suficiente.

## Estado atual

Hoje eu vejo o Atomic Chat principalmente como uma plataforma para construir uma camada local de IA ao redor dos meus projetos.

Ele pode servir de:

- interface conversacional;
- servidor local de inferência;
- camada compatível com OpenAI;
- executor de modelos GGUF;
- ponto de entrada para memória e retrieval;
- dispatcher para ferramentas;
- dispatcher para agentes mais fortes.

O valor não está em tentar substituir todas as ferramentas por um único modelo local.

Está em conseguir manter uma parte cada vez maior do pipeline sob meu controle e escalar apenas o que realmente precisa de mais capacidade.

## Próximos experimentos

Os próximos pontos que considero mais importantes são:

- medir objetivamente qualidade e latência entre modelos locais;
- comparar contexto grande com retrieval mais agressivo;
- separar inferência interativa de workloads de indexação;
- testar melhor FTS5/BM25 contra PageIndex para histórico de conversa;
- amadurecer memória persistente fora do system prompt;
- melhorar o routing entre modelo local e Codex;
- medir quanto uso de cloud realmente pode ser evitado sem reduzir qualidade;
- manter contratos explícitos entre chat, tools e executores.

O experimento com Atomic começou como uma pergunta sobre rodar modelos locais.

Hoje a pergunta é outra:

> até onde consigo construir uma infraestrutura de IA local útil sem obrigar o modelo local a ser responsável por tudo?
