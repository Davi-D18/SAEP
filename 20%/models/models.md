### Guia Essencial 20/80: Models no Django

Os models são o coração do Django, definindo a estrutura do banco de dados e contendo a lógica de negócio. Vamos focar nos 20% mais importantes que resolvem 80% dos problemas.

---

#### **1. Estrutura Básica de um Model**
```python
from django.db import models

class Produto(models.Model):
    # Campos aqui
    nome = models.CharField(max_length=100)
    preco = models.DecimalField(max_digits=10, decimal_places=2)
    criado_em = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.nome
```

---

#### **2. Tipos de Campos e Parâmetros Essenciais**

| Campo                  | Descrição                          | Parâmetros Comuns                          |
|------------------------|-----------------------------------|-------------------------------------------|
| **`CharField`**        | Texto curto                      | `max_length`, `blank=True`, `null=True`   |
| **`TextField`**        | Texto longo                      | `blank=True`, `null=True`                 |
| **`IntegerField`**     | Números inteiros                 | `default=0`, `blank=True`, `null=True`    |
| **`DecimalField`**     | Números decimais                 | `max_digits`, `decimal_places`            |
| **`BooleanField`**     | True/False                       | `default=False`                           |
| **`DateTimeField`**    | Data e hora                      | `auto_now_add` (criação), `auto_now` (atualização) |
| **`DateField`**        | Apenas data                      | Mesmos que DateTimeField                  |
| **`EmailField`**       | Email com validação              | `max_length=254`, `unique=True`           |
| **`URLField`**         | URLs                             | `max_length=200`, `blank=True`            |
| **`FileField`**        | Upload de arquivos               | `upload_to='pasta/'`                      |
| **`ImageField`**       | Upload de imagens                | `upload_to='imagens/'`, `height_field`, `width_field` |

**Parâmetros universais:**
- `blank=True`: Permite campo vazio no formulário
- `null=True`: Permite NULL no banco de dados
- `default`: Valor padrão
- `unique=True`: Valor único no banco
- `choices`: Opções pré-definidas
  ```python
  TAMANHOS = [
      ('P', 'Pequeno'),
      ('M', 'Médio'),
      ('G', 'Grande')
  ]
  tamanho = models.CharField(max_length=1, choices=TAMANHOS, default='M')
  ```

---

#### **3. Relacionamentos (Chaves)**

**a) ForeignKey (1:N ou N:1)**
```python
class Categoria(models.Model):
    nome = models.CharField(max_length=50)

class Produto(models.Model):
    categoria = models.ForeignKey(
        Categoria,
        on_delete=models.CASCADE,
        related_name='produtos'
    )
```
**Parâmetros:**
- `on_delete`: O que acontece quando o objeto relacionado é deletado:
  - `models.CASCADE`: Deleta todos os produtos da categoria
  - `models.PROTECT`: Impede deleção da categoria se houver produtos
  - `models.SET_NULL`: Define categoria como NULL (precisa `null=True`)
  - `models.SET_DEFAULT`: Usa valor padrão
- `related_name`: Nome para acessar produtos a partir da categoria
  ```python
  # Uso:
  categoria = Categoria.objects.get(nome="Eletrônicos")
  produtos = categoria.produtos.all()
  ```

**b) ManyToManyField (N:N)**
```python
class Tag(models.Model):
    nome = models.CharField(max_length=30)

class Produto(models.Model):
    tags = models.ManyToManyField(Tag, related_name='produtos')
```
**Parâmetros:**
- `through`: Para relações customizadas (dados extras)
  ```python
  class ProdutoTag(models.Model):
      produto = models.ForeignKey(Produto, on_delete=models.CASCADE)
      tag = models.ForeignKey(Tag, on_delete=models.CASCADE)
      data_associacao = models.DateTimeField(auto_now_add=True)

  class Produto(models.Model):
      tags = models.ManyToManyField(Tag, through='ProdutoTag')
  ```

**c) OneToOneField (1:1)**
```python
class Perfil(models.Model):
    usuario = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='perfil'
    )
    telefone = models.CharField(max_length=20)
```

---

#### **4. Métodos em Models**

**a) Métodos básicos**
```python
class Pedido(models.Model):
    total = models.DecimalField(max_digits=10, decimal_places=2)
    pago = models.BooleanField(default=False)

    def marcar_como_pago(self):
        self.pago = True
        self.save()

    def aplicar_desconto(self, percentual):
        desconto = self.total * (percentual / 100)
        self.total -= desconto
        self.save()
```

**b) Propriedades calculadas**
```python
@property
def preco_com_taxa(self):
    return self.preco * 1.1  # Adiciona 10% de taxa
```
Uso: `produto.preco_com_taxa` (sem parênteses)

