# Flutter State Management: GetX, BLoC, Cubit, Riverpod e Provider

> Um guia prático para entender as principais soluções de gerenciamento de estado no Flutter, comparar suas diferenças e escolher a mais adequada para cada projeto.

## Índice

- [1. O que é gerenciamento de estado?](#1-o-que-é-gerenciamento-de-estado)
- [2. Conhecendo as principais soluções](#2-conhecendo-as-principais-soluções)
- [3. Comparando com um exemplo de login](#3-comparando-com-um-exemplo-de-login)
  - [GetX](#getx)
  - [Cubit](#cubit)
  - [BLoC](#bloc)
- [4. Comparação prática](#4-comparação-prática)
- [5. Como isso se encaixa na arquitetura](#5-como-isso-se-encaixa-na-arquitetura)
- [6. Qual solução escolher?](#6-qual-solução-escolher)
- [7. E para quem já trabalha com GetX?](#7-e-para-quem-já-trabalha-com-getx)
- [8. Conclusão](#8-conclusão)

---

## 1. O que é gerenciamento de estado?

A forma mais fácil de entender é pensar que **todas essas soluções tentam resolver o mesmo problema: como atualizar a interface quando os dados do aplicativo mudam**.

Imagine uma tela de login. O usuário digita e-mail e senha, toca em **Entrar**, o aplicativo chama a API e precisa mostrar um carregamento, exibir um erro ou navegar para a próxima tela.

Cada solução faz isso de uma maneira diferente.

## 2. Conhecendo as principais soluções

### GetX

**Prático e direto**

O GetX reúne gerenciamento de estado, injeção de dependências e navegação em uma mesma ferramenta.

**Como funciona:** o Controller guarda os dados, e a interface observa as mudanças, geralmente com `Obx`.

**Pontos fortes**
- Pouco código para começar.
- Estado, navegação e dependências integrados.
- Fluxo de desenvolvimento rápido.

**Ponto de atenção:** sem uma organização consistente, controllers podem acumular responsabilidades e dependências globais podem ficar difíceis de rastrear.

### BLoC

**Fluxo bem definido**

O BLoC separa as ações do usuário das mudanças de estado. Isso ajuda a manter regras de negócio previsíveis e a testar o comportamento do aplicativo.

**Como funciona:** a interface envia um evento, o BLoC processa esse evento e emite um novo estado.

### Cubit

**Mais simples que BLoC**

O Cubit faz parte da mesma família do BLoC, mas elimina a necessidade de criar eventos separados.

**Como funciona:** a interface chama um método, como `login()`, e o Cubit emite um novo estado.

### Riverpod

**Flexível e testável**

O Riverpod ajuda a gerenciar estados e dependências sem depender diretamente do `BuildContext`.

**Como funciona:** a interface observa um provider, que disponibiliza ou calcula um valor e pode executar operações assíncronas.

### Provider

**Simples e integrado ao Flutter**

O Provider é bastante utilizado para disponibilizar objetos e observar mudanças. Frequentemente trabalha junto com `ChangeNotifier`.

**Como funciona:** um objeto altera seus dados e notifica os widgets para reconstruí-los.

### `setState` e `ValueNotifier`

**Soluções nativas para necessidades localizadas**

São recursos do próprio Flutter para controlar mudanças menores, sem precisar adotar uma solução completa de gerenciamento de estado.

**Como funcionam:** você altera um valor e informa ao Flutter que a interface precisa ser atualizada.

---

## 3. Comparando com um exemplo de login

Imagine que a tela de login precisa lidar com três situações:

- `Loading`: a requisição está acontecendo.
- `Success`: o login deu certo.
- `Error`: ocorreu um erro.

A diferença entre as ferramentas está em como você representa e produz esses estados.

### GetX

O Controller pode manter variáveis observáveis:

```dart
class LoginController extends GetxController {
  final isLoading = false.obs;
  final errorMessage = ''.obs;

  Future<void> login() async {
    isLoading.value = true;
    errorMessage.value = '';

    try {
      await repository.login();

      Get.offAllNamed('/home');
    } catch (e) {
      errorMessage.value = 'Falha ao entrar';
    } finally {
      isLoading.value = false;
    }
  }
}
```

Na interface, o `Obx` observa o valor de `isLoading`:

```dart
Obx(() {
  if (controller.isLoading.value) {
    return const CircularProgressIndicator();
  }

  return ElevatedButton(
    onPressed: controller.login,
    child: const Text('Entrar'),
  );
});
```

**A ideia:** você altera uma variável observável, e o `Obx` atualiza a parte da interface que depende dela.

> **Observação:** o exemplo é ilustrativo. `repository` precisa estar definido ou ser injetado no Controller.

### Cubit

Primeiro, definimos os estados possíveis:

```dart
sealed class LoginState {}

class LoginInitial extends LoginState {}

class LoginLoading extends LoginState {}

class LoginSuccess extends LoginState {}

class LoginError extends LoginState {
  final String message;

  LoginError(this.message);
}
```

Depois, o Cubit controla as transições:

```dart
class LoginCubit extends Cubit<LoginState> {
  final LoginRepository repository;

  LoginCubit(this.repository) : super(LoginInitial());

  Future<void> login() async {
    emit(LoginLoading());

    try {
      await repository.login();
      emit(LoginSuccess());
    } catch (e) {
      emit(LoginError('Falha ao entrar'));
    }
  }
}
```

A interface observa o estado emitido pelo Cubit, normalmente com `BlocBuilder`, e pode reagir a sucessos e erros usando `BlocListener`.

**A ideia:** em vez de alterar várias variáveis independentes, você emite um estado que representa a situação atual da tela.

> **Observação:** o exemplo pressupõe que os pacotes `bloc` e/ou `flutter_bloc` estejam instalados e que `LoginRepository` esteja definido.

### BLoC

O BLoC usa estados como o Cubit, mas acrescenta uma camada de eventos.

O fluxo é:

```text
Interface
   │
   │ Usuário toca em "Entrar"
   ▼
LoginEvent
   │
   │ Representa a ação solicitada
   ▼
LoginBloc
   │
   │ Processa o evento e chama o repositório
   ▼
LoginLoading → LoginSuccess
   │
   ▼
Interface atualizada
```

O BLoC também consegue executar o login. A diferença é que a ação é representada explicitamente por um evento, como `LoginSubmitted`.

**Resumindo:**
- No Cubit, você chama `login()`.
- No BLoC, você adiciona um evento, como `LoginSubmitted`.

---

## 4. Comparação prática

| Característica | GetX | Cubit | BLoC | Riverpod | Provider |
|---|---|---|---|---|---|
| Facilidade inicial | Alta | Média | Média/baixa | Média | Alta |
| Quantidade de código | Baixa | Média | Maior | Variável | Baixa/média |
| Eventos separados | Não exige | Não | Sim | Não exige | Não exige |
| Injeção de dependências | Integrada | Externa ou complementar | Externa ou complementar | Integrada ao sistema de providers | Composição via providers |
| Navegação integrada | Sim | Não | Não | Não | Não |
| Testes unitários | Sim | Sim | Sim | Sim | Sim |
| Controle explícito dos estados | Variável | Alto | Alto | Alto, conforme a abordagem | Depende da implementação |

> Essas comparações são tendências, não limitações absolutas. Todas podem ser usadas em projetos bem estruturados e testáveis.

---

## 5. Como isso se encaixa na arquitetura?

Uma distinção importante: **GetX, BLoC, Cubit, Riverpod e Provider não são arquiteturas completas por si só**. São ferramentas que ajudam a organizar o estado e, em alguns casos, as dependências.

Você pode combinar qualquer uma delas com uma arquitetura organizada por funcionalidades (*feature-first*).

Por exemplo, uma estrutura de login usando GetX poderia ser:

```text
features/
└── login/
    ├── data/
    │   └── login_repository.dart
    └── presentation/
        ├── controller/       # GetX
        ├── bindings/         # GetX
        └── pages/
            └── login_page.dart
```

Se o projeto usar Cubit ou BLoC, a pasta `controller/` pode ser substituída por `cubit/` ou `bloc/`, conforme a convenção escolhida.

O mais importante é separar as responsabilidades:

- **Page:** desenha a interface.
- **Controller, Cubit ou Bloc:** controla o estado e coordena as ações.
- **Repository:** conversa com a API e fornece os dados.
- **Model:** representa os dados do domínio ou da API.

Essa separação é possível com todas essas ferramentas.

---

## 6. Qual solução escolher?

### GetX — produtividade e simplicidade

Bom para desenvolver rapidamente e ter estado, dependências e navegação integrados. Exige disciplina para evitar controllers com responsabilidades demais e dependências globais difíceis de rastrear.

### Cubit — equilíbrio entre simplicidade e organização

Uma boa escolha quando você quer estados explícitos, testes claros e menos cerimônia que o BLoC tradicional.

### BLoC — fluxos complexos e explícitos

Útil quando diferentes ações e eventos precisam ser identificados, rastreados e testados de forma rigorosa, especialmente em equipes grandes.

### Riverpod — gerenciamento flexível de estado e dependências

Interessante para aplicações que precisam de composição de dependências, estado assíncrono e testes isolados.

### Provider — integração simples com o ecossistema Flutter

Uma boa opção para projetos que precisam de uma solução familiar e relativamente leve, sobretudo quando já usam `ChangeNotifier`.

### `setState` / `ValueNotifier` — simplicidade nativa

Funcionam bem para estados locais e pequenos. Nem toda tela precisa de uma biblioteca adicional.

---

## 7. E para quem já trabalha com GetX?

Não é necessário migrar de GetX para BLoC ou Riverpod apenas porque são alternativas conhecidas.

Em um aplicativo com várias funcionalidades — como autenticação, empresas, recebíveis e recargas —, o ganho de qualidade normalmente vem primeiro de:

1. Manter a lógica fora das Pages.
2. Separar Controllers dos Repositories.
3. Definir estados de carregamento, sucesso e erro de maneira consistente.
4. Organizar as dependências por funcionalidade.
5. Criar testes para regras de negócio e fluxos importantes.

É possível fazer tudo isso com GetX.

Se quiser aprender outra abordagem, **comece pelo Cubit e depois estude BLoC**. Assim, você entende primeiro o gerenciamento por estados e depois aprende o modelo adicional de eventos.

---

## 8. Conclusão

A melhor ferramenta não é necessariamente a que tem mais recursos, mas aquela que permite que a equipe **entenda, teste e mantenha o código com facilidade**.

Em resumo:

- **GetX:** rapidez e recursos integrados.
- **Cubit:** estados explícitos com menos cerimônia.
- **BLoC:** eventos e estados com fluxo bem definido.
- **Riverpod:** composição flexível de estado e dependências.
- **Provider:** solução simples e familiar.
- **`setState` / `ValueNotifier`:** ótimos para necessidades locais.

Antes de trocar de ferramenta, avalie a organização atual, a complexidade do aplicativo, a experiência da equipe e a facilidade de testar as regras de negócio.
