# 🏗️ Resumo Completo da Arquitetura Flutter

Este documento serve como referência completa para a arquitetura recomendada no desenvolvimento de aplicações Flutter. Ele contém informações detalhadas sobre cada camada, padrões, boas práticas e instruções passo a passo para desenvolvedores juniores.

---

## 📚 Referências Oficiais Flutter

- [Flutter Documentation](https://docs.flutter.dev/)
- [Flutter Architectural Overview](https://docs.flutter.dev/resources/architectural-overview)
- [Flutter App Architecture Guide](https://docs.flutter.dev/development/data-and-backend/state-mgmt/options)
- [Package Management](https://docs.flutter.dev/development/packages-and-plugins/using-packages)

---

## 📂 Estrutura Geral da Arquitetura

A arquitetura é dividida em **4 camadas principais** seguindo os princípios de **Clean Architecture**:

```
lib/
├── 📂 domain/          # Regras de negócio e entidades (CORE)
│   ├── entities/       # Modelos fundamentais (User, Product, etc.)
│   ├── dtos/          # Objetos de transferência de dados
│   └── validators/    # Regras de validação de negócio
├── 📂 data/            # Acesso a dados e implementações
│   ├── repositories/  # Implementações dos contratos
│   ├── services/      # Acesso a APIs e storage
│   └── exceptions/    # Tratamento de erros
├── 📂 ui/              # Interface do usuário (MVVM)
│   ├── auth/         # Features de autenticação
│   ├── home/         # Página principal
│   └── splash/       # Tela inicial
└── ⚙️ config/          # Configurações e dependências
    └── dependencies.dart # Injeção de dependências
```

### 🎯 Princípios Fundamentais

1. **Separação de Responsabilidades**: Cada camada tem função específica
2. **Inversão de Dependência**: Camadas superiores não dependem de implementações concretas
3. **Single Source of Truth**: Estado centralizado
4. **Testabilidade**: Cada camada pode ser testada independentemente

### 📦 Dependências Necessárias

```yaml
dependencies:
  auto_injector: ^2.0.4 # Injeção de dependências
  result_dart: ^1.1.0 # Result Pattern
  freezed_annotation: ^2.4.1 # Geração de código
  lucid_validation: ^3.0.0 # Validação
  dio: ^5.4.0 # Cliente HTTP
  shared_preferences: ^2.2.0 # Storage local

dev_dependencies:
  build_runner: ^2.4.8
  freezed: ^2.4.7
  json_serializable: ^6.7.1
  mocktail: ^1.0.0 # Para testes com mocks
```

---

## 🚀 Criação de Projeto do Zero

### **Passo 0: Setup Inicial do Projeto**

#### 0.1 Criar Projeto Flutter

```bash
flutter create nome_do_projeto
cd nome_do_projeto
```

#### 0.2 Adicionar Dependências

- Edite o `pubspec.yaml` com as dependências listadas acima
- Execute: `flutter pub get`

#### 0.3 Criar Estrutura de Pastas

```bash
# Dentro da pasta lib/
mkdir -p domain/entities domain/dtos domain/validators
mkdir -p data/repositories data/services data/exceptions
mkdir -p ui/auth/login ui/auth/logout ui/home ui/splash
mkdir -p config
mkdir -p utils/exceptions
```

#### 0.4 Configurar analysis_options.yaml

```yaml
# analysis_options.yaml (adicionar no root do projeto)
include: package:flutter_lints/flutter.yaml

analyzer:
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
  errors:
    invalid_annotation_target: ignore
```

#### 0.5 Configurar build.yaml

```yaml
# build.yaml (criar no root do projeto)
targets:
  $default:
    builders:
      freezed:
        generate_for:
          - lib/**/*.dart
      json_serializable:
        generate_for:
          - lib/**/*.dart
```

#### 0.6 Configurar main.dart Base

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'config/dependencies.dart';

void main() {
  setupDependencies();  // Configura injeção de dependências
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Nome do App',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: const SplashPage(), // Começar sempre com splash
    );
  }
}
```

#### 0.7 Criar Arquivos Base Obrigatórios

**Client HTTP Base:**

```dart
// lib/data/services/client_http.dart
import 'package:dio/dio.dart';
import 'package:result_dart/result_dart.dart';

class ClientHttp {
  final Dio _dio;

  ClientHttp(this._dio) {
    _dio.options.baseUrl = 'https://sua-api.com/api';
    _dio.options.connectTimeout = const Duration(seconds: 30);
    _dio.options.receiveTimeout = const Duration(seconds: 30);
  }

  AsyncResult<Response> get(String path, [Map<String, dynamic>? queryParams]) async {
    try {
      final response = await _dio.get(path, queryParameters: queryParams);
      return Success(response);
    } catch (e) {
      return Failure(Exception('Erro GET: $e'));
    }
  }

  AsyncResult<Response> post(String path, Map<String, dynamic> data) async {
    try {
      final response = await _dio.post(path, data: data);
      return Success(response);
    } catch (e) {
      return Failure(Exception('Erro POST: $e'));
    }
  }
}
```

**Storage Local Base:**

```dart
// lib/data/services/local_storage.dart
import 'package:shared_preferences/shared_preferences.dart';
import 'package:result_dart/result_dart.dart';

class LocalStorage {
  AsyncResult<String> getData(String key) async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final data = prefs.getString(key);
      if (data != null) {
        return Success(data);
      }
      return Failure(Exception('Dados não encontrados para a chave: $key'));
    } catch (e) {
      return Failure(Exception('Erro ao buscar dados: $e'));
    }
  }

  AsyncResult<bool> saveData(String key, String value) async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final result = await prefs.setString(key, value);
      return Success(result);
    } catch (e) {
      return Failure(Exception('Erro ao salvar dados: $e'));
    }
  }

  AsyncResult<bool> removeData(String key) async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final result = await prefs.remove(key);
      return Success(result);
    } catch (e) {
      return Failure(Exception('Erro ao remover dados: $e'));
    }
  }
}
```

**Splash Page Base:**

```dart
// lib/ui/splash/splash_page.dart
import 'package:flutter/material.dart';

