# Documentação de Genéricos e Constraints

## Visão Geral

A aplicação faz uso extensivo de **Genéricos (Generics)** em Dart para criar código reutilizável, type-safe e flexível. Os genéricos permitem que classes, interfaces e métodos operem em tipos especificados pelo usuário, mantendo a segurança de tipos em tempo de compilação.

## 1. Repositório Genérico - IRepository<T>

### Definição
`lib/repositories/repository.dart`
```dart
abstract class IRepository<T extends BaseModel> {
  Future<T?> findById(String id);
  Future<List<T>> findAll();
  Future<void> save(T entity);
  Future<void> saveAll(List<T> entities);
  Future<void> delete(String id);
  Future<void> deleteAll();
}
```

### Constraint Aplicado
- **`T extends BaseModel`**: Garante que apenas tipos que herdam de `BaseModel` podem ser usados.
- **Benefício**: Acesso garantido aos métodos e propriedades de `BaseModel` (id, createdAt, toMap, etc.) em tempo de compilação.

### Implementação Concreta
`lib/repositories/repository.dart`
```dart
class MemoryRepository<T extends BaseModel> implements IRepository<T> {
  final Map<String, T> _storage = {};
  
  @override
  Future<T?> findById(String id) async {
    return _storage[id];
  }
  
  @override
  Future<List<T>> findAll() async {
    return _storage.values.toList();
  }
  
  // ... outras implementações de CRUD
}
```

### Uso Prático
```dart
// Repositórios tipados específicos usando typedefs para clareza
typedef ProductRepository = MemoryRepository<Product>;
typedef BusinessRuleRepository = MemoryRepository<BusinessRule>;
typedef FormFieldRepository = MemoryRepository<FormFieldModel>;
```

### Vantagens
1. **Type Safety**: O compilador garante que apenas tipos corretos são usados, prevenindo erros em tempo de execução.
2. **Reutilização de Código**: Uma única implementação de `MemoryRepository` serve para múltiplos tipos de dados, reduzindo a duplicação.
3. **Intellisense e Refatoração**: Ferramentas de desenvolvimento fornecem autocomplete preciso e facilitam refatorações seguras.
4. **Manutenibilidade**: Mudanças na interface `IRepository` ou na implementação `MemoryRepository` afetam todas as instâncias de forma consistente.

## 2. Engine de Regras Genérica - RulesEngine<T>

### Definição
`lib/services/engine/rules_engine.dart`
```dart
class RulesEngine<T extends BusinessRule> {
  final IRepository<T> _repository;
  
  RulesEngine(this._repository);
  
  Future<RuleExecutionResult> processRules(Map<String, dynamic> context) async {
    final rules = await _repository.findAll();
    // ... lógica de processamento e aplicação de regras
  }
}
```

### Constraint Aplicado
- **`T extends BusinessRule`**: Garante que a `RulesEngine` só pode operar com tipos que herdam de `BusinessRule`.
- **Benefício**: Permite que a engine acesse métodos e propriedades específicas de regras de negócio (como `evaluate()` e `apply()`) de forma segura.

### Uso Especializado
```dart
// Engine específica para regras de negócio
final RulesEngine<BusinessRule> engine = RulesEngine(businessRuleRepository);

// Conceitualmente, poderia ser estendido para outros tipos de regras no futuro
// final RulesEngine<SecurityRule> securityEngine = RulesEngine(securityRuleRepository);
```

## 3. Controller de Formulário Genérico - FormController<T>

### Definição
`lib/controllers/form_controller.dart`
```dart
class FormController<T extends Product> extends ChangeNotifier 
    with ValidatorMixin, CalculatorMixin {
  
  T? _selectedProduct;
  
  Future<void> initializeForm(T product) async {
    _selectedProduct = product;
    // ... inicialização específica do tipo de produto
  }
}
```

### Constraint Aplicado
- **`T extends Product`**: Garante que o `FormController` só pode gerenciar formulários relacionados a tipos que herdam de `Product`.
- **Benefício**: Permite que o controller acesse propriedades e métodos específicos de produtos (como `getRequiredFields()` ou `calculatePrice()`) de forma segura e tipada.

### Especializações por Tipo (Implícitas)
Embora não haja subclasses explícitas de `FormController` para cada tipo de produto, a tipagem genérica permite que o mesmo `FormController` seja instanciado com diferentes tipos de `Product`, adaptando seu comportamento internamente.

