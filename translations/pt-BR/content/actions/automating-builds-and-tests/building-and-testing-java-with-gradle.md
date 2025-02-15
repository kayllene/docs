---
title: Criar e estar o Java com o Gradle
intro: Você pode criar um fluxo de trabalho de integração contínua (CI) no GitHub Actions para criar e testar o seu projeto Java com o Gradle.
redirect_from:
  - /actions/language-and-framework-guides/building-and-testing-java-with-gradle
  - /actions/guides/building-and-testing-java-with-gradle
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
type: tutorial
topics:
  - CI
  - Java
  - Gradle
shortTitle: Criar & testar Java & Gradle
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

## Introdução

Este guia mostra como criar um fluxo de trabalho que realiza a integração contínua (CI) para o seu projeto Java usando o sistema de criação do Gradle. O fluxo de trabalho que você criar permitirá que você veja quando commits em um pull request gerarão falhas de criação ou de teste em comparação com o seu branch-padrão. Essa abordagem pode ajudar a garantir que seu código seja sempre saudável. Você pode estender seu fluxo de trabalho de CI para memorizar arquivos e fazer o upload de artefatos a partir da execução de um fluxo de trabalho.

{% ifversion ghae %}
{% data reusables.actions.self-hosted-runners-software %}
{% else %}
Os executores hospedados em {% data variables.product.prodname_dotcom %} têm uma cache de ferramentas com com software pré-instalado, que inclui kits de desenvolvimento Java (JDKs) e Gradle. Para obter uma lista de software e as versões pré-instaladas para JDK e Gradle, consulte "[Especificações para executores hospedados em {% data variables.product.prodname_dotcom %}](/actions/reference/specifications-for-github-hosted-runners/#supported-software)".
{% endif %}

## Pré-requisitos

Você deve estar familiarizado com o YAML e a sintaxe do {% data variables.product.prodname_actions %}. Para obter mais informações, consulte:
- "[Sintaxe de fluxo de trabalho para o {% data variables.product.prodname_actions %}](/actions/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions)"
- "[Aprenda {% data variables.product.prodname_actions %}](/actions/learn-github-actions)"

Recomendamos que você tenha um entendimento básico da estrutura do Java e do Gradle. Para obter mais informações, consulte "[Introdução](https://docs.gradle.org/current/userguide/getting_started.html)" na documentação do Gradle.

{% data reusables.actions.enterprise-setup-prereq %}

## Usando o fluxo de trabalho inicial do Gradle

