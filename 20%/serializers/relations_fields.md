#### **9. Parâmetros Universais de Campos (Core Arguments)**

Todos os campos de serializer compartilham estes parâmetros essenciais:

| Parâmetro       | Descrição                                                                 | Padrão   |
|-----------------|---------------------------------------------------------------------------|----------|
| `read_only`     | Campo aparece na saída mas é ignorado na entrada (create/update)          | `False`  |
| `write_only`    | Campo usado na entrada mas não aparece na saída                           | `False`  |
| `required`      | Se o campo é obrigatório durante a deserialização                         | `True`   |
| `default`       | Valor padrão se o campo não for fornecido (pode ser callable)             | -        |
| `allow_null`    | Aceita `None` como valor válido                                           | `False`  |
| `source`        | Especifica o atributo do model a ser usado (pode usar notação de ponto)   | Nome do campo |
| `validators`    | Lista de funções de validação                                             | `[]`     |
| `error_messages`| Dicionário de mensagens de erro personalizadas                            | `{}`     |
| `label`         | Rótulo para campos em formulários HTML                                    | -        |
| `help_text`     | Texto de ajuda para campos em formulários HTML                            | -        |
| `style`         | Dicionário para controle de renderização em formulários HTML              | `{}`     |

**Exemplo com múltiplos parâmetros:**
```python
preco = serializers.DecimalField(
    max_digits=10,
    decimal_places=2,
    required=False,
    default=0.0,
    allow_null=True,
    source='preco_venda',
    error_messages={
        'invalid': 'Valor monetário inválido',
        'max_value': 'Preço não pode exceder R$ 10.000,00'
    }
)
```

---

#### **10. Campos Especiais e Seus Parâmetros**

**a) `DateTimeField` e `DateField`**
```python
criado_em = serializers.DateTimeField(
    format='%d/%m/%Y %H:%M',  # Formato de saída
    input_formats=['%d/%m/%Y', 'iso-8601'],  # Formatos de entrada aceitos
    default_timezone=timezone.utc,
    allow_null=False
)
```

**b) `DecimalField`**
```python
preco = serializers.DecimalField(
    max_digits=10,
    decimal_places=2,
    coerce_to_string=False,  # Mantém como Decimal em vez de string
    max_value=10000.00,
    min_value=0.01,
    rounding='ROUND_HALF_UP'
)
```

**c) `ListField` e `DictField`**
```python
tags = serializers.ListField(
    child=serializers.CharField(max_length=30),
    min_length=1,
    max_length=10
)

metadata = serializers.DictField(
    child=serializers.CharField(),
    allow_empty=True
)
```

**d) `FileField` e `ImageField`**
```python
avatar = serializers.ImageField(
    max_length=100,
    allow_empty_file=False,
    use_url=True  # Retorna URL completa
)
```

**e) `SerializerMethodField` (Aprimorado)**
```python
class ProdutoSerializer(serializers.ModelSerializer):
    status_estoque = serializers.SerializerMethodField(method_name='get_status')
    
    def get_status(self, obj):
        if obj.estoque > 50: return 'alto'
        if obj.estoque > 10: return 'médio'
        return 'baixo'
```

---

#### **11. Campos de Relacionamento Avançados**

**a) `HyperlinkedRelatedField`**
```python
categoria = serializers.HyperlinkedRelatedField(
    view_name='categoria-detail',
    queryset=Categoria.objects.all(),
    lookup_field='slug',  # Usa slug em vez de pk
    lookup_url_kwarg='categoria_slug'
)
```

**b) `HyperlinkedIdentityField`**
```python
class ProdutoSerializer(serializers.ModelSerializer):
    url = serializers.HyperlinkedIdentityField(
        view_name='produto-detail',
        lookup_field='codigo',
        lookup_url_kwarg='produto_cod'
    )
```

**c) `PrimaryKeyRelatedField` com validação**
```python
itens = serializers.PrimaryKeyRelatedField(
    many=True,
    queryset=Item.objects.all(),
    pk_field=serializers.UUIDField(format='hex'),  # Para UUIDs
    error_messages={
        'does_not_exist': 'Item ID {pk_value} não existe',
        'incorrect_type': 'Tipo de dado inválido'
    }
)
```

**d) `SlugRelatedField` com fallback**
```python
vendedor = serializers.SlugRelatedField(
    slug_field='username',
    queryset=User.objects.all(),
    allow_null=True,
    default=None
)
```

---

#### **12. Relacionamentos Aninhados Customizados**

