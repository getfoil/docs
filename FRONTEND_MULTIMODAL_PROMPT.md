# Frontend Implementation: Multimodal Content Display on Trace Page

## Overview

Add multimodal content display to the trace information page. The system supports various file types that agents can receive as input or generate as output. Images and videos should be displayed inline, while other file types should be downloadable.

---

## Media Categories

| Category | File Types | Display Behavior |
|----------|-----------|------------------|
| `image` | PNG, JPEG, GIF, WebP, SVG, BMP, TIFF | **Inline display** - Show image directly |
| `video` | MP4, WebM, MOV, AVI | **Inline display** - Embedded video player |
| `audio` | MP3, WAV, OGG, FLAC, AAC | **Inline player** - Audio player with controls |
| `document` | PDF, DOCX, DOC, PPT, PPTX, RTF | **Download link** + Show extracted text preview |
| `spreadsheet` | CSV, XLSX, XLS | **Download link** + Show data preview (first few rows) |
| `code` | JS, PY, TS, JSON, etc. | **Download link** + Show code with syntax highlighting |
| `notebook` | IPYNB | **Download link** + Show extracted content |
| `archive` | ZIP, TAR, RAR, 7Z | **Download link only** |
| `other` | Unknown types | **Download link only** |

---

## API Endpoints

### 1. Get Trace with Media

Fetch a trace including all associated media files.

```
GET /api/traces/:traceId?includeMedia=true
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "traceId": "test_1234567890_abc123",
  "spanCount": 3,
  "spans": [
    {
      "eventId": "span_001",
      "traceId": "test_1234567890_abc123",
      "parentEventId": null,
      "spanKind": "llm",
      "event": "analyze-documents",
      "agentId": "agent_123",
      "userId": "user_456",
      "model": "gpt-4",
      "inputMessage": [
        { "type": "text", "text": "Please analyze these files:" },
        {
          "type": "media",
          "mediaId": "med_abc123",
          "category": "image",
          "filename": "screenshot.png"
        },
        {
          "type": "media",
          "mediaId": "med_def456",
          "category": "document",
          "filename": "report.pdf"
        }
      ],
      "output": [
        { "type": "text", "text": "I analyzed the files. Here are my findings:" },
        {
          "type": "media",
          "mediaId": "med_ghi789",
          "category": "spreadsheet",
          "filename": "analysis-results.csv"
        }
      ],
      "status": "ok",
      "createdAt": "2024-01-15T10:30:00.000Z",
      "updatedAt": "2024-01-15T10:30:05.000Z",
      "durationMs": 5000,
      "inputUnits": 150,
      "outputUnits": 200,
      "cost": 0.0035
    }
  ],
  "tree": { /* hierarchical span structure */ },
  "media": [
    {
      "mediaId": "med_abc123",
      "spanId": "span_001",
      "category": "image",
      "direction": "input",
      "filename": "screenshot.png",
      "mimeType": "image/png",
      "size": 245000,
      "processing": {
        "status": "completed",
        "completedAt": "2024-01-15T10:29:55.000Z"
      },
      "urls": {
        "original": "https://s3.amazonaws.com/...presigned-url...",
        "extracted": null,
        "preview": "https://s3.amazonaws.com/...preview-url..."
      },
      "extracted": null,
      "metadata": {
        "width": 1920,
        "height": 1080
      },
      "createdAt": "2024-01-15T10:29:50.000Z"
    },
    {
      "mediaId": "med_def456",
      "spanId": "span_001",
      "category": "document",
      "direction": "input",
      "filename": "report.pdf",
      "mimeType": "application/pdf",
      "size": 1500000,
      "processing": {
        "status": "completed",
        "completedAt": "2024-01-15T10:29:58.000Z"
      },
      "urls": {
        "original": "https://s3.amazonaws.com/...presigned-url...",
        "extracted": "https://s3.amazonaws.com/...extracted-text-url...",
        "preview": "https://s3.amazonaws.com/...preview-image-url..."
      },
      "extracted": {
        "length": 15000,
        "preview": "Executive Summary\n\nThis report covers the Q4 2024 performance metrics..."
      },
      "metadata": {
        "pageCount": 12,
        "title": "Q4 2024 Report",
        "author": "John Doe"
      },
      "createdAt": "2024-01-15T10:29:52.000Z"
    }
  ],
  "mediaBySpan": {
    "span_001": [
      { /* media object for med_abc123 */ },
      { /* media object for med_def456 */ },
      { /* media object for med_ghi789 */ }
    ]
  }
}
```