class SplashPage extends StatefulWidget {
  const SplashPage({Key? key}) : super(key: key);

  @override
  State<SplashPage> createState() => _SplashPageState();
}

class _SplashPageState extends State<SplashPage> {
  @override
  void initState() {
    super.initState();
    _navigateToHome();
  }

  _navigateToHome() async {
    await Future.delayed(const Duration(seconds: 2));
    if (mounted) {
      Navigator.pushReplacementNamed(context, '/home');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.flutter_dash, size: 100, color: Colors.blue),
            SizedBox(height: 20),
            Text('Nome do App', style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
            SizedBox(height: 20),
            CircularProgressIndicator(),
          ],
        ),
      ),
    );
  }
}
```

#### 0.8 Comandos Finais de Setup

```bash
# Executar para gerar arquivos necessários
flutter pub get
flutter packages pub run build_runner build

# Verificar se tudo está funcionando
flutter analyze
flutter test
```

#### 0.9 Checklist de Validação do Setup

- [ ] Projeto Flutter criado com sucesso
- [ ] Todas as dependências adicionadas no pubspec.yaml
- [ ] Estrutura de pastas criada conforme padrão
- [ ] Arquivos de configuração (analysis_options.yaml, build.yaml) criados
- [ ] Arquivos base (ClientHttp, LocalStorage, SplashPage) implementados
- [ ] main.dart configurado com setupDependencies()
- [ ] `flutter analyze` executa sem erros
- [ ] Projeto compila com `flutter run`

**⚠️ IMPORTANTE:** Após o setup inicial, SEMPRE começar pelo Domain Layer ao implementar novas features!

---

## ⚙️ Configuração e Injeção de Dependências

### 🎯 Objetivo

Gerenciar dependências de forma centralizada, promovendo baixo acoplamento e alta testabilidade.

### 📁 Estrutura Obrigatória

```
config/
└── dependencies.dart  # Arquivo principal de configuração
```

### 🔧 Implementação Base

```dart
// config/dependencies.dart
import 'package:auto_injector/auto_injector.dart';
// imports dos repositories, services, etc.

