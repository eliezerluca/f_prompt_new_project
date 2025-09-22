# 📖 Guia Prático - Implementando Novos Features

**[⬅️ Voltar: Padrões](./patterns.md)** | **[🏠 Início](./README.md)**

---

## 🎯 Objetivo

Este guia ensina como **implementar novos features** seguindo a arquitetura apresentada. Vamos criar um exemplo prático: **Sistema de Perfil de Usuário**.

---

## 🚀 Feature: Sistema de Perfil

### 📋 Requisitos

- ✅ Exibir perfil do usuário logado
- ✅ Editar dados do perfil
- ✅ Upload de foto de perfil
- ✅ Salvar alterações localmente e remotamente

---

## 📚 Passo a Passo Completo

### 1️⃣ **Domain Layer - Definindo Entidades**

#### 📄 Criando Profile Entity

```dart
// lib/domain/entities/profile_entity.dart
import "package:freezed_annotation/freezed_annotation.dart";

part 'profile_entity.freezed.dart';
part 'profile_entity.g.dart';

@freezed
class Profile with _$Profile {
  const factory Profile({
    required String userId,
    required String name,
    required String email,
    String? bio,
    String? avatarUrl,
    DateTime? birthDate,
    required DateTime updatedAt,
  }) = _Profile;

  factory Profile.fromJson(Map<String, Object?> json) => _$ProfileFromJson(json);
}
```

#### 📄 DTO para Edição

```dart
// lib/domain/dtos/profile_update_dto.dart
class ProfileUpdateDto {
  String name;
  String? bio;
  String? avatarPath;
  DateTime? birthDate;

  ProfileUpdateDto({
    required this.name,
    this.bio,
    this.avatarPath,
    this.birthDate,
  });

  void setName(String name) => this.name = name;
  void setBio(String? bio) => this.bio = bio;
  void setAvatarPath(String? path) => this.avatarPath = path;
  void setBirthDate(DateTime? date) => this.birthDate = date;
}
```

#### 📄 Validator

```dart
// lib/domain/validators/profile_validator.dart
import 'package:lucid_validation/lucid_validation.dart';
import '../dtos/profile_update_dto.dart';

class ProfileValidator extends LucidValidator<ProfileUpdateDto> {
  ProfileValidator() {
    ruleFor((p) => p.name, key: 'name')
        .notEmpty()
        .minLength(2)
        .maxLength(50);

    ruleFor((p) => p.bio, key: 'bio')
        .maxLength(200);
  }
}
```

---

### 2️⃣ **Data Layer - Acesso a Dados**

#### 📄 Repository Interface

```dart
// lib/data/repositories/profile/profile_repository.dart
import 'package:result_dart/result_dart.dart';
import '../../../domain/entities/profile_entity.dart';
import '../../../domain/dtos/profile_update_dto.dart';

abstract interface class ProfileRepository {
  AsyncResult<Profile> getProfile(String userId);
  AsyncResult<Profile> updateProfile(String userId, ProfileUpdateDto dto);
  AsyncResult<String> uploadAvatar(String userId, String imagePath);
  Stream<Profile> profileObserver(String userId);
  void dispose();
}
```

#### 📄 Repository Implementation

```dart
// lib/data/repositories/profile/remote_profile_repository.dart
import 'dart:async';
import 'package:result_dart/result_dart.dart';
import 'profile_repository.dart';
import '../../services/profile/profile_client_http.dart';
import '../../services/profile/profile_local_storage.dart';
import '../../../domain/entities/profile_entity.dart';
import '../../../domain/dtos/profile_update_dto.dart';

class RemoteProfileRepository implements ProfileRepository {
  final ProfileClientHttp _profileClientHttp;
  final ProfileLocalStorage _profileLocalStorage;
  final _streamController = StreamController<Profile>.broadcast();

  RemoteProfileRepository(this._profileClientHttp, this._profileLocalStorage);

  @override
  AsyncResult<Profile> getProfile(String userId) {
    return _profileClientHttp
        .getProfile(userId)
        .flatMap((profile) => _profileLocalStorage.saveProfile(profile))
        .onSuccess(_streamController.add);
  }

  @override
  AsyncResult<Profile> updateProfile(String userId, ProfileUpdateDto dto) {
    return _profileClientHttp
        .updateProfile(userId, dto)
        .flatMap((profile) => _profileLocalStorage.saveProfile(profile))
        .onSuccess(_streamController.add);
  }

  @override
  AsyncResult<String> uploadAvatar(String userId, String imagePath) {
    return _profileClientHttp.uploadAvatar(userId, imagePath);
  }

  @override
  Stream<Profile> profileObserver(String userId) {
    return _streamController.stream.where((profile) => profile.userId == userId);
  }

  @override
  void dispose() {
    _streamController.close();
  }
}
```