**c) Sobrescrevendo métodos padrão**
```python
def save(self, *args, **kwargs):
    # Antes de salvar
    if not self.slug:
        self.slug = slugify(self.nome)
    super().save(*args, **kwargs)  # Chama o save original

def delete(self, *args, **kwargs):
    # Antes de deletar
    enviar_email_administrador(f"Produto {self.nome} deletado")
    super().delete(*args, **kwargs)
```

---

#### **5. Classe Meta**
Configurações avançadas do model:
```python
class Produto(models.Model):
    # ...
    
    class Meta:
        verbose_name = 'Produto'
        verbose_name_plural = 'Produtos'
        ordering = ['-criado_em']  # Ordenação padrão
        indexes = [
            models.Index(fields=['nome']),  # Cria índice no banco
            models.Index(fields=['preco'], name='preco_idx')
        ]
        constraints = [
            models.UniqueConstraint(
                fields=['nome', 'categoria'], 
                name='nome_categoria_unica'
            ),
            models.CheckConstraint(
                check=models.Q(preco__gte=0),
                name='preco_nao_negativo'
            )
        ]
```

**Opções comuns:**
- `db_table`: Nome personalizado da tabela
- `abstract`: Para criar models base abstratos
- `default_related_name`: Nome padrão para relações reversas

---

#### **6. Boas Práticas Essenciais**

1. **Nomenclatura:**
   - Models no singular (`Produto`, não `Produtos`)
   - Campos em snake_case (`preco_medio`, não `precoMedio`)

2. **Relacionamentos:**
   - Sempre defina `related_name` em ForeignKeys e ManyToManyFields
   - Pense cuidadosamente no `on_delete`

3. **Campos calculados:**
   - Use `@property` para dados derivados que não precisam ser armazenados
   - Use métodos para operações que modificam o estado

4. **Performance:**
   - Adicione `db_index=True` em campos frequentemente filtrados
   - Use `select_related()` e `prefetch_related()` em consultas

5. **Validação:**
   - Valide dados no nível do model com `clean()`:
     ```python
     def clean(self):
         if self.preco < 0:
             raise ValidationError("Preço não pode ser negativo")
     ```

6. **Sinais (Signals):**
   - Use para lógica que responde a eventos:
     ```python
     from django.db.models.signals import pre_save
     from django.dispatch import receiver
     
     @receiver(pre_save, sender=Produto)
     def atualizar_slug(sender, instance, **kwargs):
         if not instance.slug:
             instance.slug = slugify(instance.nome)
     ```

---

#### **Exemplo Completo**
```python
from django.db import models
from django.utils.text import slugify
from django.core.validators import MinValueValidator

class Categoria(models.Model):
    nome = models.CharField(max_length=50, unique=True)
    slug = models.SlugField(unique=True)
    
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.nome)
        super().save(*args, **kwargs)
    
    def __str__(self):
        return self.nome

class Produto(models.Model):
    nome = models.CharField(max_length=100)
    slug = models.SlugField(blank=True)
    descricao = models.TextField(blank=True)
    preco = models.DecimalField(
        max_digits=10,
        decimal_places=2,
        validators=[MinValueValidator(0.01)]
    )
    categoria = models.ForeignKey(
        Categoria,
        on_delete=models.PROTECT,
        related_name='produtos'
    )
    tags = models.ManyToManyField('Tag', related_name='produtos')
    criado_em = models.DateTimeField(auto_now_add=True)
    atualizado_em = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-criado_em']
        indexes = [models.Index(fields=['slug'])]
        constraints = [
            models.UniqueConstraint(
                fields=['nome', 'categoria'],
                name='nome_categoria_unico'
            )
        ]
    
    @property
    def preco_com_taxa(self):
        return self.preco * 1.1
    
    def save(self, *args, **kwargs):
        if not self.slug:
            self.slug = slugify(self.nome)
        super().save(*args, **kwargs)
    
    def __str__(self):
        return f"{self.nome} (R${self.preco})"

class Tag(models.Model):
    nome = models.CharField(max_length=30, unique=True)
    
    def __str__(self):
        return self.nome
```

---

### **Checklist 20/80 para Models**
1. **Campos:** Escolha o tipo correto para cada dado
2. **Parâmetros:** Use `blank`, `null`, `default` e `choices` quando necessário
3. **Relacionamentos:**
   - `ForeignKey` para 1:N
   - `ManyToManyField` para N:N
   - `OneToOneField` para 1:1
4. **Métodos:** Adicione lógica de negócio com métodos e `@property`
5. **Meta:** Configure `ordering`, `indexes` e `constraints`
6. **Validação:** Implemente `clean()` e validadores
7. **Sinais:** Use para operações pós/persalvamento
8. **__str__:** Defina representação legível

Dominando esses 20%, você resolve 80% dos desafios com Models Django! 🚀