{% data variables.product.prodname_dotcom %} fornece um fluxo de trabalho inicial do Gradle que funcionará para a maioria dos projetos Java baseados no Gradle. Para obter mais informações, consulte o [Fluxo de trabalho inicial do Gradle](https://github.com/actions/starter-workflows/blob/main/ci/gradle.yml).

Para começar rapidamente, você pode escolher o fluxo de trabalho inicial pré-configurado do Gradle ao criar um novo fluxo de trabalho. Para obter mais informações, consulte o início rápido "[{% data variables.product.prodname_actions %}](/actions/quickstart)".

Você também pode adicionar este fluxo de trabalho manualmente, criando um novo arquivo no diretório `.github/workflows` do seu repositório.

```yaml{:copy}
{% data reusables.actions.actions-not-certified-by-github-comment %}

name: Java CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
      - name: Validate Gradle wrapper
        uses: gradle/wrapper-validation-action@e6e38bacfdf1a337459f332974bb2327a31aaf4b
      - name: Build with Gradle
        run: ./gradlew build
```

Este fluxo de trabalho executa os seguintes passos:

1. O `checkout` faz o download de uma cópia do seu repositório no executor.
2. A etapa `setup-java` configura o Java 11 JDK pelo Adoptium.
3. A etapa "Validar o invólucro do Gradle" valida as somas de verificação dos arquivos JAR do Gradle Wrapper presentes na árvore de origem.
4. A etapa "Criar com Gradle" executa o script wrapper `gradlew` para garantir que o seu código seja criado, o seu teste seja aprovado e que seja possível criar um pacote.

Os fluxos de trabalho inicial padrão são excelentes pontos de partida ao criar seu fluxo de trabalho de criação e teste, e você pode personalizar o fluxo de trabalho inicial para atender às necessidades do seu projeto.

{% data reusables.github-actions.example-github-runner %}

{% data reusables.github-actions.java-jvm-architecture %}

## Criar e testar seu código

Você pode usar os mesmos comandos usados localmente para criar e testar seu código.

O fluxo de tarbalho inicial executará a tarefa `criar` por padrão. Na configuração-padrão do Gradle, este comando irá baixar dependências, criar classes, executar testes e classes de pacotes em seu formato distribuível, como, por exemplo, um arquivo JAR.

Se você usa comandos diferentes para criar seu projeto ou se você desejar usar uma atividade diferente, você poderá especificá-los. Por exemplo, é possível que você deseje executar a tarefa `pacote` configurada no seu arquivo _ci.gradle_.

{% raw %}
```yaml{:copy}
steps:
  - uses: actions/checkout@v2
  - uses: actions/setup-java@v2
    with:
      java-version: '11'
      distribution: 'adopt'
  - name: Validate Gradle wrapper
    uses: gradle/wrapper-validation-action@e6e38bacfdf1a337459f332974bb2327a31aaf4b
  - name: Run the Gradle package task
    run: ./gradlew -b ci.gradle package
```
{% endraw %}

## Memorizar dependências

Ao usar executores hospedados em {% data variables.product.prodname_dotcom %}, você poderá armazenar em cache suas dependências para acelerar as execuções do seu fluxo de trabalho. Após a conclusão bem-sucedida, a sua cache do pacote do Gradle local será armazenada na infraestrutura do GitHub Actions. Para os fluxos de trabalho futuros, a cache será restaurada para que as dependências não precisem ser baixadas dos repositórios de pacotes remotos. Você pode armazenar dependências simplesmente usando a ação [`setup-java`](https://github.com/marketplace/actions/setup-java-jdk) ou pode usar a ação [`cache` ](https://github.com/actions/cache) para uma configuração mais avançada e personalizada.

{% raw %}
```yaml{:copy}
steps:
  - uses: actions/checkout@v2
  - name: Set up JDK 11
    uses: actions/setup-java@v2
    with:
      java-version: '11'
      distribution: 'adopt'
      cache: gradle
  - name: Validate Gradle wrapper
    uses: gradle/wrapper-validation-action@e6e38bacfdf1a337459f332974bb2327a31aaf4b
  - name: Build with Gradle
    run: ./gradlew build
  - name: Cleanup Gradle Cache
    # Remove some files from the Gradle cache, so they aren't cached by GitHub Actions.
    # Restoring these files from a GitHub Actions cache might cause problems for future builds.
    run: |
      rm -f ~/.gradle/caches/modules-2/modules-2.lock
      rm -f ~/.gradle/caches/modules-2/gc.properties
```
{% endraw %}

Este fluxo de trabalho salvará o conteúdo de seu cache local do pacote Gradle, localizado nos diretórios `.gradle/caches` e `.gradle/wrapper` do diretório inicial do executor. A chave de cache será o conteúdo da compilação com hash (incluindo o arquivo de propriedades do wrapper do Gradle). Portanto, qualquer alteração neles irá invalidar o cache.

## Empacotar dados do fluxo de trabalho como artefatos

Após a sua criação ter sido criada com sucesso e os seus testes aprovados, é possível que você deseje fazer o upload dos Java resultantes como um artefato de criação. Isso armazenará os pacotes criados como parte da execução do fluxo de trabalho e permitirá que você faça o download desses pacotes. Os artefatos podem ajudá-lo a testar e depurar os pull requests no seu ambiente local antes de serem mesclados. Para obter mais informações, consulte "[Dados recorrentes do fluxo de trabalho que usam artefatos](/actions/automating-your-workflow-with-github-actions/persisting-workflow-data-using-artifacts)".

De modo geral, o Gradle cria arquivos de saída como JARs, EARs ou WARs no diretório `build/libs`. Você pode fazer upload do conteúdo desse diretório usando a ação `upload-artefact`.

{% raw %}
```yaml{:copy}
steps:
  - uses: actions/checkout@v2
  - uses: actions/setup-java@v2
    with:
      java-version: '11'
      distribution: 'adopt'
  - name: Validate Gradle wrapper
    uses: gradle/wrapper-validation-action@e6e38bacfdf1a337459f332974bb2327a31aaf4b
  - run: ./gradlew build
  - uses: actions/upload-artifact@v2
    with:
      name: Package
      path: build/libs
```
{% endraw %}
