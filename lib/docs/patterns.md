# 🛠️ Padrões e Boas Práticas

**[⬅️ Voltar: Config](./config.md)** | **[➡️ Próximo: Guia Prático](./practical-guide.md)**

---

## 🎯 Padrões Fundamentais

Esta arquitetura utiliza diversos **padrões de design** e **boas práticas** modernas. Vamos entender cada um deles e como aplicar no seu dia a dia.

---

## ⚡ Result Pattern - Tratamento de Erros

### 🔍 O que é o Result Pattern?

O **Result Pattern** é uma abordagem **funcional** para tratamento de erros que substitui exceptions por tipos explícitos.

```dart
// ❌ Abordagem tradicional (com exceptions)
User login(String email, String password) {
  if (email.isEmpty) throw ValidationException('Email required');
  if (!isValidEmail(email)) throw ValidationException('Invalid email');
  // ...
  return user;
}

// ✅ Abordagem com Result Pattern
AsyncResult<User> login(String email, String password) async {
  if (email.isEmpty) return Failure(ValidationException('Email required'));
  if (!isValidEmail(email)) return Failure(ValidationException('Invalid email'));
  // ...
  return Success(user);
}
```

### 🎯 Vantagens do Result Pattern

| Tradicional (Exceptions)    | Result Pattern                    |
| --------------------------- | --------------------------------- |
| ❌ Erros implícitos         | ✅ Erros explícitos na assinatura |
| ❌ Fácil esquecer try/catch | ✅ Compilador força tratamento    |
| ❌ Stack unwinding custoso  | ✅ Performance superior           |
| ❌ Difícil composição       | ✅ Operações encadeáveis          |

### 💻 Uso Prático no Projeto

```dart
// No AuthRepository
AsyncResult<LoggedUser> login(Credentials credentials) async {
  return _authClientHttp
      .login(credentials)                    // Pode falhar
      .flatMap(_authLocalStorage.saveUser)   // Só executa se anterior deu certo
      .onSuccess(_streamController.add);     // Side effect apenas no sucesso
}

// Na UI
final result = await viewModel.loginCommand.execute(credentials);
result.fold(
  (error) => showError(error),
  (user) => navigateToHome(),
);
```

### 🔧 Operações Úteis

```dart
// Transformação
result.map((user) => user.name);

// Encadeamento
result.flatMap((user) => getProfile(user.id));

// Side effects
result.onSuccess((user) => analytics.trackLogin());
result.onFailure((error) => logger.error(error));

// Pattern matching
result.fold(
  (error) => handleError(error),
  (success) => handleSuccess(success),
);
```

---

## ❄️ Freezed Pattern - Imutabilidade

### 🔍 O que é o Freezed?

**Freezed** é uma biblioteca que gera código para criar **classes imutáveis** com recursos avançados.

### 📄 Exemplo: User Entity

```dart
@freezed
sealed class User with _$User {
  const factory User.basic({
    required int id,
    required String name,
    required String email,
  }) = BasicUser;

  const factory User.notLogged() = NotLoggedUser;

  const factory User.logged({
    required int id,
    required String name,
    required String email,
    required String token,
    required String refreshtoken,
  }) = LoggedUser;

  factory User.fromJson(Map<String, Object?> json) => _$UserFromJson(json);
}
```

### 🎯 O que o Freezed Gera Automaticamente

#### 1. **Imutabilidade**

```dart
final user = User.logged(id: 1, name: 'João', /* ... */);
// user.name = 'Pedro'; // ❌ Erro de compilação!
```

#### 2. **Igualdade Baseada em Valor**

```dart
final user1 = User.basic(id: 1, name: 'João', email: 'joao@test.com');
final user2 = User.basic(id: 1, name: 'João', email: 'joao@test.com');
print(user1 == user2); // ✅ true
```

#### 3. **copyWith para Atualizações**

```dart
final user = User.basic(id: 1, name: 'João', email: 'joao@test.com');
final updatedUser = user.copyWith(name: 'João Silva');
```

#### 4. **Pattern Matching**

