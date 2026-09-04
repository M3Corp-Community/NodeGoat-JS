# Documentação dos pipelines Veracode

Este documento descreve as etapas dos templates de pipeline Veracode para Azure DevOps em ambiente Linux:

- `veracode-ci-linux.yml`
- `veracode-ci-linux.net.yml`

O objetivo dos templates é automatizar a preparação dos pacotes de análise e executar as validações de segurança da Veracode durante o processo de CI, incluindo SAST, SCA e IaC.

## Visão geral

Os dois templates possuem um único stage chamado `Veracode`. Esse stage concentra as etapas responsáveis por preparar o artefato da aplicação, enviar o pacote para a plataforma Veracode e executar análises complementares no próprio pipeline.

A principal diferença entre os templates está na forma como o pacote ZIP é criado. O template Linux genérico monta o pacote com base em extensões de arquivos relevantes para análise de código-fonte. Já o template Linux .NET monta um pacote otimizado com assemblies, símbolos e arquivos auxiliares necessários para uma análise mais precisa de aplicações .NET.

## Pipeline `veracode-ci-linux.yml`

Este template é indicado para aplicações em que o pacote de análise pode ser montado a partir dos arquivos de código-fonte do repositório.

### Criação do pacote para análise

A primeira etapa gera o arquivo ZIP que será enviado para a Veracode.

O pacote é criado incluindo apenas extensões relevantes para análise, como arquivos `.js`, `.ts`, `.jsx`, `.tsx`, `.java`, `.py`, `.php`, `.html`, `.json`, `.lock`, entre outros.

Ao mesmo tempo, são excluídos diretórios e arquivos que não agregam valor para a análise, como:

- `node_modules`
- `.git`
- `dist`
- arquivos relacionados a testes, como `test`, `Test`, `TEST`, `teste` e `Teste`

Esse ajuste evita o envio de arquivos desnecessários ou muito volumosos para a Veracode. Um pacote menor e mais direcionado reduz o tempo de upload, diminui ruídos na análise e ajuda a plataforma a focar nos arquivos que realmente representam o código da aplicação.

### Download do Veracode Wrapper

Após a criação do pacote, o pipeline baixa a versão mais recente do `Veracode Java API Wrapper` a partir do Maven.

Esse wrapper é utilizado para interagir com a plataforma Veracode por linha de comando. Neste pipeline, ele é usado principalmente para executar o processo de `uploadandscan`, responsável por enviar o pacote e iniciar a análise estática na plataforma.

### Veracode Upload & Scan

A etapa `Veracode U&S` envia o ZIP gerado para a Veracode e inicia a análise.

O comportamento varia conforme a branch:

- Na branch `main`, o scan é enviado para o perfil principal da aplicação.
- Em outras branches, o pipeline cria ou utiliza uma sandbox com o nome da branch.

Essa separação permite manter a branch principal como referência oficial da aplicação, enquanto branches de desenvolvimento podem ser analisadas em sandboxes sem impactar diretamente os resultados do perfil principal.

A opção `-createprofile true` permite criar automaticamente o perfil da aplicação caso ele ainda não exista. A opção `-deleteincompletescan 2` remove scans incompletos antigos, evitando bloqueios causados por análises anteriores que não foram finalizadas corretamente.

No template Linux genérico, também é utilizado o parâmetro `-teams "$(veracodeTeams)"`, que associa o perfil da aplicação ao time configurado na Veracode.

A etapa utiliza `continueOnError: true`, permitindo que o pipeline continue executando as demais análises mesmo que o Upload & Scan retorne erro.

### Download do Veracode CLI

Depois do Upload & Scan, o pipeline instala o Veracode CLI.

O CLI é utilizado nas etapas seguintes, principalmente para a análise de infraestrutura como código e para a execução do `static scan` diretamente no pipeline.

### Veracode SCA

A etapa `Veracode SCA` executa a análise de composição de software, voltada para dependências open source.

Ela utiliza o script da Veracode/SRC:CLR com os seguintes parâmetros:

- `--update-advisor`, para trazer recomendações de atualização.
- `--allow-dirty`, para permitir a execução mesmo com alterações locais no workspace.
- `--recursive`, para avaliar projetos e subdiretórios de forma recursiva.