---

### 2. Get Media for a Span

Fetch all media attached to a specific span, grouped by direction.

```
GET /api/media/span/:spanId?includeUrls=true
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "spanId": "span_001",
  "total": 3,
  "input": [
    {
      "mediaId": "med_abc123",
      "category": "image",
      "direction": "input",
      "filename": "screenshot.png",
      "mimeType": "image/png",
      "size": 245000,
      "processing": {
        "status": "completed",
        "completedAt": "2024-01-15T10:29:55.000Z"
      },
      "urls": {
        "original": "https://s3.amazonaws.com/...presigned-url...",
        "extracted": null,
        "preview": null
      },
      "extracted": null,
      "metadata": { "width": 1920, "height": 1080 },
      "createdAt": "2024-01-15T10:29:50.000Z"
    }
  ],
  "output": [
    {
      "mediaId": "med_ghi789",
      "category": "spreadsheet",
      "direction": "output",
      "filename": "analysis-results.csv",
      "mimeType": "text/csv",
      "size": 5000,
      "processing": {
        "status": "completed"
      },
      "urls": {
        "original": "https://s3.amazonaws.com/...presigned-url...",
        "extracted": "https://s3.amazonaws.com/...structured-url..."
      },
      "extracted": {
        "length": 4500,
        "preview": "name,age,city\nJohn,30,NYC\nJane,25,LA..."
      },
      "metadata": {
        "rowCount": 100,
        "columnCount": 5,
        "columns": ["name", "age", "city", "occupation", "salary"]
      },
      "createdAt": "2024-01-15T10:30:05.000Z"
    }
  ],
  "media": [ /* all media items */ ]
}
```

---

### 3. Get Single Media Details

Fetch detailed information about a specific media file.

```
GET /api/media/:mediaId
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "mediaId": "med_abc123",
  "category": "image",
  "original": {
    "filename": "screenshot.png",
    "mimeType": "image/png",
    "size": 245000,
    "url": "https://s3.amazonaws.com/...presigned-url..."
  },
  "processing": {
    "status": "completed",
    "completedAt": "2024-01-15T10:29:55.000Z",
    "error": null
  },
  "extracted": {
    "text": null,
    "preview": {
      "width": 400,
      "height": 300,
      "url": "https://s3.amazonaws.com/...preview-url..."
    },
    "transcription": null,
    "structured": null
  },
  "metadata": {
    "width": 1920,
    "height": 1080,
    "format": "png"
  },
  "spanId": "span_001",
  "traceId": "test_1234567890_abc123",
  "direction": "input",
  "createdAt": "2024-01-15T10:29:50.000Z"
}
```

---

### 4. Get Original Content URL

Get a presigned URL to download/view the original file.

```
GET /api/media/:mediaId/content
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "url": "https://s3.amazonaws.com/...presigned-url...",
  "expiresAt": "2024-01-15T10:45:00.000Z",
  "contentType": "image/png",
  "filename": "screenshot.png",
  "size": 245000
}
```

---

### 5. Get Extracted Text URL

Get a presigned URL for extracted text content (for documents, code, etc.).

```
GET /api/media/:mediaId/extracted
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "url": "https://s3.amazonaws.com/...presigned-url...",
  "expiresAt": "2024-01-15T10:45:00.000Z",
  "contentType": "text/plain",
  "length": 15000,
  "truncated": false,
  "preview": "Executive Summary\n\nThis report covers..."
}
```

**Error (404) if no extracted text:**
```json
{
  "message": "Extracted text not available for this media"
}
```

---

### 6. Get Preview Image URL

Get a presigned URL for the preview thumbnail (for documents, videos, etc.).

```
GET /api/media/:mediaId/preview
Authorization: Bearer <jwt_token>
```

**Response:**
```json
{
  "url": "https://s3.amazonaws.com/...presigned-url...",
  "expiresAt": "2024-01-15T10:45:00.000Z",
  "contentType": "image/png",
  "width": 400,
  "height": 300
}
```

---

### 7. Batch Get Media Info

Fetch information for multiple media IDs at once (useful for rendering lists).

```
POST /api/media/batch
Authorization: Bearer <jwt_token>
Content-Type: application/json

{
  "mediaIds": ["med_abc123", "med_def456", "med_ghi789"]
}
```