final injector = AutoInjector();

void setupDependencies() {
  // ORDEM IMPORTANTE: Dependências base primeiro

  // 1. Dependências externas (HTTP, Storage)
  injector.addInstance(Dio());
  injector.addSingleton(LocalStorage.new);

  // 2. Services compostos
  injector.addSingleton(ClientHttp.new);
  injector.addSingleton(AuthClientHttp.new);
  injector.addSingleton(AuthLocalStorage.new);

  // 3. Repositories (implementações)
  injector.addSingleton<AuthRepository>(RemoteAuthRepository.new);

  // 4. ViewModels
  injector.addSingleton(LoginViewmodel.new);
  injector.addSingleton(LogoutViewmodel.new);
  injector.addSingleton(MainViewmodel.new);
}
```

### 📱 Uso no main.dart

```dart
// main.dart
void main() {
  setupDependencies();  // 1. Configura todas as dependências
  runApp(const MainApp());  // 2. Inicia a aplicação
}
```

### 🔗 Tipos de Registro

| Tipo           | Uso                   | Quando usar                           |
| -------------- | --------------------- | ------------------------------------- |
| `addSingleton` | Uma instância global  | Repositories, Services, ViewModels    |
| `addInstance`  | Objeto já criado      | Configurações específicas (Dio, etc.) |
| `addTransient` | Nova instância sempre | Objetos temporários, sem estado       |

### 💻 Como Injetar nas Classes

```dart
// Em qualquer Widget/Page
class LoginPage extends StatefulWidget {
  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final viewModel = injector.get<LoginViewmodel>();  // ⚡ Injeção
  // resto do código...
}
```

Para mais detalhes, consulte o arquivo [Configuração e Injeção de Dependências](../config.md).

---

## 📂 Camada Domain (CORE)

### 🎯 Objetivo

Definir as regras de negócio, entidades e contratos da aplicação. Esta é a camada **mais importante** e **independente** de todas as outras.

### 📁 Estrutura Obrigatória

```
domain/
├── entities/           # Modelos fundamentais da aplicação
│   ├── user_entity.dart
│   ├── user_entity.freezed.dart  # Gerado automaticamente
│   └── user_entity.g.dart        # Gerado automaticamente
├── dtos/              # Objetos para transferência de dados
│   └── credentials.dart
└── validators/        # Regras de validação de negócio
    └── credentials_validator.dart
```

### 🏗️ Entidades com Freezed

```dart
// domain/entities/user_entity.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_entity.freezed.dart';
part 'user_entity.g.dart';

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

### 📦 DTOs (Data Transfer Objects)

```dart
// domain/dtos/credentials.dart
class Credentials {
  String email;
  String password;

  Credentials({
    this.email = '',
    this.password = '',
  });

  void setEmail(String email) => this.email = email;
  void setPassword(String password) => this.password = password;
}
```

### ✅ Validadores com Lucid Validation

```dart
// domain/validators/credentials_validator.dart
import 'package:lucid_validation/lucid_validation.dart';

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

### 🔨 Comandos para Gerar Código

```bash
# Após criar/modificar entidades
flutter packages pub run build_runner build
```

### 💡 Regras Importantes

- ❌ **NÃO deve** depender de outras camadas
- ✅ **DEVE** conter apenas regras de negócio
- ✅ **DEVE** usar sealed classes para estados
- ✅ **DEVE** usar Freezed para imutabilidade

Para mais detalhes, consulte o arquivo [Camada Domain](../domain.md).

---

## 📂 Camada Data

### 🎯 Objetivo

Implementar os contratos definidos na camada Domain e gerenciar o acesso a dados (APIs, storage local, cache).

### 📁 Estrutura Obrigatória

```
data/
├── repositories/          # Implementações dos contratos
│   └── auth/
│       ├── auth_repository.dart         # Interface/Contrato
│       └── remote_auth_repository.dart  # Implementação
├── services/             # Serviços de acesso a dados
│   ├── client_http.dart              # Cliente HTTP genérico
│   ├── local_storage.dart            # Storage local genérico
│   └── auth/
│       ├── auth_client_http.dart     # Cliente HTTP específico
│       └── auth_local_storage.dart   # Storage específico
└── exceptions/           # Tratamento de erros customizados
    └── exceptions.dart
