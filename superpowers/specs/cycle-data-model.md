# Cycle Data Model — Spec

**Status:** Approved for implementation  
**Implements:** Phase 1 (foundation module) + Phase 2 (core-service)

---

## Overview

The `Cycle` is the central job entity of the platform. One Cycle represents one audiobook being processed from discovery to publication. All other entities (`AudioPart`, `ImagePart`, `VideoPart`, `TorrentTask`, `TelegramMediaGroup`) are children of a Cycle.

Before this model, state was scattered across JSON files and in-memory service variables. This model replaces all of that.

---

## Entity Relationships

```
Cycle (1)
├── AudioPart (many)      — one per audio segment/chapter set
├── ImagePart (many)      — covers at different processing stages
├── VideoPart (many)      — final MP4 outputs
├── TorrentTask (many)    — one per torrent download triggered
└── TelegramMediaGroup (many)
      └── TelegramMediaItem (many)  — individual messages in a group
```

---

## Enums (in `foundation/enums/`)

```java
public enum CycleStatus {
    PENDING,            // Created, not yet started
    SEARCHING,          // Scraping / search in progress
    DOWNLOADING,        // Torrent added, waiting for completion
    PROCESSING_AUDIO,   // Concatenate / trim / compress running
    PROCESSING_IMAGE,   // Cover art generation running
    PROCESSING_VIDEO,   // Video creation running
    POSTING,            // Uploading to Boosty / YouTube / Telegram
    COMPLETED,          // All steps done successfully
    FAILED              // A step failed — check cycle_logs for details
}

public enum ProcessingStatus {
    PENDING,
    IN_PROGRESS,
    COMPLETED,
    FAILED
}

public enum TorrentStatus {
    PENDING,
    DOWNLOADING,
    STALLED,
    COMPLETED,
    FAILED,
    DELETED_FROM_CLIENT
}
```

---

## JPA Entities

### Cycle

```java
@Entity
@Table(name = "cycles")
public class Cycle {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String cycleIdentifier;   // e.g. "Witcher_Part1_2024"

    private String title;
    private String author;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private CycleStatus status = CycleStatus.PENDING;

    @Lob
    private String description;       // Final aggregated description

    private String basePath;          // Filesystem root for this cycle's media files

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime completedAt;

    @OneToMany(mappedBy = "cycle", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<AudioPart> audioParts = new ArrayList<>();

    @OneToMany(mappedBy = "cycle", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<ImagePart> imageParts = new ArrayList<>();

    @OneToMany(mappedBy = "cycle", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<VideoPart> videoParts = new ArrayList<>();

    @OneToMany(mappedBy = "cycle", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<TorrentTask> torrentTasks = new ArrayList<>();

    @OneToMany(mappedBy = "cycle", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<TelegramMediaGroup> telegramGroups = new ArrayList<>();
}
```

### AudioPart

```java
@Entity
@Table(name = "audio_parts")
public class AudioPart {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cycle_id", nullable = false)
    private Cycle cycle;

    @Column(nullable = false)
    private Integer partNumber;

    private String title;
    private Long durationSeconds;

    private String rawChaptersPath;       // Downloaded files, pre-processing
    private String concatenatedPath;      // After joining chapters
    private String trimmedPath;           // After silence trimming
    private String compressedPath;        // Final — used for video and Telegram upload

    @Enumerated(EnumType.STRING)
    private ProcessingStatus status = ProcessingStatus.PENDING;

    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

### ImagePart

```java
@Entity
@Table(name = "image_parts")
public class ImagePart {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cycle_id", nullable = false)
    private Cycle cycle;

    private Integer partNumber;           // null if image applies to whole cycle

    @Column(nullable = false)
    private String imageType;             // "base_ai_cover", "final_cover", "thumbnail"

    @Column(nullable = false)
    private String imagePath;

    private String aiPromptUsed;          // DALL-E prompt that generated base image

    private LocalDateTime createdAt;
}
```

### TorrentTask

```java
@Entity
@Table(name = "torrent_tasks")
public class TorrentTask {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cycle_id", nullable = false)
    private Cycle cycle;

    @Column(unique = true)
    private String torrentHash;

    private String torrentName;
    private String downloadPath;
    private Long sizeBytes;

    @Enumerated(EnumType.STRING)
    private TorrentStatus status = TorrentStatus.PENDING;

    private LocalDateTime addedAt;
    private LocalDateTime completedAt;
}
```

### TelegramMediaGroup + TelegramMediaItem

```java
@Entity
@Table(name = "telegram_media_groups")
public class TelegramMediaGroup {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cycle_id", nullable = false)
    private Cycle cycle;

    @Column(nullable = false)
    private Long telegramChatId;

    private String groupLabel;            // e.g. "part1_announcement"

    private LocalDateTime sentAt;

    @OneToMany(mappedBy = "mediaGroup", cascade = CascadeType.ALL, orphanRemoval = true,
               fetch = FetchType.EAGER)
    private List<TelegramMediaItem> items = new ArrayList<>();
}

@Entity
@Table(name = "telegram_media_items")
public class TelegramMediaItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "media_group_id", nullable = false)
    private TelegramMediaGroup mediaGroup;

    @Column(nullable = false)
    private Integer telegramMessageId;

    private String telegramFileId;        // Reuse on re-send without re-uploading

    @Column(nullable = false)
    private String mediaType;             // "photo", "audio", "text"

    @Lob
    private String captionOrText;

    private Boolean isCoverPhoto = false;
}
```

---

## Repositories (in `foundation/repository/`)

```java
public interface CycleRepository extends JpaRepository<Cycle, Long> {
    Optional<Cycle> findByCycleIdentifier(String cycleIdentifier);
    List<Cycle> findByStatus(CycleStatus status);
    List<Cycle> findByStatusIn(List<CycleStatus> statuses);
}