**Response:**
```json
{
  "media": [
    {
      "mediaId": "med_abc123",
      "category": "image",
      "filename": "screenshot.png",
      "mimeType": "image/png",
      "size": 245000,
      "processing": { "status": "completed" },
      "hasExtractedText": false,
      "hasPreview": true,
      "metadata": { "width": 1920, "height": 1080 }
    },
    {
      "mediaId": "med_def456",
      "category": "document",
      "filename": "report.pdf",
      "mimeType": "application/pdf",
      "size": 1500000,
      "processing": { "status": "completed" },
      "hasExtractedText": true,
      "hasPreview": true,
      "metadata": { "pageCount": 12 }
    },
    null  // med_ghi789 not found or no access
  ]
}
```

---

## Content Block Structure

Input and output messages can contain mixed content blocks:

### Text Block
```json
{
  "type": "text",
  "text": "Please analyze this image..."
}
```

### Media Block
```json
{
  "type": "media",
  "mediaId": "med_abc123",
  "category": "image",
  "filename": "screenshot.png",
  "mimeType": "image/png",
  "size": 245000
}
```

---

## UI Component Requirements

### 1. Content Block Renderer

Create a component that renders mixed content blocks:

```tsx
interface ContentBlock {
  type: 'text' | 'media';
  text?: string;
  mediaId?: string;
  category?: string;
  filename?: string;
  mimeType?: string;
  size?: number;
}

interface ContentRendererProps {
  content: ContentBlock[] | string;
  mediaMap: Record<string, MediaItem>;
}
```

**Rendering Logic:**
- If `content` is a string, render as plain text
- If `content` is an array, iterate and render each block:
  - `type: 'text'` → Render text (with markdown support if needed)
  - `type: 'media'` → Look up media in `mediaMap` and render appropriate component

### 2. Media Display Components

#### ImageMedia
- Display image inline using `urls.original`
- Click to open in lightbox/modal for full-size view
- Show loading state while image loads
- Fallback to placeholder on error

```tsx
interface ImageMediaProps {
  media: MediaItem;
  maxHeight?: number;
  onClick?: () => void;
}
```

#### VideoMedia
- Embedded video player with controls
- Use `urls.original` as source
- Show poster image from `urls.preview` if available
- Support common formats: MP4, WebM

```tsx
interface VideoMediaProps {
  media: MediaItem;
  autoPlay?: boolean;
  muted?: boolean;
}
```

#### AudioMedia
- Audio player with play/pause, progress bar, volume
- Display filename and duration
- Use `urls.original` as source

```tsx
interface AudioMediaProps {
  media: MediaItem;
}
```

#### DocumentMedia
- Show preview thumbnail from `urls.preview`
- Display filename, page count, size
- "Download" button using `urls.original`
- "View Text" button to show extracted text (if available)
- Expandable text preview from `extracted.preview`

```tsx
interface DocumentMediaProps {
  media: MediaItem;
  showExtractedText?: boolean;
}
```

#### SpreadsheetMedia
- Show preview of first few rows from `extracted.preview`
- Display column headers from `metadata.columns`
- Show row/column counts
- "Download" button for original file

```tsx
interface SpreadsheetMediaProps {
  media: MediaItem;
  maxPreviewRows?: number;
}
```

#### CodeMedia
- Syntax-highlighted code preview from `extracted.preview`
- Language detection from filename extension
- "Download" button for original file
- "Copy" button for code content

```tsx
interface CodeMediaProps {
  media: MediaItem;
  maxLines?: number;
}
```

#### GenericMedia (for archive, other)
- File icon based on category/mimeType
- Display filename and size
- "Download" button only

```tsx
interface GenericMediaProps {
  media: MediaItem;
}
```

### 3. Processing Status Indicator

Show processing status for files still being processed:

```tsx
interface ProcessingStatusProps {
  status: 'pending' | 'processing' | 'completed' | 'failed';
  error?: string;
}
```

- `pending` → "Processing queued..."
- `processing` → Spinner + "Processing..."
- `completed` → Green checkmark (or just show content)
- `failed` → Red error icon + error message

---

## Trace Page Layout

### Span Detail View

