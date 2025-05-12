---
layout: post
title: "Guia: Contribuindo com a Localização da Documentação do OpenTelemetry para Português"
date: 2024-07-26
author: "Vitor Vasconcellos"
lang: pt
img: ":2024-07-26-guia-contribuicao-otel-docs-pt.png"
categories: [OpenTelemetry, Localização, Contribuição]
tags: [opentelemetry, localization, portugues, guia, contribuicao]
---

Olá, aqui você vai encontrar todas (ou quase todas) as informações para começar
a contribuir com a localização da documentação do OpenTelemetry para Português!

## Como começar

Antes de colocar a mão na massa, recomendamos a leitura da seção [Contribuindo](https://opentelemetry.io/pt/docs/contributing/){:target="_blank"}, na documentação oficial do OpenTelemetry. Lá você encontrará dicas preciosas sobre como contribuir, padrões e guias de estilo que vão te ajudar bastante!

Na sequência deste guia, abordaremos pontos importantes para te guiar durante o processo de contribuição.

## Sobre a documentação do OpenTelemetry

- Link para a documentação: [https://opentelemetry.io/pt/docs/](https://opentelemetry.io/pt/docs/){:target="_blank"}.
- Link para o repositório da documentação: [https://github.com/open-telemetry/opentelemetry.io](https://github.com/open-telemetry/opentelemetry.io){:target="_blank"} 
- As documentações são escritas em [Markdown](https://www.markdownguide.org/basic-syntax/){:target="_blank"}.

## Sobre o time de localização (Português)

- Toda a comunicação acontece no canal [#otel-localization-ptbr](https://cloud-native.slack.com/archives/C076LET8YSK){:target="_blank"} no workspace da CNCF no slack, então antes de tudo, entre no canal!
  - Se você ainda não estiver entrado no workspace, pode solicitar um convite [aqui](https://communityinviter.com/apps/cloud-native/cncf){:target="_blank"}
- Temos encontros quinzenais via Zoom (quarta-feira, 19:15-19:45 BRT), detalhes da agenda e pauta, podem ser encontrados [aqui](https://docs.google.com/document/d/1W1jJ4OTm53sbOp7CrbNBMvR_2Z8TQRCkwejqD4f21SE/edit){:target="_blank"}.

## Páginas prioritárias para tradução/localização

As páginas priorizadas podem ser encontradas através da utilização do filtro [`lang:pt`](https://github.com/open-telemetry/opentelemetry.io/issues?q=is%3Aissue%20state%3Aopen%20label%3Alang%3Apt){:target="_blank"}, no repositório da documentação.

> Nota: Antes de iniciar uma tradução, verifique que a tradução da página não foi iniciada por outra pessoa! Isso é importante para evitar duplicação de trabalho.

## Detalhes importantes

Alguns detalhes importantes que você deve conferir antes de abrir seu _Pull Request_:

- Familiarize-se com os termos do [glossário](https://opentelemetry.io/docs/concepts/glossary/){:target="_blank"} de OpenTelemetry e as suas respectivas [traduções](https://opentelemetry.io/pt/docs/concepts/glossary/){:target="_blank"} para Português. Isso nos ajuda a manter consistência entre as documentações.
  - Caso tenha dúvidas sobre a tradução de um termo, verifique o [Glossário de Localização](https://docs.google.com/document/d/1kyu6HgdsM3-iDyf3OuRmfw4SwNJy9BVui6nn5JsKb5I/edit?tab=t.0#heading=h.o389kqcsmupl){:target="_blank"}, desenvolvido pelo time.
- Para garantir que a validação dos links na página funcione corretamente, precisamos manter as âncoras (_anchors_) sem tradução.  
  Temos uma action do Github que verifica os links da página, caso esteja quebrado gera avisos de `Warnings` no build, exemplo de âncora:

```md
Título: ## O que é Observabilidade? {#what-is-observability}

Título: ## Trechos {#span}

Título: ## Rastros {#traces}
```

- Caso o arquivo original contenha um campo `redirects:` no cabeçalho, remova.

- Cada tradução é baseada em um commit da documentação original, em inglês. As páginas traduzidas, obrigatóriamente devem ter um campo chamado `default_lang_commit` em seu cabeçalho, contendo a hash do commit da documentação original. Por exemplo:

```md
title: OpenTelemetry
description: >-
Telemetria de alta qualidade, abrangente e portátil para permitir uma observabilidade eficaz
developer_note:
O shortcode blocks/cover (usado abaixo) vai servir como imagem de background
para qualquer arquivo de imagem que contenha "background" no nome.
show_banner: true
default_lang_commit: 08799298
```

### Páginas novas

Para uma página nova, você precisa executar o seguinte comando para buscar a hash do commit:

```bash
docker run -it  --rm -v$(pwd):/app -w /app  --entrypoint "" node:latest npm run fix:i18n:new
```

### Páginas existentes

Antes de iniciar a atualização na localização de uma página existente, recomendamos que você busque a diferença entre a versão original e a versão que você está traduzindo. Isso pode ser feito executando o seguinte comando:

```bash
npm run check:i18n -- -d <PATH-TO-FILE>
```

**Exemplo:**

Vamos supor que você está traduzindo a página `content/pt/docs/languages/go/instrumentation.md`. O comando para verificar a diferença é:

```bash
npm run check:i18n -- -d content/pt/docs/languages/go/instrumentation.md
```

Output:

```diff
❯ npm run check:i18n -- -d content/pt/docs/languages/go/instrumentation.md

> check:i18n
> scripts/check-i18n.sh -d content/pt/docs/languages/go/instrumentation.md

Processing paths: content/pt/docs/languages/go/instrumentation.md
diff --git a/content/en/docs/languages/go/instrumentation.md b/content/en/docs/languages/go/instrumentation.md
index 9a7d8bf7..ace082d7 100644
--- a/content/en/docs/languages/go/instrumentation.md
+++ b/content/en/docs/languages/go/instrumentation.md
@@ -8,7 +8,7 @@ description: Manual instrumentation for OpenTelemetry Go
 cSpell:ignore: fatalf logr logrus otlplog otlploghttp sdktrace sighup
 ---

-{{% docs/languages/instrumentation-intro %}}
+{{% include instrumentation-intro.md %}}

 ## Setup

@@ -49,7 +49,7 @@ func newExporter(ctx context.Context)  /* (someExporter.Exporter, error) */ {
        // Your preferred exporter: console, jaeger, zipkin, OTLP, etc.
 }

-func newTraceProvider(exp sdktrace.SpanExporter) *sdktrace.TracerProvider {
+func newTracerProvider(exp sdktrace.SpanExporter) *sdktrace.TracerProvider {
        // Ensure default SDK resources and the required service name are set.
        r, err := resource.Merge(
                resource.Default(),
@@ -78,7 +78,7 @@ func main() {
        }

        // Create a new tracer provider with a batch span processor and the given exporter.
-       tp := newTraceProvider(exp)
+       tp := newTracerProvider(exp)

        // Handle shutdown properly so nothing leaks.
        defer func() { _ = tp.Shutdown(ctx) }()
@@ -221,7 +221,7 @@ span.AddEvent("Cancelled wait due to external signal", trace.WithAttributes(attr

 ### Set span status

-{{% docs/languages/span-status-preamble %}}
+{{% include "span-status-preamble.md" %}}

 ```go
 import (
@@ -549,10 +549,14 @@ import (
        "go.opentelemetry.io/otel/metric"
 )

-var fanSpeedSubscription chan int64
+var (
+  fanSpeedSubscription chan int64
+  speedGauge metric.Int64Gauge
+)

 func init() {
-       speedGauge, err := meter.Int64Gauge(
+       var err error
+       speedGauge, err = meter.Int64Gauge(
                "cpu.fan.speed",
                metric.WithDescription("Speed of CPU fan"),
                metric.WithUnit("RPM"),
DRIFTED files: 1 out of 1
```

> Nota: O comando acima irá retornar as linhas que foram alteradas. Considere adicionar esta diferença na descrição do seu pull request! Isso facilita a validação das revisões :)
> Exemplo: [Pull Request #6783](https://github.com/open-telemetry/opentelemetry.io/pull/6783){:target="_blank"}

Após verificar a diferença, você pode iniciar a atualização na localização da página.

Ao finalizar, não se esqueça de atualizar o `default_lang_commit` com o hash do commit da documentação original! Isso pode ser feito executando o seguinte comando:

```bash
npm run fix:i18n:status
```

E finalmente, garanta que a o campo `drifted_from_default` também foi atualizado e removido do cabeçalho. Isso pode ser feito executando o seguinte comando:

```bash
npm run fix:i18n:status
```

Mais detalhes sobre como atualizar a tradução podem ser encontrados [aqui](https://opentelemetry.io/docs/contributing/localization/#track-changes){:target="_blank"}.

## Passo a Passo

Após estar familiarizado com o processo de contribuição, siga o passo a passo abaixo para fazer sua primeira contribuição.

1. Pesquise se já existe uma issue aberta para a documentação que quer localizar (pode usar [esse](https://github.com/open-telemetry/opentelemetry.io/issues?q=is%3Aopen+is%3Aissue+label%3Alang%3Apt){:target="_blank"} filtro de busca), se já existir uma issue e alguém trabalhando, procure outra página.
2. Caso não exista issue aberta para a página que quer localizar, pode criar uma, seguindo o padrão de título: `[pt] Localize <caminho do arquivo que você vai trabalhar>`

3. Faça o fork do repositório da documentação: `https://github.com/open-telemetry/opentelemetry.io`

4. Faça o clone do seu fork para sua máquina local, exemplo:

```bash
git clone git@github.com:edsoncelio/opentelemetry.io.git
```

5. Crie uma branch a partir da branch `main` para fazer sua tradução, exemplo:

```bash
git checkout -b adiciona_traducao_traces
```

6. Com a branch criada, faça toda a sua tradução, sempre lembrando de fazer o push para o seu fork remoto, exemplo:

```bash
git add .
git commit -m "docs: adiciona traducao de traces"
git push origin adiciona_traducao_traces
```

7. Sempre antes de abrir um _Pull Request_, execute o _lint_, assim não vai quebrar nas checagens automáticas:

```bash
docker run -it  --rm -v$(pwd):/app -w /app  --entrypoint "" node:latest npx prettier --write .
```

8. Quando finalizar, ou apenas quiser solicitar uma revisão, abre o _Pull Request_ para a branch main.  
   Para essa etapa, sugerimos que use o prefixo `[pt]`no título, assim mantemos consistência, por exemplo:

```bash
[pt] Localize content/pt/docs/concepts/components.md
```

9. Envie o PR no canal do slack `#otel-localization-ptbr`.
10. Aguarde as interações para revisão, é esperado que tenha bastante, não se assuste :)

## ID ausente no Commit

O problema é que o autor dos commits não está vinculado à conta do usuário no GitHub e isso é necessário para que o EasyCLA identifique o usuário e conceda autorização.

![image](https://github.com/user-attachments/assets/525f72b0-09ef-42dc-86f2-9ede4e7e3fc2)

Geralmente ajustar as configurações globais do GitHub resolve o problema.

```sh
git commit --amend --author="FirstName LastName <emailaddress>" --no-edit

git push --force
```

### Caso Contrário faça rebase do commit:

Primeiro identifique qual commit você precisa ajustar.

![image](https://github.com/user-attachments/assets/ac173020-0bd0-4e22-a980-b7489b3ea0a1)

```sh
git rebase -i <SHA-Resumido-Commit>^
```

Isso abrirá um editor, mude o comando `pick` para `edit` no commit que deseja corrigir:

```sh
pick <SHA-Resumido-Commit> feat: mensagem de exemplo do commit
```

Salve e feche o editor.

Agora, atualize o autor do commit com o comando abaixo, substituindo pelo nome e e-mail corretos:

```sh
git commit --amend --author="FirstName LastName <emailaddress>" --no-edit
``` 