#### 📄 HTTP Service

```dart
// lib/data/services/profile/profile_client_http.dart
import 'package:result_dart/result_dart.dart';
import '../client_http.dart';
import '../../../domain/entities/profile_entity.dart';
import '../../../domain/dtos/profile_update_dto.dart';

class ProfileClientHttp {
  final ClientHttp _clientHttp;

  ProfileClientHttp(this._clientHttp);

  AsyncResult<Profile> getProfile(String userId) async {
    final result = await _clientHttp.get('/profile/$userId');

    return result.map((response) => Profile.fromJson(response.data));
  }

  AsyncResult<Profile> updateProfile(String userId, ProfileUpdateDto dto) async {
    final result = await _clientHttp.put('/profile/$userId', {
      'name': dto.name,
      'bio': dto.bio,
      'birthDate': dto.birthDate?.toIso8601String(),
    });

    return result.map((response) => Profile.fromJson(response.data));
  }

  AsyncResult<String> uploadAvatar(String userId, String imagePath) async {
    final result = await _clientHttp.postFile('/profile/$userId/avatar', imagePath);

    return result.map((response) => response.data['avatarUrl']);
  }
}
```

#### 📄 Local Storage

```dart
// lib/data/services/profile/profile_local_storage.dart
import 'dart:convert';
import 'package:result_dart/result_dart.dart';
import '../local_storage.dart';
import '../../../domain/entities/profile_entity.dart';

class ProfileLocalStorage {
  final LocalStorage _localStorage;

  ProfileLocalStorage(this._localStorage);

  String _profileKey(String userId) => 'profile_$userId';

  AsyncResult<Profile> getProfile(String userId) {
    return _localStorage
        .getData(_profileKey(userId))
        .map((json) => Profile.fromJson(jsonDecode(json)));
  }

  AsyncResult<Profile> saveProfile(Profile profile) {
    return _localStorage
        .saveData(_profileKey(profile.userId), jsonEncode(profile.toJson()))
        .then((_) => Success(profile));
  }

  AsyncResult<Unit> removeProfile(String userId) {
    return _localStorage.removeData(_profileKey(userId));
  }
}
```

---

### 3️⃣ **UI Layer - Interface e ViewModels**

#### 📄 Profile ViewModel

```dart
// lib/ui/profile/viewmodels/profile_viewmodel.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:result_command/result_command.dart';
import 'package:result_dart/result_dart.dart';
import '../../../data/repositories/profile/profile_repository.dart';
import '../../../domain/entities/profile_entity.dart';
import '../../../domain/dtos/profile_update_dto.dart';

class ProfileViewmodel extends ChangeNotifier {
  final ProfileRepository _profileRepository;
  final String _userId;

  Profile? _profile;
  Profile? get profile => _profile;

  late final getProfileCommand = Command0(_getProfile);
  late final updateProfileCommand = Command1(_updateProfile);
  late final uploadAvatarCommand = Command1(_uploadAvatar);

  late final StreamSubscription _profileSubscription;

  ProfileViewmodel(this._profileRepository, this._userId) {
    _profileSubscription = _profileRepository
        .profileObserver(_userId)
        .listen(_onProfileChanged);

    // Carrega perfil automaticamente
    getProfileCommand.execute();
  }

  void _onProfileChanged(Profile profile) {
    _profile = profile;
    notifyListeners();
  }

  AsyncResult<Profile> _getProfile() {
    return _profileRepository.getProfile(_userId);
  }

  AsyncResult<Profile> _updateProfile(ProfileUpdateDto dto) {
    return _profileRepository.updateProfile(_userId, dto);
  }

  AsyncResult<String> _uploadAvatar(String imagePath) {
    return _profileRepository.uploadAvatar(_userId, imagePath);
  }

  @override
  void dispose() {
    _profileSubscription.cancel();
    _profileRepository.dispose();
    super.dispose();
  }
}
```

#### 📄 Profile Page

