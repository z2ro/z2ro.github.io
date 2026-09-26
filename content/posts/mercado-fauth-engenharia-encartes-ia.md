---
title: "Mercado Fauth — engenharia de um gerador determinístico de encartes com IA controlada"
date: 2026-09-25T23:52:00-03:00
draft: false
---

## Visão geral

O projeto Mercado Fauth nasceu de um problema simples de explicar e trabalhoso de resolver bem:

> transformar uma lista de produtos, preços, unidades e imagens em um encarte promocional 1080×1080 que pareça material real de supermercado.

A solução óbvia seria pedir para um modelo de imagem ou um LLM "criar o banner inteiro".

Eu segui na direção oposta.

O projeto foi desenhado para manter os dados comerciais determinísticos e verificáveis, enquanto a IA, quando usada, fica restrita a decisões de direção visual de alto nível.

A arquitetura atual combina:

- FastAPI;
- Pydantic v2;
- `Decimal` para preços;
- processamento local de imagens;
- classificação de produtos;
- planner de layout;
- templates HTML/CSS;
- Playwright/Chromium;
- QA visual determinístico;
- AI Art Director opcional;
- Visual QA multimodal opcional.

O código está no repositório:

[github.com/z2ro/mercado-fauth](https://github.com/z2ro/mercado-fauth)

## O problema real

Um encarte de supermercado parece simples até tentar automatizá-lo.

Cada peça precisa preservar exatamente:

- nome do produto;
- preço;
- unidade;
- validade;
- endereço;
- telefone;
- Instagram;
- imagens;
- regras comerciais.

Ao mesmo tempo, precisa resolver decisões visuais:

- quais produtos devem receber mais destaque;
- quais categorias podem ficar próximas;
- quanto espaço cada card recebe;
- como equilibrar preço, imagem e texto;
- como evitar clipping;
- como manter a identidade visual;
- como produzir sempre 1080×1080.

Um erro de layout é incômodo.

Um erro de preço é inaceitável.

Essa diferença determinou praticamente toda a arquitetura.

## Princípio central: IA não é fonte de verdade comercial

Os dados comerciais vivem nos modelos de entrada e permanecem fora do controle da IA.

O Art Director pode escolher algo como:

```text
template = weekend_hero
hero = p001
featured = p006, p008
mood = bold
presentation_profile = image_focus
```

Mas ele não pode decidir:

```text
price = 29.90
name = "Picanha Premium"
x = 124
y = 391
css = "..."
```

Preço continua sendo `Decimal`.

Nome, unidade, SKU e demais campos continuam vindo do payload original.

O `DesignSpec` descreve layout, não duplica informação comercial.

Essa separação também simplifica testes: se o renderer recebe um produto de R$ 39,90, o HTML e o PNG precisam continuar representando R$ 39,90.

## Arquitetura atual

O fluxo conceitual ficou assim:

```text
Campaign JSON
      ↓
Pydantic validation
      ↓
Product classification
      ↓
Resolved products
      ↓
Optional Art Director
      ↓
Layout Planner
      ↓
DesignSpec
      ↓
Jinja2 / HTML / CSS
      ↓
Playwright / Chromium
      ↓
Deterministic Visual QA
      ↓
Optional multimodal QA
      ↓
PNG 1080×1080
```

Cada camada tem uma responsabilidade diferente.

### API e validação

A API usa FastAPI.

Os schemas são Pydantic v2 com validações estritas.

A campanha atual trabalha com 12 produtos por banner.

Campos inesperados são rejeitados em partes críticas, e os contratos de direção de arte usam `extra='forbid'`.

Isso é especialmente importante quando um modelo externo participa do fluxo.

### Layout V1 e V2

O projeto mantém dois contratos de layout.

O V1 preserva o comportamento legado do template `supermarket_12`, em uma grade fixa 4×3.

O V2 suporta placements retangulares com:

- `x`;
- `y`;
- `w`;
- `h`;
- role;
- zone;
- escala de imagem;
- escala de preço.

As roles atuais incluem:

- `hero`;
- `featured`;
- `standard`;
- `compact`.

Os templates V2 principais são:

- `weekend_hero`;
- `price_attack`.

Posteriormente foi adicionado o `faith_reference_12`, criado para reproduzir mais de perto a linguagem visual de um encarte real do Mercado Fauth.

## Regras de adjacência

Nem todo produto pode ser colocado em qualquer posição.

Uma das regras comerciais/visuais adotadas no MVP é impedir que produtos de limpeza fiquem diretamente adjacentes a:

- açougue;
- hortifruti;
- padaria.

Isso começou no planner legado e depois precisou ser generalizado para placements retangulares.

O planner não escolhe apenas uma ordem.

Ele precisa encontrar uma disposição que:

1. contenha todos os produtos;
2. respeite hard rules;
3. não tenha overlap;
4. mantenha o conteúdo dentro do grid;
5. maximize afinidade entre categorias quando possível.

## Affinity score

Além das hard rules, existe um score de afinidade.

Alguns pares de departamentos fazem mais sentido próximos que outros.

Exemplos:

```text
açougue + frios       → positivo
hortifruti + mercearia → positivo
limpeza + higiene     → positivo
limpeza + bebidas     → negativo
```

O objetivo não é provar que existe uma única disposição "correta".

O objetivo é reduzir layouts obviamente ruins e tornar o comportamento reproduzível.

A arquitetura separa duas coisas:

```text
hard rules
→ nunca podem ser violadas

soft score
→ ajuda a escolher entre layouts válidos
```

## Por que o planner não é um LLM

Uma decisão importante foi não delegar coordenadas diretamente a um modelo.

O LLM pode ser bom em:

- intenção;
- estilo;
- seleção de destaque;
- variedade visual.

Mas geometria tem propriedades melhores quando é tratada de forma determinística.

O Layout Engine continua sendo responsável por:

- posições;
- dimensões;
- overlap;
- limites;
- separação entre departamentos.

Isso cria uma fronteira clara:

```text
AI Art Director
→ "o que destacar"

Layout Engine
→ "como isso cabe"
```

## Pipeline de imagens reais

O sistema também ganhou um pipeline local para imagens de produto.

As imagens passam por:

- leitura limitada por tamanho;
- correção de orientação via EXIF;
- conversão para RGBA;
- remoção de fundo opcional;
- crop pelo alpha;
- padding;
- normalização;
- resize;
- cache SHA-256.

O limite atual normaliza assets para no máximo 600 px por lado sem upscale agressivo.

A remoção de fundo é opcional e local.

Se o remover falhar, o sistema preserva o asset original em vez de interromper toda a campanha.

Essa decisão foi importante porque pipelines de imagem reais falham de formas pouco elegantes: transparência vazia, arquivo corrompido, EXIF estranho, imagem gigante ou recorte ruim.

## Classificação de produtos

O sistema aceita categoria fornecida explicitamente.

Quando não existe, há uma cadeia de resolução.

Conceitualmente:

```text
user-provided category
      ↓
cache
      ↓
strong rules
      ↓
optional LLM
      ↓
fallback rules
      ↓
unresolved
```

As categorias atuais são:

- açougue;
- hortifruti;
- bebidas;
- mercearia;
- frios;
- padaria;
- limpeza;
- higiene.

O classificador por LLM recebe apenas o necessário para classificar.

Preço, SKU e imagem não precisam participar dessa decisão.

## O evaluator de classificação

Antes de continuar adicionando inteligência, eu quis medir a classificação atual.

Foi criado um dataset de avaliação com 120 produtos:

- 15 por categoria;
- 40 fáceis;
- 40 médios;
- 40 difíceis.

O baseline de regras atingiu:

```text
accuracy: 0.35
macro precision: 0.9479
macro recall: 0.35
macro F1: 0.5027
```

A leitura importante desses números não é "o classificador está bom" ou "está ruim".

É que as regras conhecem poucos casos, mas quando reconhecem algo tendem a ser precisas.

Isso confirmou que o componente de regras funciona melhor como camada de alta confiança do que como classificador universal.

O provider fake foi mantido apenas para testar a infraestrutura do pipeline, não para medir qualidade de IA.

## AI Art Director

Com o Layout Engine já funcionando, foi possível adicionar uma camada de direção de arte opcional.

O modo padrão continua manual.

O modo automático é opt-in.

O `ArtDirectionSpec` permite apenas escolhas bounded:

- `weekend_hero` ou `price_attack`;
- até um hero;
- até dois featured;
- mood;
- emphasis;
- presentation profile.

Os profiles atuais são:

- `balanced`;
- `image_focus`;
- `price_focus`;
- `dense`;
- `premium`.

O modelo não recebe preço para decidir a direção.

A resposta passa novamente por validação Pydantic.

Se o provider falha, expira ou retorna algo inválido, o sistema cai para um Art Director determinístico.

## Structured Outputs e fronteira de segurança

O transporte de provider usa schema JSON estrito.

Os prompts também tratam nomes de campanha e produtos como dados não confiáveis.

Isso reduz duas classes de problema:

1. output malformado;
2. conteúdo de produto tentando se comportar como instrução.

O modelo pode selecionar opções existentes.

Ele não pode criar CSS arbitrário nem inventar geometria.

Essa restrição foi intencional: o objetivo da IA é ajudar na composição, não ganhar autoridade sobre o renderer.

## Visual QA determinístico

Depois do Chromium gerar o PNG, o pipeline executa verificações estruturais.

Entre elas:

- PNG válido;
- 1080×1080;
- canvas correto;
- 12 produtos presentes;
- IDs únicos;
- dados comerciais preservados;
- imagens carregadas;
- bounds válidos;
- ausência de overlap;
- detecção de overflow;
- presença de hero quando o template exige.

O QA determinístico é importante porque problemas verificáveis não deveriam depender de julgamento de um modelo multimodal.

## Visual QA multimodal opcional

Existe também uma camada opcional de QA por visão.

Ela recebe o PNG final e retorna apenas códigos de problemas permitidos, como:

- `text_clipping`;
- `image_clipping`;
- `weak_hero_emphasis`;
- `weak_price_hierarchy`;
- `crowded_layout`;
- `excessive_empty_space`;
- `poor_balance`;
- `low_contrast`;
- `footer_too_prominent`;
- `header_too_prominent`.

Mesmo nessa etapa o modelo não recebe liberdade para editar HTML ou CSS.

Ele pode sugerir um profile permitido.

O refinement é limitado e não entra em loop infinito.

Os providers reais continuam opcionais e podem permanecer desligados durante desenvolvimento e CI.

## O primeiro golden reference

Depois de construir bastante infraestrutura, ficou claro que o projeto precisava parar de adicionar abstrações e responder a uma pergunta mais simples:

> consegue gerar uma peça visualmente boa?

Para isso foi escolhida uma arte real do Mercado Fauth como referência.

O objetivo não era originalidade.

Era criar um ponto de comparação concreto.

A primeira estratégia foi reproduzir:

- fundo amarelo;
- faixa superior;
- hashtags;
- validade;
- grid 4×3;
- brush verde para preço;
- placa azul para nome;
- footer com contatos, marca, horários e endereço.

Foi criado o template:

```text
faith_reference_12
```

Ele reaproveita o planner 4×3.

O `supermarket_12` continua sendo o padrão para payloads legados.

Quando a ordem original dos produtos respeita as hard rules, ela pode ser preservada. Quando existe conflito, o planner continua tendo autoridade para reorganizar os itens.

## Campos opcionais de campanha

Para aproximar o template da peça real, a campanha passou a aceitar também dados opcionais como:

- telefone secundário;
- horários de funcionamento.

Isso permitiu reproduzir um footer mais próximo do material oficial sem obrigar todos os templates a dependerem desses campos.

## Estado atual da validação

Na etapa mais recente:

- 230 testes passaram;
- o Docker foi reconstruído;
- a API subiu normalmente;
- o banner foi gerado via endpoint;
- o QA determinístico aprovou;
- Pillow confirmou 1080×1080;
- `git diff --check` passou.

O primeiro resultado já reproduz corretamente a estrutura geral da referência, mas ainda expõe algo importante: um banner tecnicamente válido não é automaticamente um banner bonito.

Essa diferença é exatamente o foco da fase atual.

## O que funcionou

### Determinismo comercial

Nomes, preços e unidades permanecem fora da autoridade da IA.

Essa continua sendo uma das decisões mais importantes do projeto.

### Layout verificável

O sistema consegue provar propriedades simples:

- não há overlap lógico;
- o banner cabe;
- os produtos existem;
- as hard rules foram respeitadas.

### Fallbacks

Classificação, Art Director, assets e Visual QA possuem caminhos degradados.

Uma chamada externa não precisa ser um single point of failure.

### Templates diferentes sem trocar o pipeline

`supermarket_12`, `weekend_hero`, `price_attack` e `faith_reference_12` usam o mesmo núcleo.

Isso permite testar direções visuais distintas sem duplicar a aplicação inteira.

### Testes offline

Providers fake permitem validar contracts, caching, fallbacks e integração sem depender de credenciais externas.

## O que não funcionou de primeira

### Qualidade visual não emerge de arquitetura

Foi possível chegar a centenas de testes passando antes de chegar a uma peça que realmente parecesse um encarte de supermercado.

Isso é um lembrete importante.

Um sistema pode estar:

- correto;
- determinístico;
- testado;
- observável;

e ainda assim produzir algo visualmente mediano.

Para design, é necessário um loop de referência, render, comparação e refinamento.

### Placeholder não substitui produto real

O primeiro `faith_reference_12` ainda usa placeholders.

Eles são úteis para validar composição, mas criam muito espaço vazio e fazem o banner parecer um mock técnico.

A validação final precisa acontecer com fotos reais.

### Heurística não conhece estética suficiente

O Art Director determinístico pode tomar decisões coerentes do ponto de vista de regra, mas ainda sabe pouco sobre a forma visual do asset.

Um próximo passo útil é incluir metadados seguros como:

```text
vertical
horizontal
square
```

sem entregar geometria ao modelo.

### QA estrutural não é crítica de design

O Visual QA determinístico consegue dizer que o texto cabe.

Ele não consegue, sozinho, decidir se o preço deveria ter 12% mais presença ou se o footer parece fraco.

Por isso o golden reference passou a ser essencial.

## Lições

### Use IA onde ambiguidade ajuda

Direção de arte é ambígua.

Preço não é.

Geometria pode ser verificada.

Esse contraste ajuda a decidir onde a IA deve ou não entrar.

### Hard constraints antes de score

Regras obrigatórias precisam ser filtradas antes de qualquer otimização estética.

Um layout bonito que viola uma regra comercial continua inválido.

### O renderer deve conseguir funcionar sem IA

Se toda geração depender de provider externo, a aplicação perde previsibilidade e testabilidade.

O caminho determinístico precisa continuar sendo suficiente para produzir um banner.

### Golden samples são testes de produto

Testes unitários respondem se o código está correto.

Golden samples ajudam a responder se o produto está bom.

São problemas diferentes.

### Qualidade visual precisa de iteração real

A fase atual não exige mais uma grande camada arquitetural.

Exige pequenos ciclos:

```text
render
→ comparar com referência
→ identificar diferenças
→ ajustar
→ render novamente
```

## Estado atual

Hoje o projeto já possui:

- API FastAPI;
- validação Pydantic;
- preços com Decimal;
- quatro direções/templates de banner;
- Layout Engine V1/V2;
- hard rules;
- affinity scoring;
- processamento local de imagens;
- remoção de fundo opcional;
- cache de assets;
- classificação híbrida;
- evaluator de classificação;
- AI Art Director opcional;
- Structured Outputs;
- cache de direção de arte;
- Visual QA determinístico;
- Visual QA multimodal opcional;
- refinement limitado;
- CI;
- Docker;
- 230 testes passando na validação mais recente;
- primeiro golden reference renderizado em 1080×1080.

## Próximos passos

A prioridade agora é visual.

Os próximos passos mais úteis são:

1. corrigir pequenos problemas detectáveis no QA de clipping;
2. usar pipeline separado para logo e produto;
3. preservar tokens centrais da marca entre moods;
4. fornecer o logo real;
5. substituir placeholders por fotos reais;
6. melhorar tipografia de display;
7. refinar o `faith_reference_12` contra a arte real;
8. criar um golden banner aprovado;
9. usar esse golden como referência para medir o Art Director;
10. só depois expandir automação ou integrações.

Integrações como ERP, Sheets, storage, publicação automática e filas podem vir depois.

O problema mais importante neste momento não é transportar mais dados.

É provar que o resultado final merece ser publicado.

## Resumo

O Mercado Fauth começou como um gerador de banner, mas evoluiu para um experimento de separação entre decisões determinísticas e decisões assistidas por IA.

A ideia central ficou simples:

```text
dados comerciais
→ determinísticos

geometria e regras
→ determinísticas

direção visual
→ pode usar IA

qualidade estrutural
→ verificada automaticamente

qualidade estética
→ comparada com referência real
```

O projeto já provou que consegue gerar banners corretos.

A etapa atual é mais difícil de medir, mas mais importante para o produto:

fazer com que o banner pareça ter sido montado por alguém que entende publicidade de supermercado, sem abrir mão da previsibilidade do sistema.
