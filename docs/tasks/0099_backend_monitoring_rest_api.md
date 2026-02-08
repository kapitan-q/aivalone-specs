# Задача 0099: REST API Controllers

## Контекст

Presentation Layer для Monitoring Context — REST API контроллеры, обрабатывающие HTTP запросы и делегирующие логику в Application Layer через Commands и Queries.

## Цель

Создать REST API контроллеры для всех endpoints Monitoring Context.

## Спецификация

- [API Endpoints](../backend/monitoring/api/endpoints.md)
- [Services Overview](../backend/monitoring/services/overview.md)

## Файлы для создания

```
src/Context/Monitoring/Presentation/Http/Controller/
├── MonitoringProfileController.php
├── FilterController.php
├── SessionController.php
├── GroupController.php
└── LimitsController.php

src/Context/Monitoring/Presentation/Http/Request/
├── CreateFilterRequest.php
├── UpdateFilterRequest.php
├── AddGroupRequest.php
└── StartAuthRequest.php

src/Context/Monitoring/Presentation/Http/Response/
├── ProfileResponse.php
├── FilterResponse.php
├── FilterListResponse.php
├── SessionResponse.php
├── SessionListResponse.php
├── GroupResponse.php
├── GroupListResponse.php
└── LimitsResponse.php

tests/Integration/Context/Monitoring/Presentation/Http/Controller/
├── MonitoringProfileControllerTest.php
├── FilterControllerTest.php
├── SessionControllerTest.php
├── GroupControllerTest.php
└── LimitsControllerTest.php
```

## Требования

### MonitoringProfileController

```php
#[Route('/api/monitoring')]
class MonitoringProfileController extends AbstractController
{
    public function __construct(
        private CommandBusInterface $commandBus,
        private QueryBusInterface $queryBus,
    ) {}

    /** GET /api/monitoring/profile */
    #[Route('/profile', methods: ['GET'])]
    public function getProfile(): JsonResponse
    {
        // 1. Получить userId из JWT token
        // 2. GetMonitoringProfileByUserIdQuery
        // 3. Вернуть ProfileResponse (200)
        // 4. Или 404 если профиль не найден
    }

    /** POST /api/monitoring/profile */
    #[Route('/profile', methods: ['POST'])]
    public function createProfile(): JsonResponse
    {
        // 1. Получить userId из JWT token
        // 2. CreateMonitoringProfileCommand
        // 3. Вернуть ProfileResponse (201)
        // 4. Или 409 если профиль уже существует
    }
}
```

### FilterController

```php
#[Route('/api/monitoring/filters')]
class FilterController extends AbstractController
{
    /** GET /api/monitoring/filters */
    #[Route('', methods: ['GET'])]
    public function list(Request $request): JsonResponse;

    /** POST /api/monitoring/filters */
    #[Route('', methods: ['POST'])]
    public function create(CreateFilterRequest $request): JsonResponse;

    /** GET /api/monitoring/filters/{filterId} */
    #[Route('/{filterId}', methods: ['GET'])]
    public function get(string $filterId): JsonResponse;

    /** PATCH /api/monitoring/filters/{filterId} */
    #[Route('/{filterId}', methods: ['PATCH'])]
    public function update(string $filterId, UpdateFilterRequest $request): JsonResponse;

    /** DELETE /api/monitoring/filters/{filterId} */
    #[Route('/{filterId}', methods: ['DELETE'])]
    public function delete(string $filterId): JsonResponse;

    /** POST /api/monitoring/filters/{filterId}/activate */
    #[Route('/{filterId}/activate', methods: ['POST'])]
    public function activate(string $filterId): JsonResponse;

    /** POST /api/monitoring/filters/{filterId}/deactivate */
    #[Route('/{filterId}/deactivate', methods: ['POST'])]
    public function deactivate(string $filterId): JsonResponse;

    /** POST /api/monitoring/filters/{filterId}/groups/{groupId} */
    #[Route('/{filterId}/groups/{groupId}', methods: ['POST'])]
    public function bindToGroup(string $filterId, string $groupId): JsonResponse;

    /** DELETE /api/monitoring/filters/{filterId}/groups/{groupId} */
    #[Route('/{filterId}/groups/{groupId}', methods: ['DELETE'])]
    public function unbindFromGroup(string $filterId, string $groupId): JsonResponse;
}
```

### SessionController

```php
#[Route('/api/monitoring/sessions')]
class SessionController extends AbstractController
{
    /** GET /api/monitoring/sessions */
    #[Route('', methods: ['GET'])]
    public function list(): JsonResponse;

    /** GET /api/monitoring/sessions/{sessionId} */
    #[Route('/{sessionId}', methods: ['GET'])]
    public function get(string $sessionId): JsonResponse;

    /** DELETE /api/monitoring/sessions/{sessionId} */
    #[Route('/{sessionId}', methods: ['DELETE'])]
    public function stop(string $sessionId): JsonResponse;
}
```

### GroupController

```php
#[Route('/api/monitoring/groups')]
class GroupController extends AbstractController
{
    /** GET /api/monitoring/groups */
    #[Route('', methods: ['GET'])]
    public function list(Request $request): JsonResponse;

    /** POST /api/monitoring/groups */
    #[Route('', methods: ['POST'])]
    public function add(AddGroupRequest $request): JsonResponse;

    /** GET /api/monitoring/groups/{groupId} */
    #[Route('/{groupId}', methods: ['GET'])]
    public function get(string $groupId): JsonResponse;

    /** DELETE /api/monitoring/groups/{groupId} */
    #[Route('/{groupId}', methods: ['DELETE'])]
    public function remove(string $groupId): JsonResponse;
}
```

### LimitsController