```
┌─────────────────────────────────────────────────────────────┐
│ Span: analyze-documents                                     │
│ Type: llm | Model: gpt-4 | Duration: 5.0s | Cost: $0.0035  │
├─────────────────────────────────────────────────────────────┤
│ INPUT                                                       │
│ ─────                                                       │
│ Please analyze these files:                                 │
│                                                             │
│ ┌─────────────────┐  ┌─────────────────┐                   │
│ │  [Image]        │  │  📄 report.pdf  │                   │
│ │  screenshot.png │  │  12 pages, 1.5MB│                   │
│ │  1920x1080      │  │  [Download]     │                   │
│ └─────────────────┘  │  [View Text]    │                   │
│                      └─────────────────┘                   │
├─────────────────────────────────────────────────────────────┤
│ OUTPUT                                                      │
│ ──────                                                      │
│ I analyzed the files. Here are my findings:                 │
│                                                             │
│ ┌────────────────────────────────────────┐                 │
│ │  📊 analysis-results.csv               │                 │
│ │  100 rows × 5 columns                  │                 │
│ │  ┌──────┬─────┬───────┐               │                 │
│ │  │ name │ age │ city  │               │                 │
│ │  ├──────┼─────┼───────┤               │                 │
│ │  │ John │ 30  │ NYC   │               │                 │
│ │  │ Jane │ 25  │ LA    │               │                 │
│ │  └──────┴─────┴───────┘               │                 │
│ │  [Download CSV]                        │                 │
│ └────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Checklist

### Phase 1: Core Components
- [ ] Create `ContentBlockRenderer` component
- [ ] Create `MediaDisplay` component with category switching
- [ ] Create `ImageMedia` component with lightbox
- [ ] Create `GenericMedia` component (download-only fallback)
- [ ] Create `ProcessingStatus` indicator

### Phase 2: Specialized Media Components
- [ ] Create `VideoMedia` component with player
- [ ] Create `AudioMedia` component with player
- [ ] Create `DocumentMedia` component with preview/download
- [ ] Create `SpreadsheetMedia` component with table preview
- [ ] Create `CodeMedia` component with syntax highlighting

### Phase 3: Integration
- [ ] Update Span detail view to use `ContentBlockRenderer`
- [ ] Add media sidebar/panel to trace view
- [ ] Implement lazy loading for media content
- [ ] Add error boundaries for failed media loads

### Phase 4: Polish
- [ ] Add loading skeletons for media
- [ ] Implement image lightbox/gallery
- [ ] Add keyboard navigation for media
- [ ] Optimize for mobile/responsive layout

---

## Utility Functions

### Format File Size
```typescript
function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  if (bytes < 1024 * 1024 * 1024) return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
  return `${(bytes / (1024 * 1024 * 1024)).toFixed(1)} GB`;
}
```

### Get File Icon
```typescript
function getFileIcon(category: string, mimeType?: string): string {
  const icons: Record<string, string> = {
    image: '🖼️',
    video: '🎬',
    audio: '🎵',
    document: '📄',
    spreadsheet: '📊',
    code: '💻',
    notebook: '📓',
    archive: '📦',
    other: '📎',
  };
  return icons[category] || icons.other;
}
```

### Check if Media is Viewable Inline
```typescript
function isInlineViewable(category: string): boolean {
  return ['image', 'video', 'audio'].includes(category);
}
```

### Parse Content Blocks
```typescript
function parseContent(content: unknown): ContentBlock[] {
  if (typeof content === 'string') {
    return [{ type: 'text', text: content }];
  }
  if (Array.isArray(content)) {
    return content;
  }
  return [];
}
```

---

## Error Handling

### Presigned URL Expiration
- URLs expire after 15 minutes (900 seconds)
- Implement refresh mechanism when URLs expire
- Show "Refresh" button on expired media

### Processing Failures
- Show error message from `processing.error`
- Offer retry option if applicable
- Gracefully degrade (show download link even if preview failed)

### Network Errors
- Show placeholder on image load failure
- Retry logic for transient failures
- Offline indicator for network issues

---

## Notes

1. **Presigned URLs**: All media URLs are presigned S3 URLs that expire after 15 minutes. Cache them locally but be prepared to refresh.

2. **Content Parsing**: The `input` and `output` fields can be either a plain string OR an array of content blocks. Always check the type.

3. **Media Processing**: Some media types require backend processing (PDF text extraction, video thumbnails). Check `processing.status` before showing certain features.

4. **Authentication**: All API requests require the JWT token in the `Authorization: Bearer <token>` header.

5. **Large Files**: For very large images/videos, consider implementing progressive loading or showing thumbnails first.