**a) Nested serializer com lógica de criação/atualização**
```python
class TrackSerializer(serializers.ModelSerializer):
    class Meta:
        model = Track
        fields = ['order', 'title', 'duration']

class AlbumSerializer(serializers.ModelSerializer):
    tracks = TrackSerializer(many=True)

    def create(self, validated_data):
        tracks_data = validated_data.pop('tracks')
        album = Album.objects.create(**validated_data)
        
        for track_data in tracks_data:
            Track.objects.create(album=album, **track_data)
            
        return album

    def update(self, instance, validated_data):
        tracks_data = validated_data.pop('tracks')
        
        # Atualiza campos do álbum
        instance.album_name = validated_data.get('album_name', instance.album_name)
        instance.save()
        
        # Atualiza/remove tracks
        track_ids = [item.get('id') for item in tracks_data if 'id' in item]
        
        # Remove tracks não presentes nos dados
        instance.tracks.exclude(id__in=track_ids).delete()
        
        for track_data in tracks_data:
            track_id = track_data.get('id', None)
            
            if track_id:
                track = Track.objects.get(id=track_id, album=instance)
                track.order = track_data.get('order', track.order)
                track.title = track_data.get('title', track.title)
                track.duration = track_data.get('duration', track.duration)
                track.save()
            else:
                Track.objects.create(album=instance, **track_data)
                
        return instance
```

**b) Relacionamento genérico (GenericForeignKey)**
```python
class TaggedObjectField(serializers.RelatedField):
    def to_representation(self, value):
        if isinstance(value, Bookmark):
            return f'Bookmark: {value.url}'
        elif isinstance(value, Note):
            return f'Note: {value.text}'
        raise Exception('Tipo não suportado')

class TagSerializer(serializers.ModelSerializer):
    content_object = TaggedObjectField(read_only=True)
    
    class Meta:
        model = Tag
        fields = ['id', 'tag_name', 'content_object']
```

---

#### **13. Campos Personalizados (Custom Fields)**

**Exemplo 1: Campo de cor RGB**
```python
class ColorField(serializers.Field):
    default_error_messages = {
        'invalid': 'Formato inválido. Use "rgb(r,g,b)"',
        'range': 'Valores devem estar entre 0 e 255'
    }
    
    def to_representation(self, value):
        return f"rgb({value.red},{value.green},{value.blue})"
    
    def to_internal_value(self, data):
        pattern = r'^rgb\((\d+),(\d+),(\d+)\)$'
        match = re.match(pattern, data.strip())
        
        if not match:
            self.fail('invalid')
            
        red, green, blue = map(int, match.groups())
        
        if not all(0 <= x <= 255 for x in (red, green, blue)):
            self.fail('range')
            
        return Color(red, green, blue)
```

**Exemplo 2: Campo composto (coordenadas X/Y)**
```python
class CoordinateField(serializers.Field):
    def to_representation(self, obj):
        return {'x': obj.x_coord, 'y': obj.y_coord}
    
    def to_internal_value(self, data):
        return {
            'x_coordinate': data['x'],
            'y_coordinate': data['y']
        }
```

---

### Checklist Ampliado (20/80 para Campos e Relacionamentos)

1. **Parâmetros universais:** Domine `read_only`, `write_only`, `source`, e `default`
2. **Campos temporais:** Use `DateTimeField` com `format` e `input_formats`
3. **Campos numéricos:** 
   - `DecimalField` com `max_digits` e `decimal_places`
   - `IntegerField` com `min_value`/`max_value`
4. **Listas e dicionários:** Use `ListField` e `DictField` para estruturas complexas
5. **Relacionamentos:**
   - `HyperlinkedRelatedField` para APIs HATEOAS
   - `PrimaryKeyRelatedField` para eficiência
   - `SlugRelatedField` para URLs amigáveis
6. **Nested serializers:** Implemente `create()` e `update()` para relações aninhadas
7. **Campos personalizados:** Crie fields customizados para necessidades específicas
8. **Validação avançada:** 
   - Use `validators` para regras complexas
   - Personalize `error_messages`
9. **Performance:** 
   - Use `select_related` e `prefetch_related` em views
   - Evite nested serializers profundos
10. **Relacionamentos reversos:** Use `related_name` em models e declare explicitamente em serializers

### Dicas Cruciais de Relacionamentos

1. **Performance em relacionamentos:**
```python
# View (evitar N+1)
queryset = Album.objects.prefetch_related('tracks')

# Serializer (evitar nested por padrão)
class AlbumSerializer(serializers.ModelSerializer):
    # Em vez de nested, use hyperlinks ou IDs
    tracks = serializers.HyperlinkedRelatedField(
        many=True,
        view_name='track-detail',
        read_only=True
    )
```

2. **Para ManyToMany com through model:**
```python
class MembershipSerializer(serializers.ModelSerializer):
    class Meta:
        model = Membership
        fields = ['id', 'date_joined', 'status']

class GroupSerializer(serializers.ModelSerializer):
    members = MembershipSerializer(
        source='membership_set', 
        many=True,
        read_only=True
    )
```

3. **Relacionamentos genéricos:**
```python
class TagSerializer(serializers.ModelSerializer):
    content_object = serializers.SerializerMethodField()

    def get_content_object(self, obj):
        if isinstance(obj.content_object, Bookmark):
            return BookmarkSerializer(obj.content_object).data
        elif isinstance(obj.content_object, Note):
            return NoteSerializer(obj.content_object).data
        return None
```

Estas adições cobrem os aspectos mais críticos dos campos e relacionamentos no DRF, resolvendo a maioria dos cenários encontrados no desenvolvimento de APIs Django.