```dart
// Type checking
if (user is LoggedUser) {
  print('Token: ${user.token}');
}

// When method (pattern matching)
final greeting = user.when(
  basic: (id, name, email) => 'Olá, $name!',
  notLogged: () => 'Por favor, faça login',
  logged: (id, name, email, token, refresh) => 'Bem-vindo de volta, $name!',
);
```

#### 5. **JSON Serialization**

```dart
// Para JSON
final json = user.toJson();

// De JSON
final user = User.fromJson(json);
```

### ✅ Vantagens da Imutabilidade

1. **Thread Safety**: Objetos imutáveis são seguros para concorrência
2. **Previsibilidade**: Não há mudanças inesperadas
3. **Debugging**: Mais fácil rastrear mudanças de estado
4. **Caching**: Hash codes estáveis permitem cache eficiente
5. **Functional Programming**: Facilita programação funcional

---

## ⚡ Command Pattern - Operações Assíncronas

### 🔍 O que são Commands?

**Commands** encapsulam operações assíncronas e gerenciam automaticamente seus estados.

```dart
class LoginViewmodel extends ChangeNotifier {
  late final loginCommand = Command1(_login);

  AsyncResult<LoggedUser> _login(Credentials credentials) async {
    return _authRepository.login(credentials);
  }
}
```

### 📊 Estados dos Commands

| Estado      | Quando                     | UI                |
| ----------- | -------------------------- | ----------------- |
| `isIdle`    | Antes da primeira execução | Estado inicial    |
| `isRunning` | Durante execução           | Loading spinner   |
| `isSuccess` | Após sucesso               | Mostrar resultado |
| `isFailure` | Após erro                  | Mostrar erro      |

### 💻 Uso na UI

```dart
ListenableBuilder(
  listenable: viewModel.loginCommand,
  builder: (context, _) {
    final command = viewModel.loginCommand.value;

    return ElevatedButton(
      onPressed: command.isRunning
          ? null
          : () => viewModel.loginCommand.execute(credentials),
      child: command.isRunning
          ? CircularProgressIndicator()
          : Text('Login'),
    );
  },
)
```

### 🎯 Vantagens dos Commands

1. **Estado Automático**: Loading/success/error gerenciados automaticamente
2. **Prevenção de Bugs**: Evita dupla execução
3. **UI Reativa**: Interface reage automaticamente aos estados
4. **Menos Boilerplate**: Reduz código repetitivo

---

## 🏗️ Repository Pattern - Abstração de Dados

### 🔍 Conceito

O **Repository Pattern** abstrai o acesso a dados, fornecendo uma interface uniforme independente da fonte.

```dart
// Interface (contrato)
abstract interface class AuthRepository {
  AsyncResult<LoggedUser> login(Credentials credentials);
  AsyncResult<Unit> logout();
  AsyncResult<LoggedUser> getUser();
  Stream<User> userObserver();
}

// Implementação
class RemoteAuthRepository implements AuthRepository {
  // Implementação específica para API
}

class LocalAuthRepository implements AuthRepository {
  // Implementação específica para dados locais
}
```

### 🎯 Vantagens

1. **Testabilidade**: Fácil criar mocks
2. **Flexibilidade**: Trocar implementações facilmente
3. **Separation of Concerns**: Lógica de acesso isolada
4. **Reusabilidade**: Mesma interface para diferentes fontes

---

## 🔄 Observer Pattern - Reatividade

### 🔍 Stream-based State Management

```dart
class RemoteAuthRepository implements AuthRepository {
  final _streamController = StreamController<User>.broadcast();

  @override
  Stream<User> userObserver() => _streamController.stream;

  @override
  AsyncResult<LoggedUser> login(Credentials credentials) {
    return _authClientHttp
        .login(credentials)
        .flatMap(_authLocalStorage.saveUser)
        .onSuccess(_streamController.add);  // Emite mudança
  }
}
```

### 💻 Observação na UI

```dart
class MainViewmodel extends ChangeNotifier {
  MainViewmodel(this._authRepository) {
    _userSubscription = _authRepository.userObserver().listen((user) {
      _user = user;
      notifyListeners();  // Propaga mudança para UI
    });
  }
}
```