```dart
// O mesmo FormController pode ser usado para diferentes tipos de produtos
final formControllerForIndustrial = FormController<IndustrialProductModel>();
final formControllerForResidential = FormController<ResidentialProductModel>();
```

### Vantagens da Especialização Implícita
1. **Flexibilidade**: Um único `FormController` pode ser adaptado para diferentes contextos de produto.
2. **Validação Específica**: As regras de validação e visibilidade de campos podem ser aplicadas dinamicamente com base no tipo de `Product` que o controller está gerenciando.
3. **Type Safety**: O compilador garante que o `FormController` sempre trabalhará com um tipo de `Product` válido.

## 4. Serviços de Fábrica Genéricos - FactoryService<T>

### Interface Base
`lib/services/factory/factory_service.dart`
```dart
abstract class FactoryService<T> {
  T create(Map<String, dynamic> config);
  bool canCreate(Map<String, dynamic> config);
}
```

### Implementações Específicas
`lib/services/factory/product_factory.dart` (Exemplo)
```dart
class ProductFactory implements FactoryService<Product> {
  @override
  Product create(Map<String, dynamic> config) {
    final type = ProductType.values.byName(config["type"]);
    switch (type) {
      case ProductType.industrial:
        return IndustrialProductModel.fromMap(config);
      case ProductType.residential:
        return ResidentialProductModel.fromMap(config);
      case ProductType.corporate:
        return CorporateProductModel.fromMap(config);
    }
  }

  @override
  bool canCreate(Map<String, dynamic> config) {
    return config.containsKey("type") && ProductType.values.any((e) => e.name == config["type"]);
  }
}
```

### Registro de Fábricas
`lib/services/factory/factory_registry.dart`
```dart
class FactoryRegistry {
  static final Map<Type, FactoryService> _factories = {};
  
  static void register<T>(FactoryService<T> factory) {
    _factories[T] = factory;
  }
  
  static T create<T>(Map<String, dynamic> config) {
    final factory = _factories[T] as FactoryService<T>?;
    if (factory == null) {
      throw ArgumentError("Factory not registered for type $T");
    }
    return factory.create(config);
  }

  static void initializeDefaultFactories() {
    register<Product>(ProductFactory());
    register<BusinessRule>(BusinessRuleFactory());
    register<FormFieldModel>(FormFieldFactory());
  }
}
```

## 5. Extensions Genéricas

### StringExtensions
`lib/extensions/string_extensions.dart`
```dart
extension StringExtensions on String {
  bool get isValidEmail {
    const pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$';
    return RegExp(pattern).hasMatch(this);
  }

  String get numbersOnly => replaceAll(RegExp(r'[^0-9]'), '');

  String get maskAsCPF {
    final numbers = numbersOnly;
    if (numbers.length != 11) return this;
    return '${numbers.substring(0, 3)}.${numbers.substring(3, 6)}.${numbers.substring(6, 9)}-${numbers.substring(9)}';
  }
}
```

## 6. Constraints Avançados

### Múltiplos Bounds (Conceitual)
Dart suporta múltiplos bounds em genéricos, embora não seja explicitamente utilizado de forma complexa neste projeto. Por exemplo:

```dart
// class MyClass<T extends SomeBaseClass & AnotherInterface> {
//   // T deve herdar de SomeBaseClass E implementar AnotherInterface
// }
```

### Constraints em Métodos (Conceitual)
Genéricos podem ser aplicados a métodos, permitindo maior flexibilidade e segurança de tipo em operações específicas.

```dart
// T transform<T, R>(List<T> items, R Function(T) converter) {
//   // ...
// }
```

## 7. Benefícios dos Genéricos na Aplicação

### Type Safety
- Erros de tipo são capturados em tempo de compilação, antes da execução, tornando o código mais robusto.
- Ex: `MemoryRepository<Product>` garante que apenas objetos `Product` podem ser armazenados e recuperados.

### Reutilização de Código
- Componentes como `IRepository`, `RulesEngine` e `FactoryService` são escritos uma única vez e reutilizados para diferentes tipos de dados.
- Isso reduz a quantidade de código a ser escrito e mantido.

### Performance
- Em Dart, os genéricos são implementados com 