Essa análise complementa o SAST, pois identifica riscos em bibliotecas de terceiros, versões vulneráveis e possíveis recomendações de correção.

### Veracode IaC

A etapa `Veracode IaC` executa a análise de infraestrutura como código utilizando o Veracode CLI.

Antes do scan, o pipeline renomeia temporariamente a pasta do repositório para o nome da definição do pipeline, representado por `$(Build.DefinitionName)`. Isso ajuda a padronizar a identificação do projeto durante a execução da análise.

Em seguida, o pipeline executa o scan do diretório:

```bash
./veracode scan --type directory --source "$(pwd)" --format table
```

Ao final, o nome original da pasta é restaurado. Essa etapa permite identificar problemas de configuração em arquivos de infraestrutura, como definições de cloud, containers, manifests e outros artefatos suportados pela Veracode.

### Veracode SAST

A última etapa executa o `static scan` usando o pacote ZIP criado anteriormente:

```bash
./veracode static scan "$(caminhoPacote)"
```

Esse scan realiza uma análise estática do pacote preparado, servindo como validação adicional no próprio pipeline. Dessa forma, o fluxo combina o envio para a plataforma Veracode com uma execução via CLI, ampliando a cobertura das verificações automatizadas.

## Pipeline `veracode-ci-linux.net.yml`

Este template segue a mesma estrutura geral do pipeline Linux, mas adapta a etapa de empacotamento para aplicações .NET. A principal diferença está na forma como o ZIP é montado.

As etapas de download do wrapper, Upload & Scan, instalação do CLI, SCA, IaC e SAST seguem o mesmo objetivo do template anterior. A diferença mais relevante está na etapa inicial, chamada `Preparar pacote Veracode`.

### Preparação do pacote Veracode para .NET

No template .NET, o ZIP não é criado a partir de arquivos de código-fonte genéricos. Em vez disso, o pipeline procura arquivos `.dll` e monta um pacote otimizado para análise binária da Veracode.

A lógica considera apenas DLLs relevantes e exclui itens que normalmente não devem ser enviados, como:

- DLLs da Microsoft
- DLLs de testes, como `UnitTest`, `Xunit` e `Test`
- diretórios `obj`
- diretórios `ref` e `refint`
- `node_modules`
- `.git`
- `TestResults`

Além disso, o pipeline só copia uma DLL quando existe o arquivo `.pdb` correspondente.

Esse ponto é importante porque, em aplicações .NET, os arquivos `.pdb` carregam símbolos e metadados que ajudam a Veracode a analisar melhor o código compilado. Sem esses arquivos, a qualidade da análise pode ser reduzida.

Para cada DLL válida, o pipeline também copia arquivos auxiliares relacionados, quando existirem:

- `.pdb`
- `.deps.json`
- `.runtimeconfig.json`
- `web.config`

Esses arquivos ajudam a preservar o contexto de execução da aplicação, suas dependências e configurações, permitindo que a Veracode tenha uma visão mais completa do artefato analisado.

O pacote é criado dentro de um diretório temporário usando `mktemp -d`. Essa abordagem evita que arquivos desnecessários do workspace sejam incluídos por acidente. Ao final, o conteúdo selecionado é compactado no caminho definido por `$(caminhoPacote)`.

## Motivo da otimização dos ZIPs

A otimização dos ZIPs é um ponto central dos dois templates.

No pipeline Linux genérico, o pacote é filtrado por extensões e exclusões de diretórios. No pipeline Linux .NET, o pacote é filtrado por assemblies e arquivos auxiliares necessários para análise.

Essa abordagem evita enviar para a Veracode arquivos que não contribuem para o resultado, como dependências instaladas, artefatos intermediários de build, histórico Git, diretórios de teste e arquivos gerados.

Com isso, o pipeline busca:

- reduzir o tamanho do pacote enviado;
- diminuir o tempo de upload e processamento;
- evitar ruídos na análise;
- melhorar a precisão dos resultados;
- enviar somente os artefatos necessários para a tecnologia analisada;
- manter o processo mais estável em aplicações grandes.

Em resumo, os ajustes nos ZIPs foram feitos para garantir que a Veracode receba pacotes menores, mais limpos e mais aderentes ao tipo de aplicação analisada. Isso melhora a eficiência do scan e reduz a chance de resultados poluídos por arquivos que não fazem parte do escopo real da aplicação.