```

### 🏗️ Repository Pattern

#### Interface (Contrato)

```dart
// data/repositories/auth/auth_repository.dart
import 'package:result_dart/result_dart.dart';

abstract interface class AuthRepository {
  AsyncResult<LoggedUser> login(Credentials credentials);
  AsyncResult<Unit> logout();
  AsyncResult<LoggedUser> getUser();
  Stream<User> userObserver();
  void dispose();
}
```

#### Implementação

```dart
// data/repositories/auth/remote_auth_repository.dart
class RemoteAuthRepository implements AuthRepository {
  final AuthLocalStorage _authLocalStorage;
  final AuthClientHttp _authClientHttp;
  final _streamController = StreamController<User>.broadcast();

  RemoteAuthRepository(this._authLocalStorage, this._authClientHttp);

  @override
  AsyncResult<LoggedUser> login(Credentials credentials) {
    return _authClientHttp
        .login(credentials)
        .flatMap(_authLocalStorage.saveUser)
        .onSuccess(_streamController.add);
  }

  @override
  AsyncResult<Unit> logout() => _authLocalStorage.removeUser().onSuccess(
    (_) => _streamController.add(const NotLoggedUser()),
  );

  @override
  Stream<User> userObserver() => _streamController.stream;
}
```

### 🌐 Services (Acesso a Dados)

#### Cliente HTTP

```dart
// data/services/auth/auth_client_http.dart
class AuthClientHttp {
  final ClientHttp _clientHttp;
  AuthClientHttp(this._clientHttp);

  AsyncResult<LoggedUser> login(Credentials credentials) async {
    final result = await _clientHttp.post('/login', {
      'email': credentials.email,
      'password': credentials.password,
    });

    return result.map((response) => LoggedUser.fromJson(response.data));
  }
}
```

#### Storage Local

```dart
// data/services/auth/auth_local_storage.dart
class AuthLocalStorage {
  final LocalStorage _localStorage;
  static const _userKey = 'user_data';

  AuthLocalStorage(this._localStorage);

  AsyncResult<LoggedUser> getUser() => _localStorage
      .getData(_userKey)
      .map((json) => LoggedUser.fromJson(jsonDecode(json)));

  AsyncResult<LoggedUser> saveUser(LoggedUser user) => _localStorage
      .saveData(_userKey, jsonEncode(user.toJson()))
      .then((_) => Success(user));
}
```

### 💡 Regras Importantes

- ✅ **DEVE** implementar interfaces do Domain
- ✅ **DEVE** usar Result Pattern para erros
- ✅ **DEVE** usar composição de services
- ❌ **NÃO deve** conter regras de negócio

Para mais detalhes, consulte o arquivo [Camada Data](../data.md).

---

## 📂 Camada UI (MVVM Pattern)

### 🎯 Objetivo

Criar a interface do usuário seguindo o padrão **MVVM (Model-View-ViewModel)**.

### 📁 Estrutura Obrigatória

```
ui/
├── auth/                    # Feature de autenticação
│   ├── login/
│   │   ├── login_page.dart           # View (Página)
│   │   └── viewmodels/
│   │       └── login_viewmodel.dart  # ViewModel
│   └── logout/
│       ├── viewmodels/
│       │   └── logout_viewmodel.dart
│       └── widgets/
│           └── logout_button.dart    # Widget reutilizável
├── home/                    # Feature home
│   └── home_page.dart
└── splash/                  # Splash screen
    └── splash.dart
```

### 🏗️ MVVM - Separação de Responsabilidades

#### 📱 View (Page/Widget)

```dart
// ui/auth/login/login_page.dart
class LoginPage extends StatefulWidget {
  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final viewModel = injector.get<LoginViewmodel>();
  final validator = CredentialsValidator();
  final credentials = Credentials();

