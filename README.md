# Yandex SpeechKit PHP SDK

![Yandex SpeechKit PHP SDK](https://github.com/user-attachments/assets/60dac329-9959-4856-b44f-ada33a9685e3)

[![PHP Version](https://img.shields.io/badge/php-%5E8.0-blue)](https://www.php.net/)
[![Laravel](https://img.shields.io/badge/laravel-%5E8.0%7C%5E9.0%7C%5E10.0%7C%5E11.0%7C%5E12.0-red)](https://laravel.com/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

PHP SDK для работы с [Yandex SpeechKit API](https://cloud.yandex.ru/services/speechkit). Работает как самостоятельная
библиотека или как Laravel-пакет. Если вам нужно транскрибировать аудио, определять спикеров или анализировать речь из
PHP — вы по адресу.

**Язык:** Русский | [English](README-en.md)

## Что умеет

- Асинхронное распознавание длинного аудио (до 4 часов / 1 ГБ)
- Форматы WAV, OGG_OPUS, MP3 и raw PCM
- Разделение по спикерам (диаризация) — кто что сказал
- Нормализация текста и фильтрация мата из коробки
- Мультиязычное распознавание
- Встраивается в Laravel через Facade и Service Provider
- Покрыт тестами

## Экосистема

Пакет входит в небольшое семейство PHP-библиотек для Yandex Cloud:

- **[tigusigalpa/yandex-cloud-client-php](https://github.com/tigusigalpa/yandex-cloud-client-php)** — аутентификация:
  OAuth-токены, генерация и автообновление IAM-токенов
- **[tigusigalpa/yandexgpt-php](https://github.com/tigusigalpa/yandexgpt-php)** — интеграция с YandexGPT

Об авторизации можно не думать — `yandex-cloud-client-php` сам разберётся с IAM-токенами.

## Требования

- PHP ^8.0
- Laravel ^8.0|^9.0|^10.0|^11.0|^12.0 (опционально, для интеграции с Laravel)
- tigusigalpa/yandex-cloud-client-php ^1.0

## Установка

```bash
composer require tigusigalpa/yandex-speechkit-php
```

## Настройка Laravel

### 1. Опубликуйте конфиг

```bash
php artisan vendor:publish --tag=yandex-speechkit-config
```

Создаст файл `config/yandex-speechkit.php`.

### 2. Пропишите `.env`

Минимум нужен folder ID и один из способов авторизации:

```env
# Через OAuth-токен (рекомендуется)
YANDEX_OAUTH_TOKEN=ваш_oauth_токен
YANDEX_FOLDER_ID=ваш_folder_id

# ...или через API-ключ
YANDEX_API_KEY=ваш_api_ключ
YANDEX_FOLDER_ID=ваш_folder_id

# Необязательно — дефолты подходят в большинстве случаев
YANDEX_SPEECHKIT_MODEL=general
YANDEX_SPEECHKIT_POLL_INTERVAL=10
YANDEX_SPEECHKIT_MAX_WAIT=14400
```

### 3. Service Provider

Регистрируется автоматически через package discovery Laravel — ничего делать не нужно.

## Использование

### Чистый PHP (без Laravel)

```php
use Tigusigalpa\YandexCloudClient\YandexCloudClient;
use Tigusigalpa\YandexSpeechKit\YandexSpeechKitClient;
use Tigusigalpa\YandexSpeechKit\Models\RecognitionRequest;
use Tigusigalpa\YandexSpeechKit\Models\AudioFormat;

// Настройка
$cloudClient = new YandexCloudClient('ваш_oauth_токен');
$client = new YandexSpeechKitClient($cloudClient, 'ваш_folder_id');

// Собираем запрос
$request = new RecognitionRequest(
    uri: 'https://storage.yandexcloud.net/my-bucket/audio.wav',
    model: 'general',
    audioFormat: AudioFormat::container('WAV'),
);

// Самый простой способ — recognizeAndWait() сам опрашивает статус и дождётся результата
$result = $client->recognizeAndWait($request);
echo "Транскрипция: " . $result->fullText . "\n";

// Если нужен контроль, можно опрашивать вручную:
$operation = $client->recognizeFileAsync($request);
echo "ID операции: " . $operation->id . "\n";

do {
    sleep(10);
    $operation = $client->getOperation($operation->id);
    echo "Статус: " . ($operation->isDone() ? 'готово' : 'в работе') . "\n";
} while (!$operation->isDone());

$result = $client->getRecognition($operation->id);
echo "Транскрипция: " . $result->fullText . "\n";
```

### С Laravel

```php
use Tigusigalpa\YandexSpeechKit\Laravel\Facades\YandexSpeechKit;
use Tigusigalpa\YandexSpeechKit\Models\RecognitionRequest;
use Tigusigalpa\YandexSpeechKit\Models\AudioFormat;

$request = new RecognitionRequest(
    uri: 'https://storage.yandexcloud.net/my-bucket/audio.wav',
    audioFormat: AudioFormat::container('WAV'),
);

$result = YandexSpeechKit::recognizeAndWait($request);
echo $result->fullText;
```

### Полный пример со всеми настройками

```php
use Tigusigalpa\YandexSpeechKit\Models\RecognitionRequest;
use Tigusigalpa\YandexSpeechKit\Models\AudioFormat;
use Tigusigalpa\YandexSpeechKit\Models\TextNormalization;
use Tigusigalpa\YandexSpeechKit\Models\LanguageRestriction;
use Tigusigalpa\YandexSpeechKit\Models\SpeakerLabeling;

// Настраиваем всё: нормализацию, языки, разделение по спикерам
$request = new RecognitionRequest(
    uri: 'https://storage.yandexcloud.net/my-bucket/audio.wav',
    model: 'general',
    audioFormat: AudioFormat::container('WAV'),
    textNormalization: new TextNormalization(
        textNormalization: 'TEXT_NORMALIZATION_ENABLED',
        profanityFilter: true,
        literatureText: false
    ),
    languageRestriction: new LanguageRestriction(
        restrictionType: 'WHITELIST',
        languageCode: ['ru-RU', 'en-US']
    ),
    speakerLabeling: new SpeakerLabeling('SPEAKER_LABELING_ENABLED')
);

// Запускаем распознавание
$operation = $client->recognizeFileAsync($request);
echo "ID операции: " . $operation->id . "\n";

// Ждём...
do {
    sleep(10);
    $operation = $client->getOperation($operation->id);
    echo "Статус: " . ($operation->isDone() ? 'готово' : 'в работе') . "\n";
} while (!$operation->isDone());

// Что-то пошло не так?
if ($operation->hasError()) {
    throw new \RuntimeException($operation->getErrorMessage());
}

// Забираем результат
$result = $client->getRecognition($operation->id);
echo "Транскрипция: " . $result->fullText . "\n";
echo "Слов: " . count($result->words) . "\n";

// Можно пройтись по отдельным словам с таймкодами
foreach ($result->words as $word) {
    echo sprintf(
        "%s [%s - %s]\n",
        $word['text'],
        $word['startTimeMs'],
        $word['endTimeMs']
    );
}

// Не забудьте почистить за собой
$client->deleteRecognition($operation->id);
echo "Результаты удалены.\n";
```

### Передача аудио в base64

Нет удалённого URL? Можно передать содержимое файла напрямую:

```php
$audioContent = base64_encode(file_get_contents('/path/to/audio.wav'));

$request = new RecognitionRequest(
    content: $audioContent,
    audioFormat: AudioFormat::container('WAV')
);

$result = $client->recognizeAndWait($request);
```

### Работа с raw PCM

```php
use Tigusigalpa\YandexSpeechKit\Models\AudioFormat;

$request = new RecognitionRequest(
    uri: 'https://storage.yandexcloud.net/my-bucket/audio.pcm',
    audioFormat: AudioFormat::raw(
        audioEncoding: 'LINEAR16_PCM',
        sampleRateHertz: 16000,
        audioChannelCount: 1
    )
);
```

### Отмена операции

Передумали? Не проблема:

```php
$operation = $client->recognizeFileAsync($request);

$cancelledOperation = $client->cancelOperation($operation->id);
echo "Операция отменена: " . $cancelledOperation->id . "\n";
```

## Справочник API

### Методы клиента

| Метод                                                      | Что делает                                           | Возвращает          |
|------------------------------------------------------------|------------------------------------------------------|---------------------|
| `recognizeFileAsync($request)`                             | Запускает асинхронное распознавание                  | `Operation`         |
| `getRecognition($operationId)`                             | Забирает результат распознавания                     | `RecognitionResult` |
| `deleteRecognition($operationId)`                          | Удаляет сохранённые результаты                       | `bool`              |
| `getOperation($operationId)`                               | Проверяет статус операции                            | `Operation`         |
| `cancelOperation($operationId)`                            | Отменяет операцию                                    | `Operation`         |
| `recognizeAndWait($request, $poll = 10, $maxWait = 14400)` | Всё в одном: отправить, дождаться, вернуть результат | `RecognitionResult` |
| `getCloudClient()`                                         | Возвращает базовый cloud-клиент                      | `YandexCloudClient` |

### Поддерживаемые форматы

| Формат       | Тип       | Как использовать                             |
|--------------|-----------|----------------------------------------------|
| WAV          | Контейнер | `AudioFormat::container('WAV')`              |
| OGG_OPUS     | Контейнер | `AudioFormat::container('OGG_OPUS')`         |
| MP3          | Контейнер | `AudioFormat::container('MP3')`              |
| LINEAR16_PCM | Raw       | `AudioFormat::raw('LINEAR16_PCM', 16000, 1)` |

### Модели распознавания

- `general` — основная, подходит для большинства задач
- `general:rc` — release candidate, новее, но менее обкатана
- `general:deprecated` — старая версия, пока доступна
- `deferred-general` — то же качество, но обрабатывается в очереди (дешевле)
- `deferred-general:rc` — отложенная RC
- `deferred-general:deprecated` — отложенная, старая версия

## Обработка ошибок

Три типа исключений, чтобы можно было реагировать по ситуации:

```php
use Tigusigalpa\YandexSpeechKit\Exceptions\AuthenticationException;
use Tigusigalpa\YandexSpeechKit\Exceptions\RecognitionException;
use Tigusigalpa\YandexSpeechKit\Exceptions\OperationException;

try {
    $result = $client->recognizeAndWait($request);
} catch (AuthenticationException $e) {
    // Плохой токен, истёкшие credentials и т.п.
    echo "Ошибка авторизации: " . $e->getMessage();
} catch (RecognitionException $e) {
    // Что-то не так с аудио или запросом
    echo "Ошибка распознавания: " . $e->getMessage();
    echo "Код ошибки API: " . $e->getApiErrorCode();
} catch (OperationException $e) {
    // Таймаут, отменённая операция и т.д.
    echo "Ошибка операции: " . $e->getMessage();
}
```

## Тесты

```bash
composer install
vendor/bin/phpunit
```

## Ссылки

- [Документация SpeechKit](https://cloud.yandex.ru/docs/speechkit/)
- [Справочник API](https://cloud.yandex.ru/docs/speechkit/stt/api/transcribation-api)
- [Этот репозиторий на GitHub](https://github.com/tigusigalpa/yandex-speechkit-php)
- [yandex-cloud-client-php](https://github.com/tigusigalpa/yandex-cloud-client-php) — слой авторизации
- [yandexgpt-php](https://github.com/tigusigalpa/yandexgpt-php) — интеграция с YandexGPT

## Лицензия

MIT. См. [LICENSE](LICENSE).

## Автор

**Игорь Сазонов** — [GitHub](https://github.com/tigusigalpa) · [sovletig@gmail.com](mailto:sovletig@gmail.com)

## Участие в разработке

PR и issues приветствуются. См. [CONTRIBUTING.md](CONTRIBUTING.md).