public interface AudioPartRepository extends JpaRepository<AudioPart, Long> {
    List<AudioPart> findByCycle(Cycle cycle);
    Optional<AudioPart> findByCycleAndPartNumber(Cycle cycle, Integer partNumber);
}

public interface TorrentTaskRepository extends JpaRepository<TorrentTask, Long> {
    Optional<TorrentTask> findByTorrentHash(String hash);
    List<TorrentTask> findByCycleAndStatus(Cycle cycle, TorrentStatus status);
}

public interface TelegramMediaGroupRepository extends JpaRepository<TelegramMediaGroup, Long> {
    List<TelegramMediaGroup> findByCycle(Cycle cycle);
}
```

---

## File System Layout (per Cycle)

Each Cycle owns a directory tree under `basePath`. Paths stored in DB columns point into this tree.

```
<AUDIO_BASE_PATH>/
└── <cycleIdentifier>/
    ├── media/
    │   ├── audio/
    │   │   ├── chapters/         ← rawChaptersPath (downloaded from torrent)
    │   │   ├── concatenated/     ← concatenatedPath
    │   │   ├── trimmed/          ← trimmedPath
    │   │   └── compressed/       ← compressedPath (final)
    │   ├── video/
    │   │   └── part_1/           ← VideoPart.videoPath
    │   └── image/
    │       ├── base_cover.jpg    ← ImagePart (type=base_ai_cover)
    │       └── final_cover.jpg   ← ImagePart (type=final_cover)
    ├── torrents/
    │   └── *.torrent
    └── cycle_logs/
        └── *.log
```

---

## Flyway Migration (V1)

`core-service/src/main/resources/db/migration/V1__create_cycle_schema.sql`

```sql
CREATE TYPE cycle_status AS ENUM (
    'PENDING','SEARCHING','DOWNLOADING','PROCESSING_AUDIO',
    'PROCESSING_IMAGE','PROCESSING_VIDEO','POSTING','COMPLETED','FAILED'
);

CREATE TYPE processing_status AS ENUM ('PENDING','IN_PROGRESS','COMPLETED','FAILED');
CREATE TYPE torrent_status AS ENUM ('PENDING','DOWNLOADING','STALLED','COMPLETED','FAILED','DELETED_FROM_CLIENT');

CREATE TABLE cycles (
    id                BIGSERIAL PRIMARY KEY,
    cycle_identifier  VARCHAR(255) NOT NULL UNIQUE,
    title             VARCHAR(500),
    author            VARCHAR(255),
    status            cycle_status NOT NULL DEFAULT 'PENDING',
    description       TEXT,
    base_path         VARCHAR(1000),
    created_at        TIMESTAMP,
    updated_at        TIMESTAMP,
    completed_at      TIMESTAMP
);

CREATE TABLE audio_parts (
    id                  BIGSERIAL PRIMARY KEY,
    cycle_id            BIGINT NOT NULL REFERENCES cycles(id),
    part_number         INTEGER NOT NULL,
    title               VARCHAR(500),
    duration_seconds    BIGINT,
    raw_chapters_path   VARCHAR(1000),
    concatenated_path   VARCHAR(1000),
    trimmed_path        VARCHAR(1000),
    compressed_path     VARCHAR(1000),
    status              processing_status NOT NULL DEFAULT 'PENDING',
    created_at          TIMESTAMP,
    updated_at          TIMESTAMP
);

CREATE TABLE image_parts (
    id               BIGSERIAL PRIMARY KEY,
    cycle_id         BIGINT NOT NULL REFERENCES cycles(id),
    part_number      INTEGER,
    image_type       VARCHAR(100) NOT NULL,
    image_path       VARCHAR(1000) NOT NULL,
    ai_prompt_used   TEXT,
    created_at       TIMESTAMP
);

CREATE TABLE video_parts (
    id              BIGSERIAL PRIMARY KEY,
    cycle_id        BIGINT NOT NULL REFERENCES cycles(id),
    part_number     INTEGER,
    video_path      VARCHAR(1000),
    duration_seconds BIGINT,
    status          processing_status NOT NULL DEFAULT 'PENDING',
    created_at      TIMESTAMP,
    updated_at      TIMESTAMP
);

CREATE TABLE torrent_tasks (
    id             BIGSERIAL PRIMARY KEY,
    cycle_id       BIGINT NOT NULL REFERENCES cycles(id),
    torrent_hash   VARCHAR(255) UNIQUE,
    torrent_name   VARCHAR(500),
    download_path  VARCHAR(1000),
    size_bytes     BIGINT,
    status         torrent_status NOT NULL DEFAULT 'PENDING',
    added_at       TIMESTAMP,
    completed_at   TIMESTAMP
);

CREATE TABLE telegram_media_groups (
    id              BIGSERIAL PRIMARY KEY,
    cycle_id        BIGINT NOT NULL REFERENCES cycles(id),
    telegram_chat_id BIGINT NOT NULL,
    group_label     VARCHAR(255),
    sent_at         TIMESTAMP
);

CREATE TABLE telegram_media_items (
    id                  BIGSERIAL PRIMARY KEY,
    media_group_id      BIGINT NOT NULL REFERENCES telegram_media_groups(id),
    telegram_message_id INTEGER NOT NULL,
    telegram_file_id    VARCHAR(500),
    media_type          VARCHAR(50) NOT NULL,
    caption_or_text     TEXT,
    is_cover_photo      BOOLEAN DEFAULT FALSE
);
```

---

*This spec drives Phase 1 (`foundation` entities + repositories) and Phase 2 (`core-service` Flyway V1 migration).*
