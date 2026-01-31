# Задача 14: Реализация Health Check Controller

## Описание

Реализовать базовый HTTP контроллер для health-check endpoint, используемого для мониторинга доступности сервиса.

## Требования

- [x] Класс должен быть расположен в `src/Context/Shared/Presentation/Http/Controller/HealthCheckController.php`
- [x] Реализовать GET endpoint `/health`
- [x] Возвращать JSON ответ с статусом сервиса

## Интерфейс

```php
class HealthCheckController extends AbstractController
{
    public function check(): JsonResponse;
}
```

## Endpoint

```
GET /health
Content-Type: application/json

Response (200 OK):
{
    "status": "ok",
    "timestamp": "2026-01-30T10:30:00Z",
    "service": "aivalone-backend"
}
```

## Требования

- [x] HTTP метод: GET
- [x] Route: `/health`
- [x] Response code: 200 OK
- [x] Response format: JSON
- [x] Должно содержать:
  - `status` - статус сервиса ('ok')
  - `timestamp` - текущее время в ISO 8601
  - `service` - имя сервиса

## Использование

```php
/**
 * GET /health
 * 
 * @Route("/health", methods={"GET"})
 */
public function check(): JsonResponse
{
    return $this->json([
        'status' => 'ok',
        'timestamp' => (new DateTimeImmutable())->format(DateTimeInterface::ATOM),
        'service' => 'aivalone-backend'
    ]);
}
```

## Роутинг

```yaml
# config/routes.yaml
health:
    path: /health
    controller: App\Context\Shared\Presentation\Http\Controller\HealthCheckController::check
    methods: [GET]
```

## Тестирование

```bash
curl -X GET http://localhost/health
```

## Критерии готовности

- [x] Контроллер реализован с методом `check()`
- [ ] Роут зарегистрирован
- [ ] Покрыт functional тестами (минимум 1 тест)
- [ ] Документация актуальна

## Зависимости

Нет (базовый компонент)

## Статус

done
