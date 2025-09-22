# ⚙️ Configuração e Injeção de Dependências

**[⬅️ Voltar: UI](./ui.md)** | **[➡️ Próximo: Padrões](./patterns.md)**

---

## 🎯 O que é Injeção de Dependências?

**Injeção de Dependências (DI)** é um padrão que permite **fornecer dependências** para uma classe ao invés de ela criar suas próprias dependências. Isso promove:

- ✅ **Baixo acoplamento**
- ✅ **Alto reuso**
- ✅ **Fácil testabilidade**
- ✅ **Configuração centralizada**

> 💡 **Princípio**: Classes não devem instanciar suas dependências - elas devem recebê-las.

---

## 📁 Estrutura de Configuração

```
config/
└── dependencies.dart      # Configuração de todas as dependências
```

---

## 🏗️ Configuração Atual do Projeto

### 📄 dependencies.dart - Setup Centralizado

```dart
import 'package:auto_injector/auto_injector.dart';
// ... imports

final injector = AutoInjector();

void setupDependencies() {
  // Data Layer
  injector.addSingleton<AuthRepository>(RemoteAuthRepository.new);
  injector.addInstance(Dio());
  injector.addSingleton(ClientHttp.new);
  injector.addSingleton(LocalStorage.new);
  injector.addSingleton(AuthClientHttp.new);
  injector.addSingleton(AuthLocalStorage.new);

  // UI Layer
  injector.addSingleton(LoginViewmodel.new);
  injector.addSingleton(LogoutViewmodel.new);
  injector.addSingleton(MainViewmodel.new);
}
```

#### 🎯 Estrutura da Configuração

1. **Instância Global**: `final injector = AutoInjector()`
2. **Setup Único**: `setupDependencies()` chamado no `main()`
3. **Ordem Importante**: Dependências base primeiro, depois compostas

---

## 🔍 Tipos de Registro

### 1. **addSingleton** - Instância Única

```dart
injector.addSingleton<AuthRepository>(RemoteAuthRepository.new);
```

- ✅ **Uma instância** para toda a aplicação
- ✅ **Ideal para**: Repositories, Services, ViewModels globais
- ✅ **Performance**: Evita criação desnecessária de objetos

### 2. **addInstance** - Instância Pré-criada

```dart
injector.addInstance(Dio());
```

- ✅ **Objeto já instanciado** adicionado ao container
- ✅ **Ideal para**: Configurações específicas, objetos terceiros
- ✅ **Controle Total**: Configuração antes da injeção

### 3. **addTransient** - Nova Instância Sempre

```dart
injector.addTransient(SomeClass.new);
```

- ✅ **Nova instância** a cada solicitação
- ✅ **Ideal para**: Objetos sem estado, temporários
- ✅ **Isolamento**: Cada uso tem sua própria instância

---

## 🔗 Gráfico de Dependências

```mermaid
graph TD
    A[main.dart] --> B[setupDependencies()]

    B --> C[Dio]
    B --> D[LocalStorage]
    B --> E[ClientHttp]

    C --> E
    D --> F[AuthLocalStorage]
    E --> G[AuthClientHttp]

    F --> H[RemoteAuthRepository]
    G --> H

    H --> I[AuthRepository Interface]

    I --> J[LoginViewmodel]
    I --> K[LogoutViewmodel]
    I --> L[MainViewmodel]

    J --> M[LoginPage]
    K --> N[LogoutButton]
    L --> O[MainApp]
```

---

## 🚀 Fluxo de Inicialização

### 📄 main.dart - Entry Point

```dart
void main() {
  setupDependencies();  // 1. Configura todas as dependências
  runApp(const MainApp());  // 2. Inicia a aplicação
}
```

### 🔄 Ordem de Registro

1. **Dependências Base** (Dio, LocalStorage)
2. **Services Compostos** (ClientHttp, AuthLocalStorage, AuthClientHttp)
3. **Repositories** (RemoteAuthRepository)
4. **ViewModels** (LoginViewmodel, LogoutViewmodel, MainViewmodel)

---

## 💻 Como Usar nas Classes

### 📱 Nas Pages/Widgets

```dart
class LoginPage extends StatefulWidget {
  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final viewModel = injector.get<LoginViewmodel>();  // ⚡ Injeção

  // resto do código...
}
```

### 🧠 Nos ViewModels

```dart
class LoginViewmodel extends ChangeNotifier {
  final AuthRepository _authRepository;

  // ⚡ Dependência recebida via construtor
  LoginViewmodel(this._authRepository);

  // resto do código...
}
```

### 💾 Nos Repositories

```dart
class RemoteAuthRepository implements AuthRepository {
  final AuthLocalStorage _authLocalStorage;
  final AuthClientHttp _authClientHttp;

  // ⚡ Dependências recebidas via construtor
  RemoteAuthRepository(this._authLocalStorage, this._authClientHttp);

  // resto do código...
}
```

---

## 🎯 Vantagens da Configuração Atual

### ✅ **Testabilidade**