---

## 🎯 MVVM Pattern - Separação UI/Lógica

### 🏗️ Estrutura

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    View     │    │  ViewModel  │    │    Model    │
│             │◄──►│             │◄──►│             │
│ Pages/      │    │ Commands    │    │ Entities/   │
│ Widgets     │    │ State       │    │ DTOs        │
└─────────────┘    └─────────────┘    └─────────────┘
```

### 📝 Responsabilidades

- **View**: Apresentação e captura de eventos
- **ViewModel**: Lógica de apresentação e estado
- **Model**: Dados e regras de negócio

---

## 🧪 Dependency Injection - Testabilidade

### 🔧 Configuração Flexível

```dart
// Produção
void setupProductionDependencies() {
  injector.addSingleton<AuthRepository>(RemoteAuthRepository.new);
}

// Desenvolvimento
void setupDevelopmentDependencies() {
  injector.addSingleton<AuthRepository>(MockAuthRepository.new);
}

// Testes
void setupTestDependencies() {
  injector.addSingleton<AuthRepository>(FakeAuthRepository.new);
}
```

---

## 📝 Validação com Lucid Validation

### 🔍 Centralização de Regras

```dart
class CredentialsValidator extends LucidValidator<Credentials> {
  CredentialsValidator() {
    ruleFor((c) => c.email, key: 'email')
        .notEmpty()
        .validEmail();

    ruleFor((c) => c.password, key: 'password')
        .notEmpty()
        .minLength(6)
        .mustHaveLowercase()
        .mustHaveUppercase()
        .mustHaveNumber()
        .mustHaveSpecialCharacter();
  }
}
```

### ✅ Vantagens

1. **Centralização**: Todas as regras em um local
2. **Reutilização**: Mesmo validador em múltiplos contextos
3. **Composição**: Validações complexas através de regras simples
4. **Internacionalização**: Mensagens customizáveis

---

## 🎨 Single Source of Truth (SSoT)

### 🔍 Conceito

**SSoT** significa ter **uma única fonte** autorizada para cada dado na aplicação.

### 💻 Implementação no Projeto

```dart
class RemoteAuthRepository implements AuthRepository {
  // ✅ SSoT para estado do usuário
  final _streamController = StreamController<User>.broadcast();

  @override
  AsyncResult<LoggedUser> login(Credentials credentials) {
    return _authClientHttp.login(credentials)
        .flatMap(_authLocalStorage.saveUser)    // Salva localmente
        .onSuccess(_streamController.add);      // Emite para observers
  }

  @override
  Stream<User> userObserver() => _streamController.stream;  // Fonte única
}
```

### 🎯 Benefícios

1. **Consistência**: Todos veem o mesmo estado
2. **Simplicidade**: Não há conflitos de dados
3. **Debugging**: Fácil rastrear origem dos dados
4. **Performance**: Evita sincronização complexa

---

## 📝 Resumo das Boas Práticas

### ✅ **Faça**

1. **Use Result Pattern** para tratamento explícito de erros
2. **Aplique Freezed** para entidades imutáveis
3. **Implemente Commands** para operações assíncronas
4. **Abstraia com Repositories** para acesso a dados
5. **Centralize validações** em validators específicos
6. **Mantenha SSoT** para cada tipo de dado
7. **Injete dependências** para baixo acoplamento

### ❌ **Evite**

1. **Exceptions** para controle de fluxo
2. **Mutabilidade** desnecessária em entidades
3. **State management manual** em operações async
4. **Acesso direto** a dados na UI
5. **Validações espalhadas** pelo código
6. **Múltiplas fontes** para o mesmo dado
7. **Acoplamento forte** entre camadas

---

## 💡 Próximos Passos

Agora que você entende todos os padrões, vamos ver como aplicar tudo isso na prática:

**[➡️ Próximo: Guia Prático de Implementação](./practical-guide.md)**

---

**[⬅️ Config](./config.md)** | **[🏠 Início](./README.md)** | **[➡️ Guia Prático](./practical-guide.md)**
