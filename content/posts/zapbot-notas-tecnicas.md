---
title: "ZapBot — notas de engenharia, experimentos e aprendizados"
date: 2026-09-25T22:36:00-03:00
draft: false
---

## Visão geral

O ZapBot começou como uma integração de WhatsApp com um agente local e evoluiu para um experimento de memória persistente e recuperação de contexto usando modelos locais, PageIndex e um serviço próprio de memória.

O objetivo principal deixou de ser apenas "responder mensagens no WhatsApp". A parte mais interessante do projeto passou a ser a engenharia necessária para manter contexto histórico de forma barata, controlável e independente de um único modelo de linguagem.

A implementação atual está no repositório local `~/repos/zapbot`, principalmente na branch `dev`.

## Arquitetura atual

A arquitetura atual separa transporte, memória, recuperação e geração.

### WhatsApp

O transporte é feito com `whatsapp-web.js` 1.34.7.

O WhatsApp roda no WSL e conversa com os demais componentes locais. Para facilitar a comunicação entre serviços e processos distribuídos entre host, WSL e containers, também foi necessário utilizar relay de rede com `socat`.

Um `start.sh` foi criado para reduzir o número de etapas manuais necessárias para subir o ambiente.

### memory-service

O núcleo de memória roda em um serviço FastAPI dentro do Docker Compose.

O serviço foi exposto localmente em:

```text
http://0.0.0.0:8787
```

A persistência começou em SQLite, armazenado dentro de `.zapbot`.

Durante a evolução do projeto também começou uma migração do armazenamento relacional para PostgreSQL 17.

O banco relacional continua sendo tratado como a fonte de verdade das mensagens. O índice de recuperação não substitui o histórico bruto.

## Estratégia de memória

Uma das decisões mais importantes foi não enviar toda a conversa para o LLM a cada nova mensagem.

Em vez disso, o histórico é segmentado.

A implementação atual usa segmentos imutáveis com aproximadamente 250 mensagens.

Quando um segmento é fechado:

1. as mensagens permanecem persistidas no banco;
2. o conteúdo é transformado em um documento;
3. o documento é enviado para indexação;
4. novas mensagens continuam sendo registradas no segmento ativo.

Isso permite separar memória histórica indexada de mensagens recentes ainda não indexadas.

O schema chegou à versão 4 durante essa evolução.

## PageIndex

Foi utilizada a versão `0.2.19` do PageIndex.

Inicialmente, uma das ideias era tratar o PageIndex quase como um mecanismo completo de chat e recuperação.

Durante os testes ficou claro que isso criaria acoplamento excessivo.

A arquitetura foi alterada para utilizar o PageIndex principalmente como índice estrutural e fonte de evidências.

O fluxo atual não depende de `PageIndexClient.chat()`.

A aplicação executa a recuperação e depois entrega as evidências para o pipeline de geração.

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

- conteúdo recente ainda presente no banco;
- segmentos históricos indexados;
- busca lexical;
- Markdown/conteúdo estrutural recuperado;
- consulta ao modelo local quando necessário.

Essa abordagem resolveu um problema importante: mensagens recentes ainda não pertencem necessariamente a um documento PageIndex pronto, portanto não podem depender exclusivamente do índice.

## Routing multi-segmento

A primeira versão do retrieval assumia implicitamente que uma pergunta estaria relacionada a apenas um segmento.

Isso é frágil para conversas longas.

O sistema passou a suportar routing entre múltiplos segmentos, permitindo que uma consulta reúna evidências espalhadas por diferentes períodos da conversa.

Esse ponto foi especialmente importante para perguntas sobre decisões tomadas dias antes ou assuntos que reaparecem ao longo do histórico.

## Atomic Chat e modelo local

O Atomic Chat foi usado como camada local para executar modelos GGUF.

Durante os testes, a API local respondeu corretamente no endpoint compatível com OpenAI:

```text
/v1/models
```

Um dos modelos utilizados foi:

```text
mradermacher/gemma-4-12B-it-abliterated-uncensored_Q4_K_S
```

O objetivo era manter tarefas simples e recuperação local fora de APIs pagas sempre que possível, deixando modelos mais fortes para casos em que a qualidade adicional realmente justificasse o custo.

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

A memória passou a existir fora do contexto do modelo.

Esse é provavelmente o resultado arquitetural mais importante do experimento.

Trocar o modelo não destrói a memória da aplicação.

### Segmentação

A divisão em segmentos imutáveis reduziu o problema de crescimento indefinido do histórico e criou uma unidade natural para indexação.

### Separação entre retrieval e geração

O sistema deixou de depender de uma API de chat do mecanismo de índice.

Isso reduziu acoplamento e tornou mais fácil testar cada etapa isoladamente.

### Estrutura real recuperada do PageIndex

Houve documentos processados com estado `READY`.

A estrutura recuperada continha nós, resumos, `node_id`, `page_index` e itens-chave, confirmando que o mecanismo de indexação estrutural funcionava para documentos válidos.