```dart
// Fácil criar mocks para testes
void setupTestDependencies() {
  injector.addSingleton<AuthRepository>(MockAuthRepository.new);
  // outras dependências mock
}
```

### ✅ **Flexibilidade de Ambiente**

```dart
void setupDependencies() {
  if (isProduction) {
    injector.addSingleton<AuthRepository>(RemoteAuthRepository.new);
  } else {
    injector.addSingleton<AuthRepository>(LocalAuthRepository.new);
  }
}
```

### ✅ **Singleton Pattern Automático**

```dart
// Sempre a mesma instância
final viewModel1 = injector.get<LoginViewmodel>();
final viewModel2 = injector.get<LoginViewmodel>();
print(viewModel1 == viewModel2); // true
```

---

## 🔧 AutoInjector - Features

### 📊 Resolução Automática

```dart
// AutoInjector resolve dependências automaticamente
injector.addSingleton(RemoteAuthRepository.new);

// Equivale a:
injector.addSingleton<AuthRepository>(() {
  final authLocalStorage = injector.get<AuthLocalStorage>();
  final authClientHttp = injector.get<AuthClientHttp>();
  return RemoteAuthRepository(authLocalStorage, authClientHttp);
});
```

### 🔍 Type Safety

```dart
// Tipo específico
final authRepo = injector.get<AuthRepository>();

// Compilador garante que existe
final loginVM = injector.get<LoginViewmodel>(); // ✅

// Erro em tempo de compilação se não registrado
final unknown = injector.get<UnknownClass>(); // ❌
```

---

## 🏗️ Padrões Alternativos

### 1. **Injectable + GetIt**

```dart
@Injectable()
class LoginViewmodel {
  final AuthRepository authRepository;
  LoginViewmodel(this.authRepository);
}

@InjectableInit()
void configureDependencies() => getIt.init();
```

### 2. **Provider Pattern**

```dart
MultiProvider(
  providers: [
    Provider<AuthRepository>(create: (_) => RemoteAuthRepository()),
    ChangeNotifierProvider<LoginViewmodel>(
      create: (context) => LoginViewmodel(context.read<AuthRepository>()),
    ),
  ],
  child: MyApp(),
)
```

### 3. **Riverpod**

```dart
final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return RemoteAuthRepository();
});

final loginViewmodelProvider = ChangeNotifierProvider<LoginViewmodel>((ref) {
  return LoginViewmodel(ref.read(authRepositoryProvider));
});
```

---

## 📝 Boas Práticas de DI

### ✅ **Faça**

1. **Registre dependências base primeiro**

   ```dart
   injector.addInstance(Dio());  // Primeiro
   injector.addSingleton(ClientHttp.new);  // Depois
   ```

2. **Use interfaces para abstrações**

   ```dart
   injector.addSingleton<AuthRepository>(RemoteAuthRepository.new);
   ```

3. **Centralize configuração em um arquivo**

   ```dart
   // dependencies.dart
   void setupDependencies() { ... }
   ```

4. **Use singletons para state management**
   ```dart
   injector.addSingleton(MainViewmodel.new);
   ```

### ❌ **Evite**

1. **Registros espalhados pela aplicação**
2. **Dependências circulares**
3. **Over-engineering com DI complexo**
4. **Injeção em construtores muito profundos**

---

## 🧪 Configuração para Testes

### 📝 Exemplo de Setup de Teste

```dart
// test_dependencies.dart
void setupTestDependencies() {
  injector.clearInstances();

  // Mocks para teste
  injector.addSingleton<AuthRepository>(MockAuthRepository.new);
  injector.addSingleton<LocalStorage>(MockLocalStorage.new);

  // ViewModels reais
  injector.addSingleton(LoginViewmodel.new);
}

// Em um teste
setUp(() {
  setupTestDependencies();
});
```

---

## 🎯 Troubleshooting Comum

### ❌ **Erro: Dependency not found**

```dart
// Problema: Dependência não registrada
final missing = injector.get<UnregisteredClass>(); // Erro!

// Solução: Registrar antes de usar
injector.addSingleton(UnregisteredClass.new);
```

### ❌ **Erro: Circular dependency**

```dart
// Problema: A depende de B, B depende de A
class A { A(B b); }
class B { B(A a); }

// Solução: Quebrar ciclo com interface ou event bus
```

### ❌ **Erro: Wrong order of registration**

```dart
// Problema: Dependência usada antes de ser registrada
injector.addSingleton(ClassThatNeedsX.new);  // Erro!
injector.addSingleton(X.new);

// Solução: Registrar dependências na ordem correta
injector.addSingleton(X.new);  // Primeiro
injector.addSingleton(ClassThatNeedsX.new);  // Depois
```

---

## 💡 Próximos Passos

Agora que você entende como configurar dependências, vamos ver os padrões e práticas utilizados:

**[➡️ Próximo: Padrões e Boas Práticas](./patterns.md)**

---

**[⬅️ UI Layer](./ui.md)** | **[🏠 Início](./README.md)** | **[➡️ Padrões](./patterns.md)**