  @override
  void initState() {
    super.initState();
    viewModel.loginCommand.addListener(_onLoginResult);
  }

  void _onLoginResult() {
    if (viewModel.loginCommand.value.isFailure) {
      final error = viewModel.loginCommand.value as FailureCommand;
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(error.error.toString())),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Login')),
      body: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(
              onChanged: credentials.setEmail,
              decoration: InputDecoration(labelText: 'Email'),
            ),
            TextField(
              onChanged: credentials.setPassword,
              obscureText: true,
              decoration: InputDecoration(labelText: 'Password'),
            ),
            SizedBox(height: 20),
            ListenableBuilder(
              listenable: viewModel.loginCommand,
              builder: (context, _) {
                return ElevatedButton(
                  onPressed: viewModel.loginCommand.value.isRunning
                      ? null
                      : () => _handleLogin(),
                  child: viewModel.loginCommand.value.isRunning
                      ? CircularProgressIndicator()
                      : Text('Login'),
                );
              },
            ),
          ],
        ),
      ),
    );
  }

  void _handleLogin() {
    final validation = validator.validate(credentials);
    if (validation.isValid) {
      viewModel.loginCommand.execute(credentials);
    } else {
      // Mostrar erros de validação
    }
  }
}
```

#### 🧠 ViewModel

```dart
// ui/auth/login/viewmodels/login_viewmodel.dart
import 'package:flutter/foundation.dart';
import 'package:result_dart/result_dart.dart';

class LoginViewmodel extends ChangeNotifier {
  final AuthRepository _authRepository;

  LoginViewmodel(this._authRepository);

  // Command Pattern para operações assíncronas
  late final loginCommand = Command1(_login);

  AsyncResult<LoggedUser> _login(Credentials credentials) async {
    return _authRepository.login(credentials);
  }
}
```

### ⚡ Command Pattern

O **Command Pattern** gerencia estados de operações assíncronas automaticamente:

```dart
// Estados disponíveis:
// - NotExecutedCommand: Não executado
// - RunningCommand: Em execução (loading)
// - SuccessCommand<T>: Sucesso com resultado
// - FailureCommand: Falha com erro

// Uso na UI:
viewModel.loginCommand.value.when(
  notExecuted: () => Text('Pronto para executar'),
  running: () => CircularProgressIndicator(),
  success: (result) => Text('Login realizado!'),
  failure: (error) => Text('Erro: $error'),
);
```

### 🏠 ViewModel Global (MainViewmodel)

```dart
// main_viewmodel.dart
class MainViewmodel extends ChangeNotifier {
  final AuthRepository _authRepository;
  late final StreamSubscription _userSubscription;

  User _user = NotLoggedUser();
  User get user => _user;

  MainViewmodel(this._authRepository) {
    _userSubscription = _authRepository.userObserver().listen((user) {
      _user = user;
      notifyListeners();
    });
  }

  @override
  void dispose() {
    _userSubscription.cancel();
    super.dispose();
  }
}
```

### 💡 Regras Importantes

- ❌ **View NÃO deve** conter lógica de negócio
- ❌ **ViewModel NÃO deve** acessar dados diretamente
- ✅ **DEVE** usar Command Pattern para operações assíncronas
- ✅ **DEVE** usar ListenableBuilder para reatividade

Para mais detalhes, consulte o arquivo [Camada UI](../ui.md).

---

## 🛠️ Padrões e Boas Práticas

### 🎯 Objetivo

Adotar padrões modernos para melhorar a qualidade, manutenibilidade e testabilidade do código.

### ⚡ Result Pattern - Tratamento de Erros

**Substitui exceptions** por tipos explícitos, tornando erros **visíveis na assinatura** dos métodos.

```dart
// ❌ Abordagem tradicional (com exceptions)
User login(String email, String password) {
  if (email.isEmpty) throw ValidationException('Email obrigatório');
  return user;
}

