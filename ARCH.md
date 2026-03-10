# Arquitetura do Projeto (Feature-First)

Este documento descreve a arquitetura refatorada do projeto, orientada a funcionalidades (Feature-First). O objetivo dessa mudança foi escalar e organizar as responsabilidades das classes existentes que, embora corretas, encontravam-se fragmentadas em pastas puramente técnicas.

## Diagrama do Fluxo

A comunicação flui de forma unidirecional (UI → VM → Repo → DataSources), respeitando as fronteiras arquiteturais:

```mermaid
graph TD;
    UI[UI / Widgets / Pages] -->|Delega intenção e lê estado| VM[ViewModel];
    VM -->|Inicia regra de apresentação / negócio| Repo[Repository Contract];
    Repo_Impl[Repository Implementation] -.->|Implementa| Repo;
    Repo_Impl -->|Comunicação Externa| RemoteDS[Remote DataSource];
    Repo_Impl -->|Cache e Dados Locais| LocalDS[Local DataSource];

    classDef ui fill:#A8D5BA,stroke:#333,stroke-width:2px,color:#000;
    classDef vm fill:#F9D4D5,stroke:#333,stroke-width:2px,color:#000;
    classDef rep fill:#D3CBEB,stroke:#333,stroke-width:2px,color:#000;
    classDef data fill:#FFEBB5,stroke:#333,stroke-width:2px,color:#000;

    class UI ui;
    class VM vm;
    class Repo rep;
    class Repo_Impl rep;
    class RemoteDS data;
    class LocalDS data;
```

### Estrutura de Diretórios Resultante

Abaixo a representação da árvore de pastas e arquivos após a refatoração:

```text
lib/
  core/
    app_root.dart
    app_errors.dart
  features/
    todos/
      data/
        datasources/
          todo_remote_datasource.dart
          todo_local_datasource.dart
        models/
          todo_model.dart
        repositories/
          todo_repository_impl.dart
      domain/
        entities/
          todo.dart
        repositories/
          todo_repository.dart
      presentation/
        pages/
          todos_page.dart
        viewmodels/
          todo_viewmodel.dart
        widgets/
          add_todo_dialog.dart
  main.dart
```

## Justificativa da Estrutura

A estrutura original misturava todos os arquivos referentes a várias partes do sistema em pastas genéricas como `screens`, `models` e `viewmodels`.

Para uma aplicação robusta, consolidamos a arquitetura **Feature-First**, conforme exemplificado pela feature `todos`:

1.  **`lib/core/`**: Elementos transversais que configuram a aplicação ou são amplamente reutilizados. Por exemplo, mensagens de erro (`app_errors.dart`) e a raiz que invoca o `MaterialApp` (`app_root.dart`).
2.  **`lib/features/todos/domain/`**: Coração da funcionalidade. Nele reside a entidade `todo.dart` e o contrato abstrato `todo_repository.dart`. Esta camada é independente e ignorante de qualquer framework, como Flutter, HTTP ou SharedPreferences.
3.  **`lib/features/todos/data/`**: Responsável pela orquestração dos dados.
    *   **DataSources**: `todo_remote_datasource.dart` (para HTTP) e `todo_local_datasource.dart` (SharedPreferences).
    *   **Models (DTOs)**: O `todo_model.dart` estende a entidade e sabe fazer o parser de JSON (conversão `fromJson` / `toJson`).
    *   **Repositories (`todo_repository_impl.dart`)**: Implementa o contrato de Domain e contém a inteligência de escolha e mapeamento entre fontes locais e remotas.
4.  **`lib/features/todos/presentation/`**: Acopla o visual e a gerência do seu estado. Contém os **Widgets**, **Pages** e o **ViewModel**.

## Decisões de Responsabilidade

*   **A UI não pode chamar HTTP nem SharedPreferences diretamente**: Essa responsabilidade foi abstraída para os `DataSources`, removendo a lógica da camada de apresentação para evitar acoplamento do framework visual aos detalhes de infraestrutura ou SDKs.
*   **O ViewModel não pode conhecer Widgets / BuildContext (exceto mensagens via estado)**: O `TodoViewModel` possui variáveis (`isLoading`, `errorMessage`, `items`) e expõe métodos. Ele nunca importa `material.dart` ou lida com navegação direta utilizando contexto. Todas as reações UI-driven acontecem passivamente ouvindo as atualizações disparadas por `notifyListeners()`.
*   **O Repository deve centralizar a escolha entre remoto/local**: O fluxo de dados deve passar por classes como `TodoRepositoryImpl`. Ela coordena as requisições (`_remote.fetchTodos`) e faz interações locais necessárias (ex: atualizando o tempo do último sincronismo com `_local.saveLastSync(now)`). Para o `ViewModel`, a origem real dos dados é um detalhe invisível e transparente.

## Respostas Adicionais

### Onde ficou a validação?
A validação inicial/mínima (como impedir que um *todo* seja adicionado com o texto vazio) fica no **ViewModel** (`todo_viewmodel.dart`), onde as regras de apresentação rejeitam *inputs* inválidos alterando o estado com uma `errorMessage` antes de delegar para o Repository. Validações adicionais atreladas a tipos ocorrem implicitamente no **Model** (que faz o casting de tipos do negócio).

### Onde ficou o parsing JSON?
O parsing (desserialização de JSON em objetos e vice-versa) ficou isolado na camada de *Data* usando **Models** (`todo_model.dart`). O *DataSource* (`todo_remote_datasource.dart`) apenas decodifica a *String* oriunda da API num *Map* genérico (`jsonDecode`) e delega ao Model a responsabilidade de converter o Map para uma Entidade válida do domínio, isolando a regra de negócio da estrutura que não sabe o que é JSON.

### Como você tratou erros?
O tratamento de erros ocorre de forma combinada em camadas:
1.  **DataSources** lançam `Exceptions` sempre que ocorre um erro técnico de comunicação ou o `StatusCode HTTP` é inválido.
2.  O **Repository** tipicamente propaga os erros ou resolve *fallbacks* (como acessar cache no armazenamento em caso de falha).
3.  O **ViewModel** captura as falhas centralmente em blocos `try/catch` ao invocar o repositório (`await _repo.fetchTodos(...)`), manipulando a UI apenas com a reatividade do estado e exibindo informações pela variável `errorMessage`. Em casos pontuais, como a alteração de status (`toggleCompleted`), implementa-se de forma **Otimista**: a UI é atualizada de imediato e sofre **rollback** caso a requisição lance uma exceção.
