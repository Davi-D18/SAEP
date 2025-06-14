### Guia Essencial 20/80: Serializers no Django REST Framework

Os serializers são o coração do DRF, responsáveis por:
1. Converter objetos Python/dados do banco ➔ JSON
2. Converter JSON recebido ➔ Objetos Python
3. Validar dados de entrada
4. Controlar o que é exposto na API

---

#### **1. Tipos de Serializers**
**a) `Serializer` (Base)**
```python
from rest_framework import serializers

class ProdutoSerializer(serializers.Serializer):
    nome = serializers.CharField(max_length=100)
    preco = serializers.DecimalField(max_digits=10, decimal_places=2)
    estoque = serializers.IntegerField()
```

**b) `ModelSerializer` (Recomendado para 90% dos casos)**
```python
class ProdutoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Produto  # Model do Django
        fields = ['id', 'nome', 'preco', 'estoque']  # Campos incluídos
        # fields = '__all__'  # Todos os campos
        # exclude = ['campo_secreto']  # Exclui campos específicos
```

---

#### **2. Campos Essenciais e Parâmetros**
| Campo                 | Uso Comum                     | Parâmetros Importantes                     |
|-----------------------|-------------------------------|--------------------------------------------|
| `CharField`           | Texto                         | `max_length`, `allow_blank`, `trim_whitespace` |
| `IntegerField`        | Números inteiros              | `min_value`, `max_value`                   |
| `DecimalField`        | Valores monetários            | `max_digits`, `decimal_places`, `coerce_to_string` |
| `BooleanField`        | Verdadeiro/Falso              | `default`                                  |
| `DateTimeField`       | Datas                         | `format`, `input_formats`, `default_timezone` |
| `EmailField`          | E-mails                       | Validação automática de formato            |
| `SerializerMethodField`| Campo calculado               | `method_name`                              |

**Exemplo com parâmetros:**
```python
preco = serializers.DecimalField(
    max_digits=10,
    decimal_places=2,
    min_value=0.01,  # Valor mínimo R$0.01
    max_value=1000000.00,
    coerce_to_string=False  # Mantém como decimal
)
```

---

#### **3. Validação de Dados**
Três níveis de validação:

**a) Validação por campo:**
```python
def validate_estoque(self, value):
    if value < 0:
        raise serializers.ValidationError("Estoque não pode ser negativo")
    return value
```

**b) Validação de múltiplos campos:**
```python
def validate(self, data):
    if data['preco'] < 10 and data['estoque'] > 1000:
        raise serializers.ValidationError("Produtos baratos não podem ter estoque alto")
    return data
```

**c) Validadores externos:**
```python
def validar_nome_unico(value):
    if Produto.objects.filter(nome__iexact=value).exists():
        raise serializers.ValidationError("Nome já existe")

class ProdutoSerializer(serializers.ModelSerializer):
    nome = serializers.CharField(validators=[validar_nome_unico])
```

---

#### **4. Campos Customizados e Cálculos**
**a) `SerializerMethodField` (campo somente leitura):**
```python
class ProdutoSerializer(serializers.ModelSerializer):
    preco_com_taxa = serializers.SerializerMethodField()

    class Meta:
        model = Produto
        fields = ['id', 'nome', 'preco', 'preco_com_taxa']

    def get_preco_com_taxa(self, obj):
        return obj.preco * 1.1  # Adiciona 10% de taxa
```

**b) Sobrescrever representação:**
```python
def to_representation(self, instance):
    data = super().to_representation(instance)
    data['categoria'] = instance.categoria.nome  # Adiciona campo relacionado
    return data
```

---

#### **5. Relacionamentos**
**a) PrimaryKeyRelatedField (padrão):**
```python
class PedidoSerializer(serializers.ModelSerializer):
    produtos = serializers.PrimaryKeyRelatedField(many=True, queryset=Produto.objects.all())
```

**b) Nested Serializers (objetos completos):**
```python
class ProdutoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Produto
        fields = ['id', 'nome']

class PedidoSerializer(serializers.ModelSerializer):
    produtos = ProdutoSerializer(many=True, read_only=True)
```

**c) SlugRelatedField (usando campo descritivo):**
```python
produtos = serializers.SlugRelatedField(
    many=True,
    slug_field='nome',
    queryset=Produto.objects.all()
)
```

---

#### **6. Controle de Leitura/Escrita**
| Parâmetro       | Descrição                                      | Uso Típico                     |
|-----------------|-----------------------------------------------|--------------------------------|
| `read_only`     | Campo aparece na saída, mas ignora entrada    | IDs, campos calculados         |
| `write_only`    | Campo usado na entrada, mas não na saída      | Senhas, campos temporários     |
| `required`      | Se o campo é obrigatório na criação           | `default=False` para opcionais |

**Exemplo:**
```python
class UsuarioSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)  # Nunca exposto
    ultimo_login = serializers.DateTimeField(read_only=True)

    class Meta:
        model = User
        fields = ['id', 'username', 'password', 'ultimo_login']
```

---

#### **7. Métodos Especiais para CRUD**
**a) Criação com relacionamentos (sobrescrever `create`):**
```python
def create(self, validated_data):
    produtos_data = validated_data.pop('produtos')
    pedido = Pedido.objects.create(**validated_data)
    pedido.produtos.set(produtos_data)  # Para ManyToMany
    return pedido
```

**b) Atualização parcial (sobrescrever `update`):**
```python
def update(self, instance, validated_data):
    instance.nome = validated_data.get('nome', instance.nome)
    instance.preco = validated_data.get('preco', instance.preco)
    instance.save()
    return instance
```

---

#### **8. Serializers Dinâmicos**
**a) Contexto (passar dados adicionais):**
```python
# Na view:
serializer = ProdutoSerializer(produto, context={'request': request})

# No serializer:
foto_url = serializers.SerializerMethodField()

def get_foto_url(self, obj):
    request = self.context.get('request')
    return request.build_absolute_uri(obj.foto.url)
```

**b) Remover campos dinamicamente:**
```python
class DynamicFieldsSerializer(serializers.ModelSerializer):
    def __init__(self, *args, **kwargs):
        fields = kwargs.pop('fields', None)
        super().__init__(*args, **kwargs)
        if fields:
            allowed = set(fields)
            existing = set(self.fields)
            for field in existing - allowed:
                self.fields.pop(field)

# Uso na view:
serializer = ProdutoSerializer(produto, fields=['id', 'nome'])
```

---

### **Checklist 20/80 para Serializers**
1. Use **`ModelSerializer`** para operações CRUD padrão
2. Defina **`fields`** explicitamente (evite `__all__` em produção)
3. Adicione **validações customizadas** nos níveis certo:
   - Validador por campo: `validate_<campo>`
   - Validação global: `validate()`
4. Para campos calculados: **`SerializerMethodField`**
5. Controle acesso com **`read_only`** e **`write_only`**
6. Para relacionamentos:
   - Exibição simples: **`PrimaryKeyRelatedField`**
   - Objetos completos: **Nested Serializers**
7. Sobrescreva **`create`** e **`update`** para lógica complexa
8. Use **contexto** para acessar dados da requisição

Esses 20% de funcionalidades resolvem 80% das necessidades com serializers no DRF! 🚀