// ✅ Abordagem com Result Pattern
AsyncResult<User> login(String email, String password) async {
  if (email.isEmpty) return Failure(ValidationException('Email obrigatório'));
  return Success(user);
}
```

#### Vantagens do Result Pattern

| Tradicional (Exceptions)    | Result Pattern                    |
| --------------------------- | --------------------------------- |
| ❌ Erros implícitos         | ✅ Erros explícitos na assinatura |
| ❌ Fácil esquecer try/catch | ✅ Compilador força tratamento    |
| ❌ Performance custosa      | ✅ Performance superior           |
| ❌ Difícil composição       | ✅ Operações encadeáveis          |

#### Uso Prático

```dart
// Tratamento individual
final result = await repository.login(credentials);
result.fold(
  (success) => print('Login realizado: $success'),
  (failure) => print('Erro: $failure'),
);

// Encadeamento com flatMap
return _authClientHttp
    .login(credentials)                    // AsyncResult<LoggedUser>
    .flatMap(_authLocalStorage.saveUser)   // AsyncResult<LoggedUser>
    .onSuccess(_streamController.add);     // AsyncResult<LoggedUser>
```

### 🧊 Freezed Pattern - Imutabilidade

**Gera código** para classes imutáveis, igualdade, copyWith e serialização.

```dart
@freezed
class Product with _$Product {
  const factory Product({
    required int id,
    required String name,
    required double price,
    String? description,
  }) = _Product;

  factory Product.fromJson(Map<String, Object?> json) => _$ProductFromJson(json);
}
```

#### Funcionalidades Geradas

```dart
// Imutabilidade
final product = Product(id: 1, name: 'Produto', price: 100.0);

// Igualdade automática
final product2 = Product(id: 1, name: 'Produto', price: 100.0);
print(product == product2); // true

// CopyWith
final updatedProduct = product.copyWith(price: 150.0);

// Serialização
final json = product.toJson();
final fromJson = Product.fromJson(json);
```

### ⚡ Command Pattern - Operações Assíncronas

**Encapsula operações** com estados automáticos (loading, success, failure).

```dart
// No ViewModel
late final loginCommand = Command1(_login);
late final deleteCommand = Command1(_delete);

AsyncResult<User> _login(Credentials credentials) async {
  return _authRepository.login(credentials);
}

// Na UI - Reatividade automática
ListenableBuilder(
  listenable: viewModel.loginCommand,
  builder: (context, _) {
    final command = viewModel.loginCommand.value;

    return ElevatedButton(
      onPressed: command.isRunning ? null : () => command.execute(credentials),
      child: command.isRunning
          ? CircularProgressIndicator()
          : Text('Login'),
    );
  },
)
```

### 🔄 Stream Pattern - Reatividade

**Single Source of Truth** com observação de mudanças.

```dart
// No Repository
final _streamController = StreamController<User>.broadcast();

Stream<User> userObserver() => _streamController.stream;

// Notificar mudanças
_streamController.add(loggedUser);

// No ViewModel - Escutar mudanças
_userSubscription = _authRepository.userObserver().listen((user) {
  _user = user;
  notifyListeners();
});
```

### 📝 Validation Pattern

**Regras centralizadas** de validação com Lucid Validation.

```dart
class ProductValidator extends LucidValidator<Product> {
  ProductValidator() {
    ruleFor((p) => p.name, key: 'name')
        .notEmpty()
        .minLength(3)
        .maxLength(50);

    ruleFor((p) => p.price, key: 'price')
        .greaterThan(0)
        .lessThanOrEqualTo(10000);
  }
}

// Uso
final validator = ProductValidator();
final result = validator.validate(product);