```dart
// lib/ui/profile/profile_page.dart
import 'package:flutter/material.dart';
import 'package:result_command/result_command.dart';
import '../../config/dependencies.dart';
import '../../domain/dtos/profile_update_dto.dart';
import '../../domain/validators/profile_validator.dart';
import 'viewmodels/profile_viewmodel.dart';

class ProfilePage extends StatefulWidget {
  final String userId;

  const ProfilePage({super.key, required this.userId});

  @override
  State<ProfilePage> createState() => _ProfilePageState();
}

class _ProfilePageState extends State<ProfilePage> {
  late final ProfileViewmodel viewModel;
  final _validator = ProfileValidator();
  late final ProfileUpdateDto _updateDto;

  @override
  void initState() {
    super.initState();
    viewModel = ProfileViewmodel(
      injector.get<ProfileRepository>(),
      widget.userId,
    );
    _updateDto = ProfileUpdateDto(name: '');

    _setupListeners();
  }

  void _setupListeners() {
    // Listener para comandos
    viewModel.updateProfileCommand.addListener(_onUpdateResult);
    viewModel.uploadAvatarCommand.addListener(_onUploadResult);
  }

  void _onUpdateResult() {
    final command = viewModel.updateProfileCommand.value;
    if (command.isSuccess) {
      _showSuccess('Perfil atualizado com sucesso!');
    } else if (command.isFailure) {
      _showError((command as FailureCommand).error.toString());
    }
  }

  void _onUploadResult() {
    final command = viewModel.uploadAvatarCommand.value;
    if (command.isFailure) {
      _showError((command as FailureCommand).error.toString());
    }
  }

  void _showSuccess(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message), backgroundColor: Colors.green),
    );
  }

  void _showError(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message), backgroundColor: Colors.red),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Perfil')),
      body: ListenableBuilder(
        listenable: viewModel,
        builder: (context, _) {
          if (viewModel.getProfileCommand.value.isRunning) {
            return Center(child: CircularProgressIndicator());
          }

          final profile = viewModel.profile;
          if (profile == null) {
            return Center(child: Text('Perfil não encontrado'));
          }

          return _buildProfileForm(profile);
        },
      ),
    );
  }

  Widget _buildProfileForm(Profile profile) {
    return Padding(
      padding: EdgeInsets.all(16),
      child: Column(
        children: [
          _buildAvatarSection(profile),
          SizedBox(height: 20),
          _buildNameField(profile),
          SizedBox(height: 16),
          _buildBioField(profile),
          SizedBox(height: 20),
          _buildSaveButton(),
        ],
      ),
    );
  }

  Widget _buildAvatarSection(Profile profile) {
    return GestureDetector(
      onTap: _pickAndUploadImage,
      child: Stack(
        children: [
          CircleAvatar(
            radius: 50,
            backgroundImage: profile.avatarUrl != null
                ? NetworkImage(profile.avatarUrl!)
                : null,
            child: profile.avatarUrl == null
                ? Icon(Icons.person, size: 50)
                : null,
          ),
          Positioned(
            bottom: 0,
            right: 0,
            child: Container(
              decoration: BoxDecoration(
                color: Theme.of(context).primaryColor,
                shape: BoxShape.circle,
              ),
              child: Icon(Icons.camera_alt, color: Colors.white, size: 20),
              padding: EdgeInsets.all(8),
            ),
          ),
          if (viewModel.uploadAvatarCommand.value.isRunning)
            Positioned.fill(
              child: Container(
                decoration: BoxDecoration(
                  color: Colors.black.withOpacity(0.5),
                  shape: BoxShape.circle,
                ),
                child: Center(
                  child: CircularProgressIndicator(color: Colors.white),
                ),
              ),
            ),
        ],
      ),
    );
  }

  Widget _buildNameField(Profile profile) {
    return TextFormField(
      initialValue: profile.name,
      onChanged: _updateDto.setName,
      validator: _validator.byField(_updateDto, 'name'),
      decoration: InputDecoration(
        labelText: 'Nome',
        border: OutlineInputBorder(),
      ),
    );
  }

  Widget _buildBioField(Profile profile) {
    return TextFormField(
      initialValue: profile.bio ?? '',
      onChanged: _updateDto.setBio,
      validator: _validator.byField(_updateDto, 'bio'),
      maxLines: 3,
      decoration: InputDecoration(
        labelText: 'Bio',
        border: OutlineInputBorder(),
      ),
    );
  }

  Widget _buildSaveButton() {
    return ListenableBuilder(
      listenable: viewModel.updateProfileCommand,
      builder: (context, _) {
        final command = viewModel.updateProfileCommand.value;

        return ElevatedButton(
          onPressed: command.isRunning ? null : _saveProfile,
          child: command.isRunning
              ? SizedBox(
                  width: 20,
                  height: 20,
                  child: CircularProgressIndicator(strokeWidth: 2),
                )
              : Text('Salvar'),
        );
      },
    );
  }

  void _saveProfile() {
    final validation = _validator.validate(_updateDto);
    if (validation.isValid) {
      viewModel.updateProfileCommand.execute(_updateDto);
    } else {
      _showError(validation.errors.first.message);
    }
  }

  void _pickAndUploadImage() async {
    // Implementar seleção de imagem
    // final ImagePicker picker = ImagePicker();
    // final XFile? image = await picker.pickImage(source: ImageSource.gallery);
    // if (image != null) {
    //   viewModel.uploadAvatarCommand.execute(image.path);
    // }
  }

  @override
  void dispose() {
    viewModel.dispose();
    super.dispose();
  }
}
```