### Testes automatizados

Na etapa atual foram executados com sucesso:

- 55 testes Python;
- 65 testes Node.js.

Isso trouxe uma base razoável para continuar alterando o pipeline sem depender somente de testes manuais.

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

A consequência foi adicionar validação e tratar a geração do documento como parte crítica do pipeline, não apenas como uma etapa de serialização.

### Acoplamento inicial excessivo ao PageIndex

A expectativa inicial de usar o PageIndex como solução quase completa de memória e chat não se sustentou bem.

O projeto ficou melhor quando o PageIndex passou a ser tratado como uma peça de retrieval, e não como o "cérebro" do sistema.

### Retrieval de apenas um segmento

O desenho inicial também era insuficiente para perguntas cuja resposta estivesse distribuída em vários períodos da conversa.

Isso levou à implementação do routing multi-segmento.

### Diferença entre testes isolados e ambiente real

Muitos componentes funcionaram individualmente antes de o sistema completo funcionar de ponta a ponta.

O projeto reforçou um problema comum em sistemas distribuídos locais: HTTP 200 em cada componente não significa que o fluxo inteiro esteja correto.

No estado mais recente, os testes automatizados estavam verdes, mas o smoke test real completo do fluxo ainda estava marcado como `NOT_RUN`.

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

Por isso, o armazenamento relacional precisa continuar preservando a conversa original.

### Tail não indexado

Existe inevitavelmente uma janela entre uma nova mensagem e sua presença em um segmento fechado e indexado.

O sistema precisou manter uma estratégia explícita para consultar essa "cauda" recente diretamente no banco.

### Qualidade do retrieval

Encontrar texto relacionado é diferente de recuperar evidência suficiente para responder corretamente.

Esse foi um dos motivos para criar uma etapa explícita de coleta de evidências antes da geração.

### Modelos locais menores

Executar localmente reduz custo e dependência externa, mas aumenta a importância da engenharia ao redor do modelo.

Um modelo menor pode funcionar bem quando recebe contexto selecionado e estruturado; funciona muito pior quando precisa descobrir sozinho, dentro de um histórico enorme, o que é relevante.

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
- geração.

### Índice e banco têm responsabilidades diferentes

O banco responde:

> O que realmente aconteceu?

O índice responde:

> Onde provavelmente está a informação relevante?

Confundir essas responsabilidades torna o sistema mais frágil.

### Modelos locais ficam mais úteis com boa engenharia de contexto

O experimento reforçou que o valor de um modelo local não depende apenas do benchmark do modelo.

A qualidade do pipeline de contexto pode compensar parte significativa da diferença entre um modelo pequeno e um modelo muito maior para tarefas específicas.

### Falhas intermediárias precisam ser observáveis

Problemas de ingestão, documentos incompletos ou erros de parsing precisam aparecer claramente.

Caso contrário, o sistema pode responder com confiança usando uma memória silenciosamente incompleta.

## Estado atual

Até o momento:

- integração WhatsApp funcionando em ambiente local;
- `memory-service` operacional;
- persistência relacional implementada;
- segmentação de aproximadamente 250 mensagens implementada;
- schema v4;
- PageIndex 0.2.19 integrado;
- documentos válidos chegando a `READY`;
- routing multi-segmento implementado;
- `collect_evidence` separado da geração;
- retrieval híbrido implementado;
- 55 testes Python passando;
- 65 testes Node.js passando;
- smoke test real completo ainda pendente.

## Próximos pontos a validar

O próximo passo importante não é adicionar mais componentes.

É provar que o pipeline atual funciona de ponta a ponta em conversas reais e longas.

As principais perguntas são:

1. O retrieval encontra a evidência correta depois de milhares de mensagens?
2. O routing escolhe os segmentos certos?
3. A combinação entre histórico indexado e tail recente evita lacunas?
4. O sistema consegue indicar quando não possui evidência suficiente?
5. O modelo local responde melhor com esse pipeline do que receberia apenas uma janela grande de mensagens?
6. Qual é a latência real por mensagem?
7. Quanto custa, em CPU, RAM e armazenamento, manter essa memória ao longo do tempo?

Essas medições vão determinar se a arquitetura é apenas tecnicamente interessante ou realmente útil em produção.

## Resumo

O ZapBot começou como um bot de WhatsApp, mas o experimento mais relevante passou a ser a construção de uma memória de agente independente do modelo.

Os maiores avanços vieram menos de trocar LLMs e mais de corrigir a arquitetura:

- banco como fonte de verdade;
- segmentos imutáveis;
- PageIndex como índice;
- recuperação híbrida;
- evidências separadas da geração;
- suporte a múltiplos segmentos.

Os principais fracassos — PDFs inválidos, acoplamento excessivo ao índice, retrieval limitado e dificuldades de rede — foram úteis porque expuseram onde a arquitetura precisava ser simplificada.

O projeto já provou que as peças conseguem coexistir. O próximo estágio é provar qualidade de recuperação, confiabilidade e custo no uso real.