if (result.isValid) {
  // Produto válido
} else {
  // Exibir erros
  for (final error in result.errors) {
    print('${error.key}: ${error.message}');
  }
}
```

Para mais detalhes, consulte o arquivo [Padrões e Boas Práticas](../patterns.md).

---

## 📖 Checklist de Implementação Passo a Passo

### 🚀 Para Implementar Qualquer Feature

#### 1️⃣ **Camada Domain (PRIMEIRO)**

- [ ] Criar entidades com `@freezed`
- [ ] Definir DTOs necessários
- [ ] Criar validadores com `LucidValidator`
- [ ] Executar `flutter packages pub run build_runner build`

#### 2️⃣ **Camada Data (SEGUNDO)**

- [ ] Criar interface do Repository no `data/repositories/`
- [ ] Implementar Repository concreto
- [ ] Criar Services necessários (`client_http`, `local_storage`)
- [ ] Implementar tratamento de erros com Result Pattern

#### 3️⃣ **Configuração (TERCEIRO)**

- [ ] Registrar dependências no `config/dependencies.dart`
- [ ] Seguir ordem: External → Services → Repositories → ViewModels

#### 4️⃣ **Camada UI (QUARTO)**

- [ ] Criar ViewModel com `ChangeNotifier`
- [ ] Implementar Commands para operações assíncronas
- [ ] Criar Page/Widget
- [ ] Conectar com ViewModel usando `injector.get<>`
- [ ] Usar `ListenableBuilder` para reatividade

### 🔍 Exemplo Prático: Feature de Produtos

#### Domain Layer

```dart
// 1. Entidade
@freezed
class Product with _$Product {
  const factory Product({
    required int id,
    required String name,
    required double price,
    String? description,
  }) = _Product;
}

// 2. DTO
class ProductFilter {
  String? category;
  double? minPrice;
  double? maxPrice;
}

// 3. Validador
class ProductValidator extends LucidValidator<Product> {
  ProductValidator() {
    ruleFor((p) => p.name, key: 'name').notEmpty();
    ruleFor((p) => p.price, key: 'price').greaterThan(0);
  }
}
```

#### Data Layer

```dart
// 1. Interface
abstract interface class ProductRepository {
  AsyncResult<List<Product>> getProducts(ProductFilter filter);
  AsyncResult<Product> getProduct(int id);
  AsyncResult<Product> createProduct(Product product);
}

// 2. Implementação
class RemoteProductRepository implements ProductRepository {
  final ProductClientHttp _clientHttp;

  @override
  AsyncResult<List<Product>> getProducts(ProductFilter filter) async {
    return _clientHttp.getProducts(filter);
  }
}

// 3. Service
class ProductClientHttp {
  final ClientHttp _clientHttp;

  AsyncResult<List<Product>> getProducts(ProductFilter filter) async {
    final result = await _clientHttp.get('/products', filter.toJson());
    return result.map((response) =>
      (response.data as List).map((json) => Product.fromJson(json)).toList()
    );
  }
}
```

#### UI Layer

```dart
// 1. ViewModel
class ProductListViewmodel extends ChangeNotifier {
  final ProductRepository _repository;

  ProductListViewmodel(this._repository);

  late final loadProductsCommand = Command0(_loadProducts);
  late final createProductCommand = Command1(_createProduct);

  List<Product> _products = [];
  List<Product> get products => _products;

  AsyncResult<List<Product>> _loadProducts() async {
    final result = await _repository.getProducts(ProductFilter());
    return result.onSuccess((products) {
      _products = products;
      notifyListeners();
    });
  }
}

// 2. Page
class ProductListPage extends StatefulWidget {
  @override
  State<ProductListPage> createState() => _ProductListPageState();
}

class _ProductListPageState extends State<ProductListPage> {
  final viewModel = injector.get<ProductListViewmodel>();

  @override
  void initState() {
    super.initState();
    viewModel.loadProductsCommand.execute();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Produtos')),
      body: ListenableBuilder(
        listenable: viewModel,
        builder: (context, _) {
          return ListView.builder(
            itemCount: viewModel.products.length,
            itemBuilder: (context, index) {
              final product = viewModel.products[index];
              return ListTile(
                title: Text(product.name),
                subtitle: Text('R\$ ${product.price}'),
              );
            },
          );
        },
      ),
    );
  }
}
```

Para um guia detalhado com exemplos completos, consulte o arquivo [Guia Prático](../practical-guide.md).

---

## 🧪 Testes (Obrigatório)

### 📁 Estrutura de Testes

```
test/
├── domain/
│   ├── entities/
│   └── validators/
├── data/
│   ├── repositories/
│   └── services/
└── ui/
    └── viewmodels/