---

### 4️⃣ **Configuração - Dependency Injection**

#### 📄 Atualizando dependencies.dart

```dart
// lib/config/dependencies.dart
void setupDependencies() {
  // ... dependências existentes

  // Profile dependencies
  injector.addSingleton(ProfileClientHttp.new);
  injector.addSingleton(ProfileLocalStorage.new);
  injector.addSingleton<ProfileRepository>(RemoteProfileRepository.new);
}
```

---

### 5️⃣ **Roteamento - Adicionando Nova Rota**

#### 📄 Criando nova página de rota

```dart
// lib/ui/profile/profile_route.dart
import 'package:flutter/material.dart';
import 'profile_page.dart';

class ProfileRoute extends StatelessWidget {
  const ProfileRoute({super.key});

  @override
  Widget build(BuildContext context) {
    // Aqui você normalmente obteria o userId do usuário logado
    final userId = "current_user_id"; // Obter do AuthRepository

    return ProfilePage(userId: userId);
  }
}
```

---

## 🧪 Testando o Feature

### 📄 Teste do ViewModel

```dart
// test/ui/profile/viewmodels/profile_viewmodel_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:result_dart/result_dart.dart';

class MockProfileRepository extends Mock implements ProfileRepository {}

void main() {
  group('ProfileViewmodel', () {
    late MockProfileRepository mockRepository;
    late ProfileViewmodel viewModel;

    setUp(() {
      mockRepository = MockProfileRepository();
      viewModel = ProfileViewmodel(mockRepository, 'user123');
    });

    test('should get profile on initialization', () async {
      // Arrange
      final profile = Profile(
        userId: 'user123',
        name: 'João',
        email: 'joao@test.com',
        updatedAt: DateTime.now(),
      );
      when(mockRepository.getProfile('user123'))
          .thenAnswer((_) async => Success(profile));

      // Act
      await viewModel.getProfileCommand.execute();

      // Assert
      verify(mockRepository.getProfile('user123')).called(1);
      expect(viewModel.getProfileCommand.value.isSuccess, true);
    });

    tearDown(() {
      viewModel.dispose();
    });
  });
}
```

---

## 📝 Checklist de Implementação

### ✅ **Domain Layer**

- [ ] Criar entidade com Freezed
- [ ] Definir DTOs para transferência
- [ ] Implementar validator com regras
- [ ] Gerar código com build_runner

### ✅ **Data Layer**

- [ ] Definir interface do repository
- [ ] Implementar repository concreto
- [ ] Criar HTTP service
- [ ] Criar local storage
- [ ] Implementar stream para observação

### ✅ **UI Layer**

- [ ] Criar ViewModel com commands
- [ ] Implementar Page com ListenableBuilder
- [ ] Configurar navigation
- [ ] Tratar estados de loading/error

### ✅ **Configuration**

- [ ] Registrar dependências no injector
- [ ] Configurar rotas
- [ ] Escrever testes básicos

---

## 🚀 Próximos Features Sugeridos

Seguindo esta mesma estrutura, você pode implementar:

1. **Sistema de Notificações**

   - Entity: Notification
   - Repository: NotificationRepository
   - UI: NotificationPage, NotificationListWidget

2. **Chat/Mensagens**

   - Entity: Message, Conversation
   - Repository: ChatRepository
   - UI: ChatPage, MessageBubble

3. **Configurações da App**
   - Entity: AppSettings
   - Repository: SettingsRepository
   - UI: SettingsPage

---

## 💡 Dicas Importantes

### ✅ **Sempre Siga a Ordem**

1. Domain primeiro (entidades, DTOs, validators)
2. Data depois (repositories, services)
3. UI por último (viewmodels, pages)
4. Configuração e testes

### ✅ **Use os Padrões**

- **Result Pattern** para tratamento de erros
- **Commands** para operações assíncronas
- **Streams** para observação de mudanças
- **Freezed** para entidades imutáveis

### ✅ **Mantenha Simples**

- Uma responsabilidade por classe
- Interfaces pequenas e focadas
- Composição sobre herança
- Teste as partes críticas

---

## 🎯 Conclusão

Com este guia, você tem um **template completo** para implementar qualquer feature seguindo a arquitetura. O importante é manter a **consistência** e seguir os **padrões estabelecidos**.

**Lembre-se**: A arquitetura existe para facilitar o desenvolvimento, não para complicá-lo. Se algo parecer muito complexo, provavelmente há uma forma mais simples de fazer! 😊

---

**[⬅️ Padrões](./patterns.md)** | **[🏠 Voltar ao Início](./README.md)**
