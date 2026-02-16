# php-utorrent

PHP client library for the classic uTorrent Web UI API (`/gui/`).

This fork provides an object-oriented wrapper around uTorrent endpoints and maps API responses into typed model/response objects.

## Table of Contents

-  (checking; constant name mirrors source code)[Overview](#overview)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core API (`UtorrentClient`)](#core-api-utorrentclient)
- [Response APIs](#response-apis)
  - [`BaseResponse`](#baseresponse)
  - [`ListResponse`](#listresponse)
  - [`PropResponse`](#propresponse)
  - [`SettingsResponse`](#settingsresponse)
  - [`FileResponse`](#fileresponse)
- [Model APIs](#model-apis)
  - [`BaseModel`](#basemodel)
  - [`Token`](#token)
  - [`Label`](#label)
  - [`File`](#file)
  - [`SettingItem`](#settingitem)
  - [`Torrent`](#torrent)
  - [`TorrentProp`](#torrentprop)
  - [`RssFeed`](#rssfeed)
  - [`RssItem`](#rssitem)
  - [`RssFilter`](#rssfilter)
- [Notes and Behavior](#notes-and-behavior)
- [Examples](#examples)
- [License](#license)

## Overview

The library handles:

- uTorrent token retrieval and caching
- Authenticated HTTP calls to uTorrent Web UI
- Parsing/sanitizing response payloads
- Mapping uTorrent list/props/settings payloads to PHP objects
- In-memory filtering of result collections using Doctrine `Criteria`

Primary namespace:

- `Pbxg33k\UtorrentClient`

## Requirements

Defined in `composer.json`:

- `psr/cache`
- `doctrine/collections`
- `doctrine/deprecations`
- `guzzlehttp/psr7`
- `guzzlehttp/guzzle`

## Installation

```bash
composer require shubhank008/php-utorrent
```

## Quick Start

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use Cache\Adapter\Void\VoidCachePool;
use Pbxg33k\UtorrentClient\UtorrentClient;

$client = new UtorrentClient('127.0.0.1', 8080, 'admin', 'password', '/gui/');
$client->setCache(new VoidCachePool());

$list = $client->getTorrents();

foreach ($list->getTorrents() as $torrent) {
    echo $torrent->getHash() . ' ' . $torrent->getName() . PHP_EOL;
}
```

## Core API (`UtorrentClient`)

### Constructor

- `__construct($host, $port, $username, $password, $path = '/gui/')`
  - Initializes endpoint and credentials.

### Methods

| Signature | Description |
| --- | --- |
| `getToken()` | Returns a `Model\Token`; uses cache first, then fetches `/token.html` if missing/expired. |
| `setToken(Model\Token $token): UtorrentClient` | Injects a token instance manually. |
| `getTorrents()` | Calls `?list=1` and returns `Response\ListResponse`. |
| `getTorrent(string $hash)` | Returns the first torrent in list response matching the hash. |
| `getTorrentProps(string $hash)` | Calls `?action=getprops&hash=...` and returns `Response\PropResponse`. |
| `getSettings()` | Calls `?action=getsettings` and returns `Response\SettingsResponse`. |
| `addUrl(string $url): ResponseInterface` | Calls `?action=add-url&s=...` to add torrent/magnet by URL. |
| `getHost(): string` | Gets configured host. |
| `setHost(string $address): UtorrentClient` | Sets host. |
| `getPort(): int` | Gets configured port. |
| `setPort(int $port): UtorrentClient` | Sets port. |
| `getPath(): string` | Gets configured GUI path. |
| `setPath(string $path): UtorrentClient` | Sets GUI path. |
| `getUsername(): string` | Gets username. |
| `setUsername(string $username): UtorrentClient` | Sets username. |
| `getPassword(): string` | Gets password. |
| `setPassword(string $password): UtorrentClient` | Sets password. |
| `setCache(CacheItemPoolInterface $cache)` | Sets PSR-6 cache pool used for token/torrent caching. |
| `getClient(): ClientInterface` | Gets current HTTP client; lazily creates `GuzzleHttp\Client` if unset. |
| `setClient(ClientInterface $client): UtorrentClient` | Injects custom HTTP client implementation. |

## Response APIs

### `BaseResponse`

| Signature | Description |
| --- | --- |
| `setCache(CacheItemPoolInterface $cache): BaseResponse` | Assigns cache store for responses that cache model objects. |
| `fromHtml($html)` | Sanitizes control chars and decodes JSON before dispatching to `fromJson`. |
| `toOriginalFormatString(): string` | Serializes response back to uTorrent-like JSON string format. |

### `ListResponse`

| Signature | Description |
| --- | --- |
| `__construct()` | Initializes collections (`labels`, `torrents`, `rssFeeds`, `rssFilters`). |
| `fromJson($json) : ListResponse` | Hydrates build, labels, torrents, RSS feeds/filters; updates cache for torrents. |
| `getBuild(): int` | Returns uTorrent build number. |
| `getLabels(): ArrayCollection` | Returns collection of `Model\Label`. |
| `getTorrents(): ArrayCollection` | Returns collection of `Model\Torrent`. |
| `getTorrentc(): string` | Returns incremental torrent cache token (`torrentc`). |
| `getRssFeeds(): ArrayCollection` | Returns collection of `Model\RssFeed`. |
| `getRssFilters(): ArrayCollection` | Returns collection of `Model\RssFilter`. |
| `filterTorrents(Criteria $criteria): ListResponse` | Replaces torrents collection with criteria-matched subset. |
| `filterRssFeeds(Criteria $criteria): ListResponse` | Replaces RSS feed collection with criteria-matched subset. |
| `filterRssFilters(Criteria $criteria): ListResponse` | Replaces RSS filter collection with criteria-matched subset. |

### `PropResponse`

| Signature | Description |
| --- | --- |
| `__construct()` | Initializes props collection. |
| `fromJson($json)` | Hydrates build and `Model\TorrentProp` items. |
| `getBuild(): int` | Returns uTorrent build number. |
| `getProps(): ArrayCollection` | Returns collection of `Model\TorrentProp`. |

### `SettingsResponse`

| Signature | Description |
| --- | --- |
| `__construct()` | Initializes settings collection. |
| `fromJson($json)` | Hydrates build and `Model\SettingItem` items. |
| `getBuild(): int` | Returns uTorrent build number. |
| `getSettings(): ArrayCollection` | Returns collection of `Model\SettingItem`. |

### `FileResponse`

| Signature | Description |
| --- | --- |
| `__construct()` | Initializes files collection. |
| `fromJson($json)` | Hydrates build, torrent hash and `Model\File` entries from `files` payload. |
| `getBuild(): int` | Returns uTorrent build number. |
| `getFiles(): ArrayCollection` | Returns collection of `Model\File`. |
| `getTorrentHash(): string` | Returns torrent hash that the file list belongs to. |

## Model APIs

### `BaseModel`

| Signature | Description |
| --- | --- |
| `fromHtml($html)` | Decodes JSON and forwards to `fromJson`. |
| `fromJson($json, array $filter = [])` | Hydrates model using `$map` offsets/keys. Optional filter limits updated fields. |
| `toOriginal()` | Converts model back to original uTorrent array/object representation. |
| `refreshCache($newData)` | Hook for partial updates; base implementation is no-op. |

### `Token`

| Signature | Description |
| --- | --- |
| `__construct(string $token)` | Stores token and timestamps (created now, expires +25 min). |
| `getCreatedDateTime(): \DateTimeInterface` | Returns creation datetime. |
| `getExpirationDateTime(): \DateTimeInterface` | Returns expiration datetime. |
| `getToken(): string` | Returns token string value. |
| `__toString()` | Returns token string (or empty string). |

### `Label`

| Signature | Description |
| --- | --- |
| `getName(): string` | Gets label name. |
| `setName(string $name): Label` | Sets label name. |
| `getTorrents(): int` | Gets number of torrents in label. |
| `setTorrents(int $torrents): Label` | Sets number of torrents in label. |

### `File`

| Signature | Description |
| --- | --- |
| `getFilename(): string` | Gets file name/path in torrent. |
| `setFilename(string $filename): File` | Sets file name/path. |
| `getFilesize(): int` | Gets full file size in bytes. |
| `setFilesize(int $filesize): File` | Sets file size. |
| `getDownloaded(): int` | Gets bytes downloaded for file. |
| `setDownloaded(int $downloaded): File` | Sets downloaded bytes. |
| `getPriority(): int` | Gets uTorrent file priority value. |
| `setPriority(int $priority): File` | Sets file priority value. |

### `SettingItem`

| Signature | Description |
| --- | --- |
| `fromJson($json, array $filter = null)` | Hydrates setting `[name, type, value]` and normalizes value type. |
| `toOriginal()` | Converts setting back to original API-compatible array format. |
| `getName(): string` | Returns setting key name. |
| `getType(): int` | Returns setting type (`0=int`, `1=bool`, `2=string`). |
| `getValue()` | Returns normalized setting value. |

### `Torrent`

Constants:

- `STATUS_STARTED = 1`
- `STATUS_CHEKING = 2` (checking; constant name mirrors source code)
- `STATUS_START_AFTER_CHECK = 4`
- `STATUS_CHECKED = 8`
- `STATUS_ERROR = 16`
- `STATUS_PAUSED = 32`
- `STATUS_QUEUED = 64`
- `STATUS_LOADED = 128`

| Signature | Description |
| --- | --- |
| `refreshCache($newData, array $filter = null)` | Partially updates torrent fields; updates dynamic fields when status indicates active torrent. |
| `getHash(): string` / `setHash(string $hash): Torrent` | Torrent hash identifier. |
| `getStatus(): int` / `setStatus(int $status): Torrent` | Bitmask status value. |
| `getName(): string` / `setName(string $name): Torrent` | Display name. |
| `getSize(): int` / `setSize(int $size): Torrent` | Total size in bytes. |
| `getProgress(): int` / `setProgress(int $progress): Torrent` | Progress (uTorrent integer scale). |
| `getDownloaded(): int` / `setDownloaded(int $downloaded): Torrent` | Downloaded bytes. |
| `getUploaded(): int` / `setUploaded(int $uploaded): Torrent` | Uploaded bytes. |
| `getRatio(): int` / `setRatio(int $ratio): Torrent` | Ratio (uTorrent integer scale). |
| `getUploadSpeed(): int` / `setUploadSpeed(int $uploadSpeed): Torrent` | Upload speed. |
| `getDownloadSpeed(): int` / `setDownloadSpeed(int $downloadSpeed): Torrent` | Download speed. |
| `getEta(): int` / `setEta(int $eta): Torrent` | ETA in seconds. |
| `getLabel(): string` / `setLabel(string $label): Torrent` | Label name. |
| `getPeersConnected(): int` / `setPeersConnected(int $peersConnected): Torrent` | Connected peers count. |
| `getPeersInSwarm(): int` / `setPeersInSwarm(int $peersInSwarm): Torrent` | Peers in swarm count. |
| `getSeedsConnected(): int` / `setSeedsConnected(int $seedsConnected): Torrent` | Connected seeds count. |
| `getSeedsInSwarm(): int` / `setSeedsInSwarm(int $seedsInSwarm): Torrent` | Seeds in swarm count. |
| `getAvailability(): int` / `setAvailability(int $availability): Torrent` | Availability value. |
| `getQueueOrder(): int` / `setQueueOrder(int $queueOrder): Torrent` | Queue order index. |
| `getRemaining(): int` / `setRemaining(int $remaining): Torrent` | Remaining bytes. |
| `getAdded(): int` / `setAdded(int $added): Torrent` | Added timestamp (epoch). |
| `getCompleted(): int` / `setCompleted(int $completed): Torrent` | Completed timestamp (epoch). |
| `getSourceFile(): string` / `setSourceFile(string $sourceFile): Torrent` | Source `.torrent` file path/name. |

### `TorrentProp`

| Signature | Description |
| --- | --- |
| `getHash(): string` / `setHash(string $hash): TorrentProp` | Torrent hash. |
| `getTrackers(): string` / `setTrackers(string $trackers): TorrentProp` | Tracker list string. |
| `getUlrate(): int` / `setUlrate(int $ulrate): TorrentProp` | Max upload rate (`0` means default). |
| `getDlrate(): int` / `setDlrate(int $dlrate): TorrentProp` | Max download rate (`0` means default). |
| `getSuperseed(): int` / `setSuperseed(int $superseed): TorrentProp` | Initial seeding flag/value. |
| `getDht(): int` / `setDht(int $dht): TorrentProp` | DHT flag/value. |
| `getPex(): int` / `setPex(int $pex): TorrentProp` | Peer exchange flag/value. |
| `getSeedOverride(): int` / `setSeedOverride(int $seed_override): TorrentProp` | Seed override flag/value. |
| `getSeedRatio(): int` / `setSeedRatio(int $seed_ratio): TorrentProp` | Seed ratio value. |
| `getSeedTime(): int` / `setSeedTime(int $seed_time): TorrentProp` | Seed time value. |
| `getUlslots(): int` / `setUlslots(int $ulslots): TorrentProp` | Upload slots value. |

### `RssFeed`

| Signature | Description |
| --- | --- |
| `__construct()` | Initializes RSS items collection. |
| `fromJson($json, array $filter = null)` | Hydrates feed fields and nested `RssItem` entries. |
| `toOriginal()` | Serializes feed and nested items back to original array format. |
| `getId(): int` | Feed ID. |
| `getEnabled(): bool` | Feed enabled flag. |
| `getUseFeedTitle(): bool` | Use feed title flag. |
| `getUserSelected(): bool` | User-selected flag. |
| `getProgrammed(): bool` | Programmed flag. |
| `getDownloadState(): int` | Download state value. |
| `getUrl(): string` | Feed URL. |
| `getNextUpdate(): int` | Next update timestamp/value. |
| `getItems(): ArrayCollection` | Collection of `RssItem`. |
| `getMap(): array` | Returns internal mapping array used for hydration. |

### `RssItem`

| Signature | Description |
| --- | --- |
| `getName(): string` / `setName(string $name): RssItem` | Short item name. |
| `getNameFull(): string` / `setNameFull(string $nameFull): RssItem` | Full item name/title. |
| `getUrl(): string` / `setUrl(string $url): RssItem` | Item URL/magnet/torrent link. |
| `getQuality(): int` / `setQuality(int $quality): RssItem` | Quality code. |
| `getCodec(): int` / `setCodec(int $codec): RssItem` | Codec code. |
| `getTimestamp(): int` / `setTimestamp(int $timestamp): RssItem` | Timestamp value. |
| `getSeason(): int` / `setSeason(int $season): RssItem` | Season number. |
| `getEpisode(): int` / `setEpisode(int $episode): RssItem` | Episode number. |
| `getEpisodeTo(): int` / `setEpisodeTo(int $episodeTo): RssItem` | Episode range upper bound. |
| `getFeedId(): int` / `setFeedId(int $feedId): RssItem` | Parent feed ID. |
| `getRepack(): bool` / `setRepack(bool $repack): RssItem` | Repack flag. |
| `getInHistory(): bool` / `setInHistory(bool $inHistory): RssItem` | Download-history flag. |

### `RssFilter`

| Signature | Description |
| --- | --- |
| `getId(): int` / `setId(int $id): RssFilter` | Filter ID. |
| `getFlags(): int` / `setFlags(int $flags): RssFilter` | RSS filter flags bitmask/value. |
| `getName(): string` / `setName(string $name): RssFilter` | Filter name. |
| `getFilter(): string` / `setFilter(string $filter): RssFilter` | Positive match pattern. |
| `getNotFilter(): string` / `setNotFilter(string $notFilter): RssFilter` | Exclusion pattern. |
| `getDirectory(): string` / `setDirectory(string $directory): RssFilter` | Target download directory. |
| `getFeed(): int` / `setFeed(int $feed): RssFilter` | Feed selector/index. |
| `getQuality(): int` / `setQuality(int $quality): RssFilter` | Desired quality value. |
| `getLabel(): string` / `setLabel(string $label): RssFilter` | Label to apply on match. |
| `getPostponeMode(): int` / `setPostponeMode(int $postponeMode): RssFilter` | Postpone mode value. |
| `getLastMatch(): int` / `setLastMatch(int $lastMatch): RssFilter` | Last match timestamp/value. |
| `getSmartEpFilter(): int` / `setSmartEpFilter(int $smartEpFilter): RssFilter` | Smart episode filter setting. |
| `getRepackEpFilter(): int` / `setRepackEpFilter(int $repackEpFilter): RssFilter` | Repack episode filter setting. |
| `getEpisodeFilterStr(): string` / `setEpisodeFilterStr(string $episodeFilterStr): RssFilter` | Episode filter expression string. |
| `getEpisodeFilter(): bool` / `setEpisodeFilter(bool $episodeFilter): RssFilter` | Episode filter enabled flag. |
| `getResolvingCandidate(): bool` / `setResolvingCandidate(bool $resolvingCandidate): RssFilter` | Resolving-candidate flag. |

## Notes and Behavior

- `setCache(...)` should be called before methods that rely on caching (`getToken()`, `getTorrents()`).
- Token cache key is derived from connection URI via CRC32 hash.
- `getTorrent(string $hash)` internally calls `getTorrents()` and then filters in memory.
- `BaseResponse::fromHtml(...)` strips ASCII control characters (`0..31`) before JSON decoding.
- `addUrl(...)` URL-encodes input before sending request.

## Examples

See `/examples`:

- `/home/runner/work/php-utorrent/php-utorrent/examples/get-torrents.php`
- `/home/runner/work/php-utorrent/php-utorrent/examples/get-torrents-filtered.php`
- `/home/runner/work/php-utorrent/php-utorrent/examples/get-settings.php`
- `/home/runner/work/php-utorrent/php-utorrent/examples/get-props.php`
- `/home/runner/work/php-utorrent/php-utorrent/examples/hostconf.php.dist`

## License

This fork is licensed under the MIT License. See [`LICENSE`](LICENSE).