```

### 🔬 Tipos de Testes

#### Domain Tests

```dart
// test/domain/validators/product_validator_test.dart
void main() {
  group('ProductValidator', () {
    late ProductValidator validator;

    setUp(() {
      validator = ProductValidator();
    });

    test('should validate valid product', () {
      final product = Product(id: 1, name: 'Valid Product', price: 100.0);
      final result = validator.validate(product);
      expect(result.isValid, isTrue);
    });

    test('should fail for empty name', () {
      final product = Product(id: 1, name: '', price: 100.0);
      final result = validator.validate(product);
      expect(result.isValid, isFalse);
      expect(result.errors.first.key, 'name');
    });
  });
}
```

#### Repository Tests (com Mocks)

```dart
// test/data/repositories/product_repository_test.dart
void main() {
  group('ProductRepository', () {
    late ProductRepository repository;
    late MockProductClientHttp mockClientHttp;

    setUp(() {
      mockClientHttp = MockProductClientHttp();
      repository = RemoteProductRepository(mockClientHttp);
    });

    test('should return products on success', () async {
      // Arrange
      final products = [Product(id: 1, name: 'Test', price: 100.0)];
      when(() => mockClientHttp.getProducts(any()))
          .thenAnswer((_) async => Success(products));

      // Act
      final result = await repository.getProducts(ProductFilter());

      // Assert
      expect(result.isSuccess(), isTrue);
      expect(result.getOrNull(), equals(products));
    });
  });
}
```

### � Dependências de Teste

```yaml
dev_dependencies:
  flutter_test:
  mocktail: ^1.0.0 # Para mocks
  test: ^1.24.0
```

---

## �🔗 Links para Documentação Completa

### 📚 Documentação Interna

- [Configuração e Injeção de Dependências](../config.md)
- [Camada Domain](../domain.md)
- [Camada Data](../data.md)
- [Camada UI](../ui.md)
- [Padrões e Boas Práticas](../patterns.md)
- [Guia Prático](../practical-guide.md)

### 🌐 Documentação Oficial Flutter

- [Flutter Documentation](https://docs.flutter.dev/)
- [Flutter Architectural Overview](https://docs.flutter.dev/resources/architectural-overview)
- [State Management Options](https://docs.flutter.dev/development/data-and-backend/state-mgmt/options)
- [Package Management](https://docs.flutter.dev/development/packages-and-plugins/using-packages)
- [Testing Flutter Apps](https://docs.flutter.dev/testing)

### 📦 Packages Utilizados

- [auto_injector](https://pub.dev/packages/auto_injector) - Injeção de Dependências
- [result_dart](https://pub.dev/packages/result_dart) - Result Pattern
- [freezed](https://pub.dev/packages/freezed) - Geração de código para imutabilidade
- [lucid_validation](https://pub.dev/packages/lucid_validation) - Validação de dados
- [dio](https://pub.dev/packages/dio) - Cliente HTTP

---

## ⚠️ Regras Importantes para Desenvolvedores

### ✅ O QUE FAZER

1. **Sempre começar pelo Domain** (entidades, DTOs, validadores)
2. **Usar Result Pattern** para todas as operações que podem falhar
3. **Registrar dependências** na ordem correta no `dependencies.dart`
4. **Criar testes** para cada camada
5. **Usar Command Pattern** para operações assíncronas na UI
6. **Seguir a estrutura de pastas** exatamente como definida

### ❌ O QUE NÃO FAZER

1. **Nunca acessar dados diretamente** na UI (sempre via ViewModel)
2. **Nunca colocar lógica de negócio** na UI ou Data layer
3. **Nunca usar exceptions** para fluxo normal (usar Result Pattern)
4. **Nunca fazer Domain depender** de outras camadas
5. **Nunca esquecer de executar** `build_runner` após modificar entidades
6. **Nunca misturar responsabilidades** entre as camadas

---

Este documento centraliza todas as informações necessárias para criar aplicações seguindo a arquitetura recomendada. Use-o como referência para instruir agentes ou desenvolvedores juniores.
