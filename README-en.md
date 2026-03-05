# Yandex SpeechKit PHP SDK

![Yandex SpeechKit PHP SDK](https://github.com/user-attachments/assets/60dac329-9959-4856-b44f-ada33a9685e3)

[![PHP Version](https://img.shields.io/badge/php-%5E8.0-blue)](https://www.php.net/)
[![Laravel](https://img.shields.io/badge/laravel-%5E8.0%7C%5E9.0%7C%5E10.0%7C%5E11.0%7C%5E12.0-red)](https://laravel.com/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

A PHP SDK that makes it easy to work with [Yandex SpeechKit API](https://cloud.yandex.com/en/services/speechkit). Works
great on its own or as a Laravel package. If you need to transcribe audio, identify speakers, or run speech analytics
from PHP — this is what you're looking for.

**Язык:** [Русский](README.md) | English

## What it does

- Async speech recognition for long audio (up to 4 hours / 1 GB)
- Supports WAV, OGG_OPUS, MP3, and raw PCM formats
- Speaker labeling (diarization) — tells you who said what
- Text normalization and profanity filtering out of the box
- Multi-language recognition
- Plugs right into Laravel via Facade and Service Provider
- Covered with tests

## Ecosystem

This package is part of a small family of Yandex Cloud PHP libraries:

- **[tigusigalpa/yandex-cloud-client-php](https://github.com/tigusigalpa/yandex-cloud-client-php)** — handles
  authentication: OAuth tokens, IAM token generation and auto-refresh
- **[tigusigalpa/yandexgpt-php](https://github.com/tigusigalpa/yandexgpt-php)** — YandexGPT integration

You don't need to worry about auth tokens — `yandex-cloud-client-php` takes care of IAM token generation and refresh for
you.

## Requirements

- PHP ^8.0
- Laravel ^8.0|^9.0|^10.0|^11.0|^12.0 (optional, for Laravel integration)
- tigusigalpa/yandex-cloud-client-php ^1.0

## Installation

```bash
composer require tigusigalpa/yandex-speechkit-php
```

## Laravel Setup

### 1. Publish the config

```bash
php artisan vendor:publish --tag=yandex-speechkit-config
```

This creates `config/yandex-speechkit.php`.

### 2. Set up your `.env`

You'll need at least a folder ID and one of the auth methods:

```env
# Auth via OAuth token (recommended)
YANDEX_OAUTH_TOKEN=your_oauth_token
YANDEX_FOLDER_ID=your_folder_id

# ...or via API key
YANDEX_API_KEY=your_api_key
YANDEX_FOLDER_ID=your_folder_id

# These are optional — defaults work fine for most cases
YANDEX_SPEECHKIT_MODEL=general
YANDEX_SPEECHKIT_POLL_INTERVAL=10
YANDEX_SPEECHKIT_MAX_WAIT=14400
```

### 3. Service Provider

Auto-registered via Laravel's package discovery — nothing to do here.

## Usage

### Plain PHP (no Laravel)

```php
use Tigusigalpa\YandexCloudClient\YandexCloudClient;
use Tigusigalpa\YandexSpeechKit\YandexSpeechKitClient;
use Tigusigalpa\YandexSpeechKit\Models\RecognitionRequest;
use Tigusigalpa\YandexSpeechKit\Models\AudioFormat;

// Set up
$cloudClient = new YandexCloudClient('your_oauth_token');
$client = new YandexSpeechKitClient($cloudClient, 'your_folder_id');

// Build a request
$request = new RecognitionRequest(
    uri: 'https://storage.yandexcloud.net/my-bucket/audio.wav',
    model: 'general',
    audioFormat: AudioFormat::container('WAV'),
);

// The easiest way — just call recognizeAndWait() and it handles polling for you
$result = $client->recognizeAndWait($request);
echo "Transcription: " . $result->fullText . "\n";

// If you need more control, you can poll manually:
$operation = $client->recognizeFileAsync($request);
echo "Operation ID: " . $operation->id . "\n";

do {
    sleep(10);
    $operation = $client->getOperation($operation->id);
    echo "Status: " . ($operation->isDone() ? 'done' : 'processing') . "\n";
} while (!$operation->isDone());

$result = $client->getRecognition($operation->id);
echo "Transcription: " . $result->fullText . "\n";
```

### With Laravel

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

### Full example with all the bells and whistles

```php
use Tigusigalpa\YandexSpeechKit\Models\RecognitionRequest;
use Tigusigalpa\YandexSpeechKit\Models\AudioFormat;
use Tigusigalpa\YandexSpeechKit\Models\TextNormalization;
use Tigusigalpa\YandexSpeechKit\Models\LanguageRestriction;
use Tigusigalpa\YandexSpeechKit\Models\SpeakerLabeling;

// Configure everything: normalization, language filter, speaker labeling
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

// Fire off the recognition
$operation = $client->recognizeFileAsync($request);
echo "Operation ID: " . $operation->id . "\n";

// Wait for it...
do {
    sleep(10);
    $operation = $client->getOperation($operation->id);
    echo "Status: " . ($operation->isDone() ? 'done' : 'processing') . "\n";
} while (!$operation->isDone());

// Something went wrong?
if ($operation->hasError()) {
    throw new \RuntimeException($operation->getErrorMessage());
}

// Grab the results
$result = $client->getRecognition($operation->id);
echo "Transcription: " . $result->fullText . "\n";
echo "Words count: " . count($result->words) . "\n";

// You can also walk through individual words with their timestamps
foreach ($result->words as $word) {
    echo sprintf(
        "%s [%s - %s]\n",
        $word['text'],
        $word['startTimeMs'],
        $word['endTimeMs']
    );
}

// Don't forget to clean up when you're done
$client->deleteRecognition($operation->id);
echo "Results deleted.\n";
```

### Passing audio as base64

Don't have a remote URL? You can pass the audio content directly:

```php
$audioContent = base64_encode(file_get_contents('/path/to/audio.wav'));

$request = new RecognitionRequest(
    content: $audioContent,
    audioFormat: AudioFormat::container('WAV')
);

$result = $client->recognizeAndWait($request);
```

### Working with raw PCM

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

### Cancelling an operation

Changed your mind? No problem:

```php
$operation = $client->recognizeFileAsync($request);

$cancelledOperation = $client->cancelOperation($operation->id);
echo "Operation cancelled: " . $cancelledOperation->id . "\n";
```

## API Reference

### Client methods

| Method                                                     | What it does                                  | Returns             |
|------------------------------------------------------------|-----------------------------------------------|---------------------|
| `recognizeFileAsync($request)`                             | Kicks off async recognition                   | `Operation`         |
| `getRecognition($operationId)`                             | Fetches recognition results                   | `RecognitionResult` |
| `deleteRecognition($operationId)`                          | Deletes stored results                        | `bool`              |
| `getOperation($operationId)`                               | Checks operation status                       | `Operation`         |
| `cancelOperation($operationId)`                            | Cancels a running operation                   | `Operation`         |
| `recognizeAndWait($request, $poll = 10, $maxWait = 14400)` | Does everything: submit, poll, return results | `RecognitionResult` |
| `getCloudClient()`                                         | Returns the underlying cloud client           | `YandexCloudClient` |

### Supported audio formats

| Format       | Type      | How to use                                   |
|--------------|-----------|----------------------------------------------|
| WAV          | Container | `AudioFormat::container('WAV')`              |
| OGG_OPUS     | Container | `AudioFormat::container('OGG_OPUS')`         |
| MP3          | Container | `AudioFormat::container('MP3')`              |
| LINEAR16_PCM | Raw       | `AudioFormat::raw('LINEAR16_PCM', 16000, 1)` |

### Recognition models

- `general` — the default, works well for most cases
- `general:rc` — release candidate, newer but less battle-tested
- `general:deprecated` — older version, still available
- `deferred-general` — same quality, but processed in a queue (cheaper)
- `deferred-general:rc` — deferred RC
- `deferred-general:deprecated` — deferred, older version

## Error handling

Three specific exception types so you can react accordingly:

```php
use Tigusigalpa\YandexSpeechKit\Exceptions\AuthenticationException;
use Tigusigalpa\YandexSpeechKit\Exceptions\RecognitionException;
use Tigusigalpa\YandexSpeechKit\Exceptions\OperationException;

try {
    $result = $client->recognizeAndWait($request);
} catch (AuthenticationException $e) {
    // Bad token, expired credentials, etc.
    echo "Auth error: " . $e->getMessage();
} catch (RecognitionException $e) {
    // Something wrong with the audio or request
    echo "Recognition error: " . $e->getMessage();
    echo "API error code: " . $e->getApiErrorCode();
} catch (OperationException $e) {
    // Timeout, cancelled operation, etc.
    echo "Operation error: " . $e->getMessage();
}
```

## Testing

```bash
composer install
vendor/bin/phpunit
```

## Links

- [Yandex SpeechKit docs](https://cloud.yandex.com/en/docs/speechkit/)
- [SpeechKit API reference](https://cloud.yandex.com/en/docs/speechkit/stt/api/transcribation-api)
- [This repo on GitHub](https://github.com/tigusigalpa/yandex-speechkit-php)
- [yandex-cloud-client-php](https://github.com/tigusigalpa/yandex-cloud-client-php) — auth layer
- [yandexgpt-php](https://github.com/tigusigalpa/yandexgpt-php) — YandexGPT integration

## License

MIT. See [LICENSE](LICENSE).

## Author

**Igor Sazonov** — [GitHub](https://github.com/tigusigalpa) · [sovletig@yandex.ru](mailto:sovletig@yandex.ru)

## Contributing

PRs and issues are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).
