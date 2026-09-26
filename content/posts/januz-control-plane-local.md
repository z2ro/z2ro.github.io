---
title: "Januz — construindo um control plane local entre ChatGPT, WSL, Codex, Git, Docker e Browser QA"
date: 2026-09-26T00:05:00-03:00
draft: false
layout: "paper"
paper_title: "JANUZ"
subtitle: "Construindo um control plane local entre ChatGPT, WSL, Codex, Git, Docker e Browser QA"
series: "2026 Field Guide to Local AI Systems"
edition: "September 2026"
note: "Engineering field note baseado em uma implementação real de automação local."
abstract: >-
  Januz é um control plane local que conecta uma conversa no ChatGPT a executores restritos no computador. A arquitetura separa contexto e decisão de execução: a extensão identifica comandos estruturados, um broker em loopback faz o roteamento e workers específicos cuidam de Codex, Git, Docker e Browser QA. Este artigo descreve por que o sistema foi criado, quais partes funcionaram, onde a primeira arquitetura falhou e quais princípios de segurança, idempotência e observabilidade passaram a orientar a evolução do projeto.
index_terms:
  - control plane
  - local AI
  - Codex
  - WSL
  - Git
  - Docker
  - Playwright
  - idempotência
---

## Visão geral

O Januz nasceu de uma necessidade prática: reduzir o atrito entre raciocinar sobre uma mudança de software e realmente executar essa mudança em uma máquina local.

No fluxo tradicional, eu descrevia uma tarefa em um chat, copiava um prompt para o terminal, acompanhava o Codex, voltava para o Git, rodava testes, abria o navegador, tirava screenshots, conferia o resultado e então retornava ao chat com o estado atualizado.

Cada etapa isoladamente era simples.

O problema era a quantidade de trocas de contexto.

O Januz foi criado para transformar essa sequência em um fluxo controlado:

```text
ChatGPT
  ↓
Chrome Extension
  ↓
Januz Broker
  ↓
Executors / Handoff
  ├─ Codex
  ├─ Git
  ├─ Docker
  └─ Browser QA
```

A ideia central é que o chat funcione como **control plane**, enquanto a máquina local continua responsável pela execução real.

O projeto está em:

[github.com/z2ro/januz](https://github.com/z2ro/januz)

Na versão atual, broker e extensão estão alinhados em `0.2.4`, mantendo `bridge_protocol = 1`.

{{< januz-control-loop >}}

---

## O problema que eu queria resolver

Ferramentas como Codex CLI conseguem editar código, executar testes e investigar um repositório.

Mas um agente local não possui necessariamente todo o contexto arquitetural, de produto e de decisão que está presente em uma conversa longa.

Ao mesmo tempo, deixar o chat executar qualquer comando arbitrário na máquina seria uma ideia ruim.

O problema passou a ser:

> Como usar um modelo com mais contexto para planejar e coordenar trabalho local sem transformar isso em execução remota irrestrita?

A resposta foi criar uma fronteira explícita.

O chat não recebe um terminal genérico.

Ele emite comandos dentro de um protocolo limitado.

Exemplo conceitual:

```text
# LOCAL COMMAND

{
  "bridge_protocol": 1,
  "id": "job-001",
  "action": "browser_qa",
  "payload": {
    ...
  }
}
```

A extensão detecta o comando, envia ao broker e recebe um resultado estruturado.

Depois o resultado volta para a mesma conversa:

```text
# LOCAL RESULT

{
  "bridge_protocol": 1,
  "command_id": "job-001",
  "status": "success",
  ...
}
```

Isso cria uma interface pequena e auditável entre o modelo e a máquina.

---

## Arquitetura

Hoje o Januz possui quatro blocos principais.

### Chrome Extension

A extensão observa a conversa de controle no ChatGPT.

Ela é responsável por:

- localizar `LOCAL COMMAND`;
- validar o protocolo;
- encaminhar comandos ao broker local;
- acompanhar operações assíncronas;
- buscar artifacts;
- anexar screenshots;
- publicar `LOCAL RESULT` de volta na conversa.

Na primeira versão, o ID da conversa estava hardcoded.

Isso foi removido.

Agora a extensão possui uma Options Page e usa `chrome.storage.local` para guardar:

- Control Chat ID;
- Broker URL.

O broker padrão é:

```text
http://127.0.0.1:8765
```

A configuração aceita apenas loopback, usando `127.0.0.1` ou `localhost`.

### Januz Broker

O broker principal é Node.js.

Ele escuta em:

```text
127.0.0.1:8765
```

O bind explícito em loopback é intencional.

O broker não deve ser um serviço aberto na LAN ou na internet.

Entre as actions atuais estão:

- `codex_exec`;
- `codex_status`;
- `git_finalize`;
- `git_status`;
- `wsl_docker_test`;
- `wsl_docker_status`;
- `db_connectivity_probe`;
- `browser_qa`;
- `ping`.

Não existe um `shell_exec` genérico.

Essa ausência é uma feature de segurança.

### WSL Watchers

Algumas operações atravessam a fronteira Windows → WSL por arquivos de handoff.

Os principais componentes são:

```text
codex-bridge.sh
git-finalize-bridge.sh
git-finalize-request.py
docker-test-bridge.sh
```

O runtime usa diretórios como:

```text
C:\AI\handoff
C:\AI\browser-artifacts
```

e, no WSL:

```text
/mnt/c/AI/handoff
/mnt/c/AI/browser-artifacts
```

Esses diretórios são estado operacional e não fazem parte do Git.

### Browser QA

O Januz também usa Playwright para validar interfaces web.

Isso permite transformar uma tarefa como:

> abra a página, espere o componente aparecer, valide um texto e tire uma screenshot

em uma action estruturada.

A screenshot pode voltar para a própria conversa como artifact.

---

## Por que separar control plane e executor

Um dos aprendizados mais importantes foi não tentar transformar um único agente no responsável por tudo.

O desenho que funcionou melhor separa responsabilidades.

### Control plane

Responsável por:

- manter contexto;
- entender a intenção;
- decidir o próximo passo;
- montar uma tarefa limitada;
- revisar resultados;
- detectar inconsistências.

### Executor

Responsável por:

- editar arquivos;
- executar testes;
- rodar comandos conhecidos;
- devolver evidência concreta.

Conceitualmente:

```text
modelo com contexto
      ↓
tarefa prescritiva
      ↓
executor local
      ↓
resultado verificável
      ↓
modelo com contexto
```

O ganho principal não é fazer o executor "pensar mais".

É reduzir a quantidade de ambiguidade que ele precisa resolver.

---

## Git sem dar acesso irrestrito

Uma dificuldade apareceu rapidamente: o Codex podia modificar o worktree, mas o sandbox não tinha permissão para escrever em `.git`.

Tentar contornar isso removendo o sandbox teria piorado a arquitetura.

A solução foi criar o **Git Finalizer** como componente separado.

O Codex edita.

O finalizer revisa o estado permitido e executa a parte Git.

O fluxo é:

```text
Codex
  ↓
arquivos alterados

Git Finalizer
  ↓
valida projeto
valida origin
valida HEAD esperado
valida branch
valida paths permitidos
  ↓
git add
git commit
git push
```

O finalizer ganhou várias proteções durante o desenvolvimento:

- paths explícitos;
- rejeição de paths absolutos;
- rejeição de `..`;
- validação de HEAD;
- validação de remote;
- validação de branch;
- limite de arquivos;
- commit message validada;
- replay idempotente;
- resultados persistidos;
- escrita atômica.

Isso acabou sendo melhor do que simplesmente permitir que qualquer executor rodasse Git livremente.

---

## Docker: WSL é a fonte de verdade

Outro requisito importante foi tornar explícito qual Docker deveria ser usado.

No meu ambiente existem fronteiras entre Windows e WSL.

Durante os primeiros testes surgiu a tentação de usar Docker Windows quando o socket do WSL não estivesse acessível ao sandbox.

A regra ficou diferente:

> Se a operação pertence ao ambiente Linux do projeto, Docker significa Docker dentro do WSL.

Não existe fallback silencioso para `docker.exe`.

Foi criado um watcher específico:

```text
docker-test-bridge.sh
```

O broker envia um request estruturado para esse runner, e o runner executa a suíte usando o daemon esperado.

Essa decisão evita um problema difícil de diagnosticar: um teste "passar" em um ambiente diferente daquele que realmente será usado.

---

## Browser QA como evidência

Antes do Januz, uma validação de frontend frequentemente terminava com:

> os testes passaram.

Mas testes de componente e build não provam que uma tela realmente está visualmente correta.

O `browser_qa` foi criado para fechar essa lacuna.

Ele consegue:

- abrir uma URL;
- aguardar um elemento;
- verificar conteúdo;
- tirar screenshots;
- devolver artifacts.

A diferença é pequena na implementação, mas grande no fluxo de trabalho.

Uma mudança de UI pode passar por:

```text
implementação
→ typecheck
→ testes
→ build
→ runtime
→ navegador real
→ screenshot
→ revisão
→ commit
```

Isso reduz a chance de finalizar uma feature tecnicamente correta e visualmente quebrada.

---

## Tornando o projeto reproduzível

A primeira versão do Januz era apenas um conjunto de arquivos espalhados pela máquina.

Parte estava no Windows.

Parte estava no WSL.

Parte estava em `$HOME/bin`.

As units estavam em `/etc/systemd/system`.

O broker tinha seus próprios arquivos locais.

Isso funcionava na máquina original, mas não era um projeto reproduzível.

A mudança para o repositório `z2ro/januz` exigiu trazer o source real para uma estrutura única:

```text
januz/
├── broker/
├── extension/
├── config/
├── scripts/
└── wsl/
    ├── bin/
    └── systemd/
```

Também foram adicionados:

- `scripts/install-windows.ps1`;
- `scripts/install-wsl.sh`;
- `config/januz.env.example`;
- templates systemd;
- README de bootstrap.

O objetivo passou a ser permitir:

```bash
git clone git@github.com:z2ro/januz.git
cd januz
```

e, a partir do clone, reconstruir o ambiente sem depender de conhecimento escondido na máquina original.

---

## Sucessos

### O loop completo realmente funcionou

O principal sucesso foi provar o fluxo:

```text
ChatGPT
→ extensão
→ broker
→ executor local
→ resultado
→ ChatGPT
```

Isso deixou de ser apenas arquitetura desenhada em papel.

Comandos reais chegaram ao Codex, ao Git, ao Docker e ao Browser QA.

### Resultados voltando automaticamente

A extensão não apenas dispara trabalho.

Ela também consegue devolver o resultado para a mesma conversa.

Isso mantém a conversa como histórico operacional da tarefa.

### Artifacts visuais

Screenshots produzidas pelo Browser QA podem ser recuperadas pelo broker e anexadas no chat.

O modelo consegue revisar a evidência visual no mesmo ciclo.

### Git Finalizer robusto

O finalizer começou como workaround para o sandbox do Codex e acabou virando uma camada de segurança útil.

Ele hoje sabe distinguir:

- worktree esperado;
- arquivo inesperado;
- HEAD incorreto;
- origin incorreto;
- replay;
- no changes;
- sucesso;
- erro.

### Escrita atômica e idempotência

Nos componentes Git e Docker, resultados importantes passaram a ser persistidos com escrita atômica.

Jobs já concluídos também podem ser reconhecidos novamente sem repetir determinadas operações.

### Portabilidade

O Januz deixou de ser apenas "a configuração da minha máquina".

Broker, extensão, watchers e templates agora estão no GitHub.

### Extensão configurável

O Control Chat ID pessoal deixou de existir no source.

Isso foi um passo simples, mas necessário para transformar a extensão em algo reutilizável.

### Segurança por restrição

O projeto não oferece uma action genérica de shell.

O broker fica em loopback.

Artifacts possuem validação de path.

O Git Finalizer possui allowlists e validações próprias.

Essas restrições tornam o sistema menos flexível, mas mais previsível.

---

## Falhas

As partes que falharam foram especialmente úteis porque mostraram quais problemas surgem quando se tenta conectar agentes a uma máquina real.

### `TypeError: Failed to fetch`

Durante uma validação real de UI, a extensão chegou ao estado:

```text
dispatch backoff
TypeError: Failed to fetch
```

O STARZ estava funcionando.

O problema era o bridge.

Esse incidente mostrou uma fraqueza importante: a extensão não tinha uma noção clara de:

- broker online;
- broker offline;
- timeout;
- reconnect;
- recuperação de job.

Todos esses casos acabavam parecendo praticamente o mesmo erro.

Essa falha virou a motivação principal para o futuro Januz v0.3.

### Timeout não significa que o trabalho não aconteceu

Uma operação pode ser enviada ao broker e a conexão cair antes da resposta.

Se a extensão simplesmente reenviar o comando, operações mutantes podem acontecer duas vezes.

Isso é particularmente perigoso para:

- Git;
- Docker;
- execução de tarefas longas.

A conclusão foi que reconnect precisa recuperar o estado existente por `job_id`, e não redisparar trabalho automaticamente.

### O sandbox do Codex não escreve em `.git`

Inicialmente isso parecia uma limitação incômoda.

Depois ficou claro que forçar acesso ao Git seria a solução errada.

A restrição acabou levando ao desenho do Git Finalizer.

### Docker e fronteiras de execução

O Codex nem sempre consegue acessar `/var/run/docker.sock` dependendo do sandbox.

Também houve uma falha concreta no runner PostgreSQL: em determinado ponto, o nome da network `starz_default` foi usado como hostname quando o correto era o alias de serviço `postgres`.

O problema não era Docker em si.

Era assumir que conceitos diferentes — network, container e hostname — eram intercambiáveis.

### O primeiro import levou `node_modules`

Quando o Januz foi colocado no GitHub, `broker/node_modules` foi versionado.

Isso gerou centenas de arquivos desnecessários no repositório.

A correção posterior removeu 183 arquivos tracked e deixou apenas:

```text
package.json
package-lock.json
```

Foi um erro básico, mas útil.

"Está no diretório do projeto" não significa "deve estar no Git".

### Hardcodes demais

O projeto começou muito acoplado à máquina original.

Exemplos incluíam:

- Control Chat ID fixo;
- paths locais;
- STARZ como projeto implícito;
- origin fixo;
- Docker network fixa.

Parte disso já foi removida.

Outra parte continua sendo dívida técnica.

### Ainda existem componentes específicos do STARZ

O Januz já é portátil como repositório, mas alguns executores ainda conhecem detalhes específicos do STARZ.

O Git Finalizer e o Docker runner nasceram para esse projeto.

A próxima evolução é mover isso para uma project registry declarativa, mantendo allowlists.

### Três brokers são um sinal de evolução incompleta

O repositório atualmente possui:

```text
broker.js
broker.ps1
app.py
```

O Node broker é a implementação principal.

Os outros arquivos representam etapas anteriores da evolução.

Manter implementações paralelas por muito tempo aumenta a chance de divergência.

### DOM scraping é inerentemente frágil

A extensão depende da interface web do ChatGPT.

Seletores como:

```text
#prompt-textarea
data-testid
contenteditable
aria-label
```

podem mudar.

Essa abordagem funciona, mas precisa ser tratada como adapter, e não como contrato estável.

### Proveniência de comandos precisa ficar mais rígida

Uma revisão do código mostrou outro ponto importante.

Um `LOCAL COMMAND` só deveria ser executado quando estiver inequivocamente dentro de uma mensagem do assistant.

Qualquer conteúdo equivalente vindo de:

- user;
- tool;
- container sem autoria reconhecida

deve ser ignorado.

Esse hardening está entre as próximas mudanças antes de expandir as capacidades do bridge.

---

## Aprendizados

### Automação não deve virar o produto principal

Durante o desenvolvimento foi fácil cair em um ciclo:

```text
melhorar bridge
→ melhorar bridge
→ melhorar bridge
→ esquecer o projeto que motivou o bridge
```

A regra que ficou foi:

> A automação serve ao desenvolvimento. O desenvolvimento não serve para justificar a automação.

Isso ajuda a decidir quando parar de melhorar infraestrutura e voltar para a feature real.

### Não dê shell quando uma action específica resolve

Existe uma diferença enorme entre:

```text
shell_exec("qualquer coisa")
```

e:

```text
git_finalize(...)
browser_qa(...)
wsl_docker_test(...)
```

Actions específicas permitem validar payload, path, projeto, estado e resultado.

Um shell genérico transfere toda a política de segurança para o prompt.

### Idempotência é necessária quando existe rede

A partir do momento em que existe:

```text
browser
→ HTTP
→ broker
→ filesystem
→ WSL
```

timeouts deixam de significar "nada aconteceu".

O sistema precisa distinguir:

- request nunca chegou;
- job foi aceito;
- job está rodando;
- job terminou;
- resultado foi perdido;
- processo morreu.

Sem isso, retry pode virar duplicação.

### Evidência precisa voltar junto com o status

"PASS" sozinho é fraco.

É melhor retornar:

- exit code;
- HEAD;
- diff;
- arquivos;
- screenshots;
- logs relevantes;
- estado do runtime.

Quanto mais determinística for a evidência, menos o control plane precisa adivinhar.

### Restrições criam melhores interfaces

A proibição de Git direto no sandbox levou ao Git Finalizer.

A regra de Docker somente no WSL levou a um runner específico.

O broker somente em loopback levou a uma fronteira de rede simples.

Em vários casos, a restrição melhorou o design.

### Portabilidade exige eliminar conhecimento implícito

Um projeto não é reproduzível apenas porque o source principal foi para o GitHub.

É necessário também saber:

- quais watchers existem;
- quais services existem;
- onde ficam;
- quais variáveis configuram paths;
- como instalar dependências;
- quais diretórios são runtime;
- como carregar a extensão.

Essa foi uma das diferenças entre o primeiro import e o Januz 0.2.4.

### Control plane e executor precisam ter responsabilidades diferentes

Um modelo com contexto arquitetural não precisa ser o processo que chama `git commit`.

Um executor local não precisa decidir sozinho a estratégia de produto.

Separar planejamento, execução e verificação reduziu a quantidade de decisões implícitas em cada camada.

---

## Estado atual

O Januz 0.2.4 possui:

- extensão Chrome Manifest V3;
- configuração de Control Chat ID;
- configuração de Broker URL;
- broker Node em loopback;
- protocolo de comandos estruturados;
- `codex_exec` e status;
- Git Finalizer;
- status de Git;
- Docker test bridge no WSL;
- status de Docker jobs;
- connectivity probe;
- Browser QA com Playwright;
- transferência de screenshots;
- handoff Windows ↔ WSL;
- resultados persistidos em componentes críticos;
- proteções contra path traversal em artifacts;
- scripts de instalação Windows/WSL;
- templates systemd;
- configuração por environment variables;
- source completo no GitHub.

A estrutura principal é:

```text
januz/
├── broker/
├── extension/
├── config/
├── scripts/
└── wsl/
    ├── bin/
    └── systemd/
```

---

## Próximos passos

### 1. Hardening da proveniência dos comandos

Antes de adicionar novas actions, a extensão deve aceitar comandos apenas de mensagens cujo autor seja inequivocamente o assistant.

Também devem ficar mais rígidos:

- comparação do Control Chat ID;
- validação de command ID;
- action allowlist na extensão.

### 2. Project registry

Remover hardcodes diretos do STARZ sem transformar o Januz em executor arbitrário.

A ideia é usar uma registry declarativa:

```text
project name
→ path permitido
→ origin esperado
→ capabilities
→ Docker config permitida
```

O payload referencia apenas o nome do projeto.

Paths e remotes continuam fora do controle do comando.

### 3. CI

Adicionar GitHub Actions para validar:

- Node 20;
- `npm ci`;
- sintaxe JS;
- testes da extensão;
- Bash;
- Python;
- JSON;
- manifest.

Sem executar Codex, Docker ou browser real no CI.

### 4. Januz v0.3 Reliability

Essa é a evolução mais importante.

O plano inclui:

- job store persistente;
- estados canônicos;
- heartbeat;
- reconnect;
- exponential backoff;
- recuperação sem redispatch;
- erros semânticos;
- `GET /jobs/:id`;
- `GET /diagnostics`;
- `GET /capabilities`.

Estados internos pretendidos:

```text
QUEUED
STARTING
RUNNING
SUCCEEDED
FAILED
CANCELLED
INTERRUPTED
```

Se o broker reiniciar, um job antigo não deve permanecer falsamente `RUNNING`.

### 5. Melhorar Browser QA

Além de screenshot, o QA pode retornar melhor contexto de falha:

- URL final;
- console errors;
- failed requests;
- selector timeout;
- DOM excerpt limitado;
- screenshots intermediárias.

### 6. Modularizar o broker

O `broker.js` já cresceu bastante.

Depois do v0.3 estabilizar, faz sentido separar:

```text
server
protocol
config
artifacts
actions/codex
actions/git
actions/docker
actions/browser
jobs
```

A modularização deve vir depois do comportamento estar definido, não antes.

### 7. Reduzir polling e scanning do DOM

A extensão hoje usa MutationObserver combinado com scans periódicos.

Uma evolução melhor é acompanhar apenas novos conversation turns e registrar o último comando processado.

Isso reduz trabalho em conversas longas.

### 8. Consolidar uma única implementação de broker

`app.py` e `broker.ps1` devem eventualmente ser removidos ou marcados claramente como legacy.

O Node broker deve ser a implementação canônica.

### 9. Portabilidade além de Windows + WSL

Com o source agora organizado, ficou possível considerar outros ambientes.

Uma possibilidade interessante é executar o Januz em uma VM Linux ARM64, mantendo o mesmo protocolo e substituindo os adapters específicos de Windows/WSL.

Essa evolução só faz sentido depois da base de reliability ficar estável.

---

## O que eu não quero que o Januz vire

É importante definir também o não-objetivo.

O Januz não deveria virar:

- um RAT local;
- uma API de shell remoto;
- um daemon exposto na rede;
- um agente com acesso irrestrito ao filesystem;
- uma coleção infinita de automações específicas;
- uma desculpa para não desenvolver os projetos reais.

A direção desejada é outra:

```text
poucas actions
+
contratos claros
+
allowlists
+
evidência
+
recuperação
+
automação previsível
```

---

## Resumo

O Januz começou como uma ponte improvisada entre uma conversa e um Codex local.

No processo, acabou expondo problemas mais interessantes:

- como representar trabalho assíncrono;
- como evitar redispatch;
- como separar planejamento de execução;
- como usar Git sem liberar `.git` para qualquer processo;
- como garantir que Docker seja executado no ambiente correto;
- como trazer screenshots e evidências de volta para o control plane;
- como tornar automação local reproduzível;
- como limitar autoridade sem perder utilidade.

Os sucessos provaram que o loop completo funciona.

As falhas mostraram que confiabilidade e fronteiras de segurança são mais importantes do que adicionar mais actions.

O próximo estágio não é fazer o Januz "fazer mais coisas".

É fazer o que ele já faz de maneira mais previsível:

```text
command
→ accepted
→ observable
→ recoverable
→ verifiable
→ result
```

Se essa base ficar sólida, novas capacidades deixam de ser hacks isolados e passam a ser extensões de um protocolo local controlado.
