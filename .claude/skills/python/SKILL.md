---
name: python
description: Conventions et patterns Python/Django pour chaque modification du backend OfficeIn
trigger: always
---

# Conventions Python/Django — Backend OfficeIn

Applique systématiquement ces conventions pour toute modification du backend.

## Style de code

- **Indentation** : 4 espaces
- **Guillemets** : doubles (`"string"`) de préférence
- **Formatage** : f-strings exclusivement (`f"User {user.username}"`)
- **Nommage** :
  - Variables/fonctions : `snake_case` (`pharmacy_id`, `get_invoices_queryset`)
  - Classes : `PascalCase` (`InvoiceSerializer`, `LoginView`)
  - Constantes : `UPPER_SNAKE_CASE` (`MAX_STRING_SIZE`, `INVOICE_DIR`)
  - Fonctions privées : préfixe `_` (`_normalize_vat_rate_key()`)
- **Type hints** : utiliser quand utile, syntaxe Python 3.10+ (`str | None`, `list[str]`)
- **Docstrings** : triple double quotes, uniquement sur les fonctions complexes
- **Langue** : tous les messages user-facing en français

## Organisation des imports

Ordre strict :
1. Bibliothèque standard (`import logging`, `from decimal import Decimal`)
2. Third-party (`rest_framework`, `django`)
3. Django core (`django.db.models`, `django.core.paginator`)
4. DRF imports (`rest_framework.decorators`, `rest_framework.response`)
5. Imports locaux (`rfa.models`, `rfa.serializers`, `rfa.decorators`)

## Vues API

- **Pattern principal** : Function-Based Views avec décorateurs DRF
- **Ordre des décorateurs** : `@extend_schema` → `@api_view` → `@permission_classes` → décorateurs custom
- **Réponses** : toujours `Response()` de DRF, jamais `JsonResponse` (sauf legacy/webhooks)

```python
@extend_schema(
    methods=["GET"],
    description="Description de l'endpoint",
    parameters=[OpenApiParameter(name="q", type=OpenApiTypes.STR, required=False)],
    responses={200: OpenApiResponse(response=MonSerializer(many=True))}
)
@api_view(['GET'])
@permission_classes([IsAuthenticated])
@can_access_banking
def mon_endpoint_api(request):
    """Docstring courte."""
    results = get_queryset(request.user)
    return Response({"results": results})
```

## Serializers

- `ModelSerializer` pour les modèles, `Serializer` pour les données calculées
- `fields = '__all__'` ou liste explicite dans `Meta`
- Champs liés avec `source` : `owner__name = serializers.CharField(source='owner.name', read_only=True)`
- Champs calculés avec `SerializerMethodField()`
- Serializers imbriqués avec `read_only=True`

## Modèles

- Hériter de `BaseModel` (fournit `created_at`/`updated_at`)
- Choices via `TextChoices` (Django 5.2+)
- `unique_together` dans `Meta` pour contraintes composites
- Définir `__str__()` sur chaque modèle
- **Fichier central** : `rfa/models.py` contient les modèles principaux

## Gestion des erreurs

- Structure de réponse erreur : `{'error': 'message'}` ou `{'success': False, 'error': 'message'}`
- Structure de réponse succès : `{'success': True}`, `{'results': [...]}`, ou données sérialisées
- Codes HTTP via `rest_framework.status` (`status.HTTP_400_BAD_REQUEST`)
- `get_object_or_404()` pour les lookups modèle
- Try-except avec exceptions spécifiques, jamais bare `except:`

```python
try:
    pharmacy = request.user.userprofile.pharmacy
except UserProfile.DoesNotExist:
    return Response({'error': 'Profil utilisateur introuvable'}, status=status.HTTP_404_NOT_FOUND)
except Exception as e:
    logger.error(f"Erreur dans mon_endpoint: {e}")
    return Response({'error': "Erreur interne"}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

## Logging

```python
import logging
from officeinwebsite.settings import APP_NAME
logger = logging.getLogger(APP_NAME)
```

- `logger.info()` pour opérations normales
- `logger.error()` pour erreurs
- `logger.exception()` pour exceptions avec traceback
- Toujours avec f-strings et contexte : `logger.error(f"Erreur import facture {invoice_id}: {e}")`

## Requêtes ORM

- `select_related()` pour ForeignKey/OneToOne
- `prefetch_related()` pour M2M/reverse FK
- Annotations avec `F()`, `Q()`, `Case()`, `When()`, `Count()`
- Pagination avec `Paginator`
- Querysets complexes dans des fichiers `querysets.py` séparés

## Permissions et authentification

- Token auth via `rest_framework.authtoken` (header: `Authorization: Token <token>`)
- Décorateurs de permission custom : `@can_access_banking`, `@can_access_invoices`, etc.
- Staff/assistants bypass toutes les vérifications
- `@csrf_exempt` uniquement sur les webhooks

## Structure d'une app

```
app_name/
├── models.py         # Modèles (ou dans rfa/models.py pour les modèles centraux)
├── views.py          # Endpoints API (FBV)
├── serializers.py    # Serializers DRF
├── urls.py           # Routes (webhook_urls + api_urls)
├── tests.py          # Tests
├── querysets.py      # Helpers de requêtes
```

## Tests

- Django `TestCase`
- Fichiers de test : `[app]/tests.py`
- Setup avec `get_or_create()` et `bulk_create()` pour les fixtures
- Lancer : `python manage.py test app.tests.TestClass.test_method --keepdb`
- **Ne pas utiliser pytest.** La base de test est pré-créée en local et préservée via `--keepdb`.
- Pour accélérer : `python manage.py test --exclude-tag=invoice_extraction --keepdb` (exclut les tests d'extraction factures lents)

## URLs

- Pattern RESTful : `/api/<resource>/`, `/api/<resource>/create/`, `/api/<resource>/<id>/update/`
- Nommage : `path('api/...', views.function_name, name="descriptive_name")`
- Webhooks séparés : `webhook_urls = [...]` et `api_urls = [...]`

## Documentation API

- Utiliser `@extend_schema` de drf-spectacular sur chaque endpoint
- Paramètres avec `OpenApiParameter(name=..., type=..., location=..., required=...)`
- Réponses avec `OpenApiResponse` et exemples via `OpenApiExample`

## Configuration

- Fichier unique : `officeinwebsite/settings.py`
- Piloté par variables d'environnement (`ENV_NAME`, `ENV_TYPE`, `IS_LOCAL`)
- Constantes importées depuis settings : `from officeinwebsite.settings import APP_NAME, RED_STATUS, ...`
- Jamais de valeurs hardcodées pour les constantes métier

## Tâches asynchrones

- RQ avec 3 queues : `default`, `crawler`, `admin`
- Enqueue via `enqueue_task_worker(pharmacy, function, ...)`
- Suivi de progression via Django cache
- Task IDs : `IMPORT_INVOICE`, `IMPORT_LCR`, `IMPORT_ORDER`, `IMPORT_REBATE_EXCLUSIONS`