```php
#[Route('/api/monitoring')]
class LimitsController extends AbstractController
{
    /** GET /api/monitoring/limits */
    #[Route('/limits', methods: ['GET'])]
    public function getLimits(): JsonResponse
    {
        // 1. Получить userId из JWT token
        // 2. Запросить лимиты через BillingLimitsClient
        // 3. Запросить текущие показатели из профиля
        // 4. Вернуть LimitsResponse (200)
    }
}
```

### Request DTO

```php
final class CreateFilterRequest
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 3, max: 500)]
    public string $value;

    #[Assert\NotBlank]
    #[Assert\Choice(choices: ['keyword', 'regex'])]
    public string $filterType;

    #[Assert\Length(max: 255)]
    public ?string $name = null;
}

final class UpdateFilterRequest
{
    #[Assert\Length(min: 3, max: 500)]
    public ?string $value = null;

    #[Assert\Length(max: 255)]
    public ?string $name = null;
}

final class AddGroupRequest
{
    #[Assert\NotBlank]
    #[Assert\Length(max: 255)]
    public string $externalGroupId;

    #[Assert\NotBlank]
    #[Assert\Length(max: 255)]
    public string $groupTitle;

    #[Assert\NotBlank]
    #[Assert\Choice(choices: ['telegram'])]
    public string $messengerType;

    public bool $isPrivate = false;

    /** Обязательно для приватных групп */
    public ?string $sessionId = null;
}

final class StartAuthRequest
{
    #[Assert\NotBlank]
    #[Assert\Choice(choices: ['telegram'])]
    public string $messengerType;

    /** @var array<string, string> */
    public array $authData = [];
}
```

### Коды ошибок и HTTP статусы

| Код ошибки | HTTP | Exception |
|------------|------|-----------|
| `monitoring.profile.not_found` | 404 | ProfileNotFoundException |
| `monitoring.profile.already_exists` | 409 | ProfileAlreadyExistsException |
| `monitoring.profile.not_active` | 400 | ProfileNotActiveException |
| `monitoring.filter.not_found` | 404 | FilterNotFoundException |
| `monitoring.filter.already_exists` | 409 | FilterAlreadyExistsException |
| `monitoring.filter.limit_reached` | 403 | FilterLimitReachedException |
| `monitoring.filter.type_not_allowed` | 403 | FilterTypeNotAllowedException |
| `monitoring.filter.invalid_regex` | 400 | InvalidRegexPatternException |
| `monitoring.session.not_found` | 404 | SessionNotFoundException |
| `monitoring.session.already_exists` | 409 | SessionAlreadyExistsException |
| `monitoring.session.not_authorized` | 400 | SessionNotAuthorizedException |
| `monitoring.session.invalid_state` | 400 | InvalidSessionStateException |
| `monitoring.group.not_found` | 404 | GroupNotFoundException |
| `monitoring.group.already_exists` | 409 | GroupAlreadyExistsException |
| `monitoring.group.limit_reached` | 403 | GroupLimitReachedException |
| `monitoring.group.requires_session` | 400 | SessionNotAuthorizedException |
| `monitoring.binding.already_exists` | 409 | BindingAlreadyExistsException |
| `monitoring.binding.not_found` | 404 | BindingNotFoundException |

## Тесты (Integration)

### MonitoringProfileController
- [x] GET /profile — 200 при существующем профиле
- [x] GET /profile — 404 при отсутствии профиля
- [x] POST /profile — 201 при создании нового профиля
- [x] POST /profile — 409 при дублировании

### FilterController
- [x] GET /filters — 200 со списком фильтров
- [x] GET /filters?isActive=true — фильтрация по статусу
- [x] POST /filters — 201 при создании keyword фильтра
- [x] POST /filters — 201 при создании regex фильтра
- [x] POST /filters — 400 при невалидном regex
- [x] POST /filters — 403 при превышении лимита
- [x] POST /filters — 409 при дублировании
- [x] PATCH /filters/{id} — 200 при обновлении
- [x] DELETE /filters/{id} — 204 при удалении
- [x] POST /filters/{id}/activate — 200
- [x] POST /filters/{id}/deactivate — 200
- [x] POST /filters/{id}/groups/{id} — 201 при привязке
- [x] DELETE /filters/{id}/groups/{id} — 204 при отвязке

### SessionController
- [x] GET /sessions — 200 со списком сессий
- [x] GET /sessions/{id} — 200 с деталями сессии
- [x] DELETE /sessions/{id} — 200 при деактивации

### GroupController
- [x] GET /groups — 200 со списком групп
- [x] GET /groups?messengerType=telegram — фильтрация
- [x] POST /groups — 201 при добавлении публичной группы
- [x] POST /groups — 201 при добавлении приватной группы
- [ ] POST /groups — 400 при приватной группе без сессии
- [x] POST /groups — 403 при превышении лимита
- [x] POST /groups — 409 при дублировании
- [x] DELETE /groups/{id} — 200 при удалении

### LimitsController
- [x] GET /limits — 200 с текущими лимитами

## Зависимости

- MonitoringService (задача 0091)
- Filter Management Handlers (задача 0092)
- SessionCoordinatorService (задача 0093)
- Group Management Handlers (задача 0094)
- BillingLimitsClientInterface (задача 0097)
- Exceptions (задача 0081)
- JWT Authentication (Account Context)

## Definition of Done

- [x] Все контроллеры реализованы
- [x] Request DTO с валидацией реализованы
- [x] Response DTO реализованы
- [x] Маппинг исключений на HTTP коды настроен
- [x] Интеграционные тесты написаны и проходят
- [x] Все endpoints доступны и корректно обрабатывают запросы
- [ ] Код соответствует PSR-12
- [ ] PHPStan level 9 без ошибок
