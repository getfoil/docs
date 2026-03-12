# Prompt Management & Versioning — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add prompt template storage, versioning, environment-based deployment, and SDK retrieval so users can manage prompts through Foil and track their performance against traces.

**Architecture:** New MongoDB models (Prompt, PromptVersion) in foil/server with CRUD API + a slug-based SDK lookup endpoint. SDKs get a `getPrompt()` method with client-side caching. ClickHouse spans table gets `prompt_id`/`prompt_version` columns (ingestion migration) so analytics can segment by prompt version. Plan-gated: trial gets 3 prompts, starter 0, pro unlimited.

**Tech Stack:** Express 5, Mongoose 9, Zod validation, Jest tests (foil/server), ClickHouse ALTER TABLE (foil-ingestion), node:test (foil-ingestion), axios (JS SDK), requests (Python SDK)

---

## Chunk 1: Data Models & Migrations

### Task 1: Prompt MongoDB Model

**Files:**
- Create: `foil/server/src/models/promptModel.js`
- Test: `foil/server/tests/__unit__/models/promptModel.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/models/promptModel.test.js
const mongoose = require('mongoose');

// In-memory connection for model tests
let Prompt;

beforeAll(() => {
  Prompt = require('../../../src/models/promptModel');
});

describe('Prompt Model', () => {
  it('should require user field', async () => {
    const prompt = new Prompt({ name: 'test', slug: 'test' });
    const err = prompt.validateSync();
    expect(err.errors.user).toBeDefined();
  });

  it('should require name field', async () => {
    const prompt = new Prompt({ user: new mongoose.Types.ObjectId(), slug: 'test' });
    const err = prompt.validateSync();
    expect(err.errors.name).toBeDefined();
  });

  it('should require slug field', async () => {
    const prompt = new Prompt({ user: new mongoose.Types.ObjectId(), name: 'test' });
    const err = prompt.validateSync();
    expect(err.errors.slug).toBeDefined();
  });

  it('should default tags to empty array', () => {
    const prompt = new Prompt({
      user: new mongoose.Types.ObjectId(),
      name: 'test',
      slug: 'test-slug',
    });
    expect(prompt.tags).toEqual([]);
  });

  it('should accept valid prompt data', () => {
    const prompt = new Prompt({
      user: new mongoose.Types.ObjectId(),
      name: 'My Prompt',
      slug: 'my-prompt',
      description: 'A test prompt',
      agentId: new mongoose.Types.ObjectId(),
      tags: ['production', 'chat'],
    });
    const err = prompt.validateSync();
    expect(err).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/models/promptModel.test.js --no-coverage`
Expected: FAIL — `Cannot find module '../../../src/models/promptModel'`

- [ ] **Step 3: Write the Prompt model**

```javascript
// foil/server/src/models/promptModel.js
const mongoose = require('mongoose');

const promptSchema = mongoose.Schema(
  {
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
      index: true,
    },
    agentId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Agent',
      index: true,
    },
    name: {
      type: String,
      required: [true, 'Name is required'],
      trim: true,
      maxlength: 200,
    },
    slug: {
      type: String,
      required: [true, 'Slug is required'],
      trim: true,
      lowercase: true,
      maxlength: 100,
      match: [/^[a-z0-9]+(?:-[a-z0-9]+)*$/, 'Slug must be lowercase alphanumeric with hyphens'],
    },
    description: {
      type: String,
      trim: true,
      maxlength: 1000,
    },
    tags: {
      type: [String],
      default: [],
    },
    activeVersions: {
      // Maps environment name to version number
      // e.g. { development: 3, staging: 2, production: 1 }
      type: Map,
      of: Number,
      default: new Map(),
    },
    latestVersion: {
      type: Number,
      default: 0,
    },
  },
  { timestamps: true }
);

// Unique slug per user
promptSchema.index({ user: 1, slug: 1 }, { unique: true });
promptSchema.index({ user: 1, createdAt: -1 });

const Prompt = mongoose.model('Prompt', promptSchema);
module.exports = Prompt;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/models/promptModel.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/models/promptModel.js foil/server/tests/__unit__/models/promptModel.test.js
git commit -m "feat: add Prompt model for prompt management"
```

---

### Task 2: PromptVersion MongoDB Model

**Files:**
- Create: `foil/server/src/models/promptVersionModel.js`
- Test: `foil/server/tests/__unit__/models/promptVersionModel.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/models/promptVersionModel.test.js
const mongoose = require('mongoose');

let PromptVersion;

beforeAll(() => {
  PromptVersion = require('../../../src/models/promptVersionModel');
});

describe('PromptVersion Model', () => {
  it('should require prompt field', async () => {
    const v = new PromptVersion({ version: 1, content: 'Hello {{name}}' });
    const err = v.validateSync();
    expect(err.errors.prompt).toBeDefined();
  });

  it('should require content field', async () => {
    const v = new PromptVersion({
      prompt: new mongoose.Types.ObjectId(),
      version: 1,
    });
    const err = v.validateSync();
    expect(err.errors.content).toBeDefined();
  });

  it('should default model to gpt-4o-mini', () => {
    const v = new PromptVersion({
      prompt: new mongoose.Types.ObjectId(),
      version: 1,
      content: 'Hello {{name}}',
    });
    expect(v.model).toBe('gpt-4o-mini');
  });

  it('should extract variables from content', () => {
    const v = new PromptVersion({
      prompt: new mongoose.Types.ObjectId(),
      version: 1,
      content: 'Hello {{name}}, welcome to {{company}}. Your role is {{role}}.',
    });
    expect(v.variables).toEqual(['name', 'company', 'role']);
  });

  it('should handle content with no variables', () => {
    const v = new PromptVersion({
      prompt: new mongoose.Types.ObjectId(),
      version: 1,
      content: 'Hello world, no variables here.',
    });
    expect(v.variables).toEqual([]);
  });

  it('should accept valid modelParameters', () => {
    const v = new PromptVersion({
      prompt: new mongoose.Types.ObjectId(),
      version: 1,
      content: 'Test',
      modelParameters: { temperature: 0.7, maxTokens: 1000, topP: 0.9 },
    });
    const err = v.validateSync();
    expect(err).toBeUndefined();
    expect(v.modelParameters.temperature).toBe(0.7);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/models/promptVersionModel.test.js --no-coverage`
Expected: FAIL — `Cannot find module`

- [ ] **Step 3: Write the PromptVersion model**

```javascript
// foil/server/src/models/promptVersionModel.js
const mongoose = require('mongoose');

const modelParametersSchema = mongoose.Schema(
  {
    temperature: { type: Number, min: 0, max: 2 },
    maxTokens: { type: Number, min: 1 },
    topP: { type: Number, min: 0, max: 1 },
    frequencyPenalty: { type: Number, min: -2, max: 2 },
    presencePenalty: { type: Number, min: -2, max: 2 },
    stopSequences: { type: [String], default: [] },
  },
  { _id: false }
);

const promptVersionSchema = mongoose.Schema(
  {
    prompt: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Prompt',
      required: true,
      index: true,
    },
    version: {
      type: Number,
      required: true,
      min: 1,
    },
    content: {
      type: String,
      required: [true, 'Content is required'],
      maxlength: 50000,
    },
    // Variables auto-extracted from content via pre-validate hook
    variables: {
      type: [String],
      default: [],
    },
    model: {
      type: String,
      default: 'gpt-4o-mini',
    },
    modelParameters: {
      type: modelParametersSchema,
      default: () => ({}),
    },
    commitMessage: {
      type: String,
      trim: true,
      maxlength: 500,
    },
    createdBy: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
    },
  },
  { timestamps: true }
);

// Extract {{variable}} placeholders from content
promptVersionSchema.pre('validate', function () {
  if (this.content) {
    const matches = this.content.match(/\{\{(\w+)\}\}/g) || [];
    this.variables = [...new Set(matches.map((m) => m.slice(2, -2)))];
  }
});

promptVersionSchema.index({ prompt: 1, version: -1 }, { unique: true });

const PromptVersion = mongoose.model('PromptVersion', promptVersionSchema);
module.exports = PromptVersion;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/models/promptVersionModel.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/models/promptVersionModel.js foil/server/tests/__unit__/models/promptVersionModel.test.js
git commit -m "feat: add PromptVersion model with variable extraction"
```

---

### Task 3: ClickHouse spans table migration (foil-ingestion)

**Files:**
- Modify: `foil-ingestion/src/plugins/clickhouse.js` (add migration array + update insertSpan)
- Test: `foil-ingestion/test/unit/clickhousePromptMigration.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil-ingestion/test/unit/clickhousePromptMigration.test.js
import { describe, it } from 'node:test';
import assert from 'node:assert';
import { SPANS_PROMPT_MIGRATIONS } from '../../src/plugins/clickhouse.js';

describe('ClickHouse prompt migrations', () => {
  it('should export SPANS_PROMPT_MIGRATIONS array', () => {
    assert.ok(Array.isArray(SPANS_PROMPT_MIGRATIONS));
  });

  it('should have 2 migrations (prompt_id and prompt_version)', () => {
    assert.strictEqual(SPANS_PROMPT_MIGRATIONS.length, 2);
  });

  it('should add prompt_id as Nullable(String)', () => {
    const migration = SPANS_PROMPT_MIGRATIONS[0];
    assert.ok(migration.includes('prompt_id'));
    assert.ok(migration.includes('Nullable(String)'));
    assert.ok(migration.includes('ADD COLUMN IF NOT EXISTS'));
  });

  it('should add prompt_version as Nullable(UInt32)', () => {
    const migration = SPANS_PROMPT_MIGRATIONS[1];
    assert.ok(migration.includes('prompt_version'));
    assert.ok(migration.includes('Nullable(UInt32)'));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil-ingestion && node --test test/unit/clickhousePromptMigration.test.js`
Expected: FAIL — export not found

- [ ] **Step 3: Add migration array to clickhouse.js**

Add after the `SPANS_LLM_CONFIG_MIGRATIONS` array (around line 572):

```javascript
export const SPANS_PROMPT_MIGRATIONS = [
  `ALTER TABLE spans ADD COLUMN IF NOT EXISTS prompt_id Nullable(String)`,
  `ALTER TABLE spans ADD COLUMN IF NOT EXISTS prompt_version Nullable(UInt32)`,
];
```

Then add `...SPANS_PROMPT_MIGRATIONS` to the `allMigrations` array (around line 697):

```javascript
const allMigrations = [
  ...SPAN_EVALUATIONS_MIGRATIONS,
  ...SPANS_MEDIA_MIGRATIONS,
  ...SPANS_USER_DEVICE_MIGRATIONS,
  ...SPANS_LLM_CONFIG_MIGRATIONS,
  ...SPANS_PROMPT_MIGRATIONS,
];
```

Also update `insertSpan()` and `insertSpans()` to include `prompt_id` and `prompt_version` in the inserted data objects.

In `insertSpan(data)` (around line 1300), add to the values object:
```javascript
prompt_id: data.promptId || null,
prompt_version: data.promptVersion ? parseInt(data.promptVersion, 10) : null,
```

In `insertSpans(records)` (around line 1370), add to the map function:
```javascript
prompt_id: record.promptId || null,
prompt_version: record.promptVersion ? parseInt(record.promptVersion, 10) : null,
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil-ingestion && node --test test/unit/clickhousePromptMigration.test.js`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil-ingestion/src/plugins/clickhouse.js foil-ingestion/test/unit/clickhousePromptMigration.test.js
git commit -m "feat: add prompt_id and prompt_version columns to ClickHouse spans table"
```

---

### Task 4: Update SQS message format to carry prompt fields (foil/server)

**Files:**
- Modify: `foil/server/src/utils/sqsUtils.js` (add promptId, promptVersion to span message)
- Test: `foil/server/tests/__unit__/utils/sqsUtils.test.js` (add test for new fields)

- [ ] **Step 1: Write the failing test**

Add to existing sqsUtils test file (or create if not present):

```javascript
// foil/server/tests/__unit__/utils/sqsUtils.test.js
jest.mock('@aws-sdk/client-sqs', () => ({
  SQSClient: jest.fn(() => ({ send: jest.fn() })),
  SendMessageCommand: jest.fn(),
}));

const { buildSpanMessage } = require('../../../src/utils/sqsUtils');

describe('buildSpanMessage prompt fields', () => {
  it('should include promptId and promptVersion when provided', () => {
    const span = {
      traceId: 'trace-1',
      spanId: 'span-1',
      spanType: 'llm',
      name: 'test',
      userId: 'user-1',
      agentId: 'agent-1',
      promptId: 'prompt-abc',
      promptVersion: 3,
    };
    const msg = buildSpanMessage(span);
    expect(msg.promptId).toBe('prompt-abc');
    expect(msg.promptVersion).toBe(3);
  });

  it('should default promptId and promptVersion to null', () => {
    const span = {
      traceId: 'trace-1',
      spanId: 'span-1',
      spanType: 'llm',
      name: 'test',
      userId: 'user-1',
      agentId: 'agent-1',
    };
    const msg = buildSpanMessage(span);
    expect(msg.promptId).toBeNull();
    expect(msg.promptVersion).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/utils/sqsUtils.test.js --no-coverage`
Expected: FAIL — properties missing from message

- [ ] **Step 3: Add prompt fields to buildSpanMessage**

In `foil/server/src/utils/sqsUtils.js`, inside the `buildSpanMessage` function, add:

```javascript
promptId: span.promptId || null,
promptVersion: span.promptVersion || null,
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/utils/sqsUtils.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/utils/sqsUtils.js foil/server/tests/__unit__/utils/sqsUtils.test.js
git commit -m "feat: add promptId and promptVersion to SQS span message"
```

---

## Chunk 2: Prompt CRUD Controller & Routes

### Task 5: Prompt Controller — Create & List

**Files:**
- Create: `foil/server/src/controllers/promptController.js`
- Test: `foil/server/tests/__unit__/controllers/promptController.test.js`

- [ ] **Step 1: Write failing tests for createPrompt and getPrompts**

```javascript
// foil/server/tests/__unit__/controllers/promptController.test.js
jest.mock('../../../src/models/promptModel');
jest.mock('../../../src/models/promptVersionModel');
jest.mock('../../../src/utils/logger', () => ({
  info: jest.fn(),
  warn: jest.fn(),
  error: jest.fn(),
  debug: jest.fn(),
}));

const Prompt = require('../../../src/models/promptModel');
const PromptVersion = require('../../../src/models/promptVersionModel');

const { createPrompt, getPrompts } = require('../../../src/controllers/promptController');

function makeReqRes(body = {}, params = {}, query = {}) {
  return {
    req: {
      user: { _id: 'user-1' },
      body,
      params,
      query,
    },
    res: {
      status: jest.fn().mockReturnThis(),
      json: jest.fn(),
    },
  };
}

describe('promptController', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('createPrompt', () => {
    it('should create prompt and initial version', async () => {
      const savedPrompt = {
        _id: 'prompt-1',
        user: 'user-1',
        name: 'My Prompt',
        slug: 'my-prompt',
        latestVersion: 1,
        save: jest.fn(),
      };
      const savedVersion = {
        _id: 'version-1',
        prompt: 'prompt-1',
        version: 1,
        content: 'Hello {{name}}',
        variables: ['name'],
      };

      Prompt.findOne = jest.fn().mockResolvedValue(null);
      Prompt.mockImplementation((data) => ({
        ...data,
        _id: 'prompt-1',
        latestVersion: 0,
        save: jest.fn(async function () {
          this.latestVersion = 1;
          return this;
        }),
      }));
      PromptVersion.mockImplementation((data) => ({
        ...data,
        _id: 'version-1',
        variables: ['name'],
        save: jest.fn(async () => savedVersion),
      }));

      const { req, res } = makeReqRes({
        name: 'My Prompt',
        slug: 'my-prompt',
        content: 'Hello {{name}}',
      });

      await createPrompt(req, res);

      expect(res.status).toHaveBeenCalledWith(201);
      expect(res.json).toHaveBeenCalled();
    });

    it('should reject duplicate slug for same user', async () => {
      Prompt.findOne = jest.fn().mockResolvedValue({ _id: 'existing' });

      const { req, res } = makeReqRes({
        name: 'My Prompt',
        slug: 'existing-slug',
        content: 'Hello',
      });

      await expect(createPrompt(req, res)).rejects.toThrow('already exists');
      expect(res.status).toHaveBeenCalledWith(400);
    });
  });

  describe('getPrompts', () => {
    it('should return paginated prompts', async () => {
      const mockPrompts = [
        { _id: 'p1', name: 'Prompt 1', slug: 'prompt-1' },
        { _id: 'p2', name: 'Prompt 2', slug: 'prompt-2' },
      ];

      Prompt.find = jest.fn().mockReturnValue({
        limit: jest.fn().mockReturnThis(),
        skip: jest.fn().mockReturnThis(),
        sort: jest.fn().mockReturnThis(),
        lean: jest.fn().mockResolvedValue(mockPrompts),
      });
      Prompt.countDocuments = jest.fn().mockResolvedValue(2);

      const { req, res } = makeReqRes({}, {}, { page: '1', limit: '20' });

      await getPrompts(req, res);

      expect(res.json).toHaveBeenCalledWith(
        expect.objectContaining({
          prompts: mockPrompts,
          total: 2,
        })
      );
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/controllers/promptController.test.js --no-coverage`
Expected: FAIL — module not found

- [ ] **Step 3: Write the controller**

```javascript
// foil/server/src/controllers/promptController.js
const asyncHandler = require('express-async-handler');
const Prompt = require('../models/promptModel');
const PromptVersion = require('../models/promptVersionModel');
const logger = require('../utils/logger');

const createPrompt = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { name, slug, description, agentId, tags, content, model, modelParameters, commitMessage } = req.body;

  // Check slug uniqueness for this user
  const existing = await Prompt.findOne({ user: userId, slug });
  if (existing) {
    res.status(400);
    throw new Error(`A prompt with slug "${slug}" already exists`);
  }

  // Create prompt document
  const prompt = new Prompt({
    user: userId,
    name,
    slug,
    description,
    agentId,
    tags,
    latestVersion: 1,
  });

  // Create initial version
  const version = new PromptVersion({
    prompt: prompt._id,
    version: 1,
    content,
    model,
    modelParameters,
    commitMessage: commitMessage || 'Initial version',
    createdBy: userId,
  });

  await prompt.save();
  await version.save();

  logger.info({ promptId: prompt._id, slug }, 'Prompt created');

  res.status(201).json({
    prompt,
    version,
  });
});

const getPrompts = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { page = 1, limit = 20, agentId } = req.query;

  const query = { user: userId };
  if (agentId) query.agentId = agentId;

  const pageNum = parseInt(page, 10);
  const limitNum = parseInt(limit, 10);

  const [prompts, total] = await Promise.all([
    Prompt.find(query)
      .limit(limitNum)
      .skip((pageNum - 1) * limitNum)
      .sort({ updatedAt: -1 })
      .lean(),
    Prompt.countDocuments(query),
  ]);

  res.json({
    prompts,
    total,
    page: pageNum,
    pages: Math.ceil(total / limitNum),
  });
});

const getPrompt = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId } = req.params;

  const prompt = await Prompt.findOne({ _id: promptId, user: userId }).lean();
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  // Get latest version
  const latestVersion = await PromptVersion.findOne({
    prompt: promptId,
    version: prompt.latestVersion,
  }).lean();

  res.json({ ...prompt, currentVersion: latestVersion });
});

const updatePrompt = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId } = req.params;
  const { name, description, agentId, tags } = req.body;

  const prompt = await Prompt.findOne({ _id: promptId, user: userId });
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  if (name !== undefined) prompt.name = name;
  if (description !== undefined) prompt.description = description;
  if (agentId !== undefined) prompt.agentId = agentId;
  if (tags !== undefined) prompt.tags = tags;

  await prompt.save();

  logger.info({ promptId }, 'Prompt updated');
  res.json(prompt);
});

const deletePrompt = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId } = req.params;

  const prompt = await Prompt.findOne({ _id: promptId, user: userId });
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  // Delete all versions
  await PromptVersion.deleteMany({ prompt: promptId });
  await Prompt.deleteOne({ _id: promptId });

  logger.info({ promptId }, 'Prompt deleted');
  res.json({ message: 'Prompt deleted' });
});

// --- Version management ---

const createVersion = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId } = req.params;
  const { content, model, modelParameters, commitMessage } = req.body;

  const prompt = await Prompt.findOne({ _id: promptId, user: userId });
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  const newVersionNum = prompt.latestVersion + 1;

  const version = new PromptVersion({
    prompt: promptId,
    version: newVersionNum,
    content,
    model,
    modelParameters,
    commitMessage,
    createdBy: userId,
  });

  prompt.latestVersion = newVersionNum;

  await version.save();
  await prompt.save();

  logger.info({ promptId, version: newVersionNum }, 'Prompt version created');
  res.status(201).json(version);
});

const getVersions = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId } = req.params;

  const prompt = await Prompt.findOne({ _id: promptId, user: userId });
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  const versions = await PromptVersion.find({ prompt: promptId })
    .sort({ version: -1 })
    .lean();

  res.json(versions);
});

const getVersion = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId, version } = req.params;

  const prompt = await Prompt.findOne({ _id: promptId, user: userId });
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  const versionDoc = await PromptVersion.findOne({
    prompt: promptId,
    version: parseInt(version, 10),
  }).lean();

  if (!versionDoc) {
    res.status(404);
    throw new Error('Version not found');
  }

  res.json(versionDoc);
});

// --- Deployment ---

const deployVersion = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { promptId } = req.params;
  const { version, environment } = req.body;

  const validEnvs = ['development', 'staging', 'production'];
  if (!validEnvs.includes(environment)) {
    res.status(400);
    throw new Error(`Environment must be one of: ${validEnvs.join(', ')}`);
  }

  const prompt = await Prompt.findOne({ _id: promptId, user: userId });
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  const versionDoc = await PromptVersion.findOne({
    prompt: promptId,
    version: parseInt(version, 10),
  });
  if (!versionDoc) {
    res.status(404);
    throw new Error('Version not found');
  }

  prompt.activeVersions.set(environment, parseInt(version, 10));
  await prompt.save();

  logger.info({ promptId, version, environment }, 'Prompt version deployed');
  res.json({
    promptId,
    environment,
    version: parseInt(version, 10),
    deployedAt: new Date().toISOString(),
  });
});

// --- SDK lookup (by slug, API key auth) ---

const getPromptBySlug = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { slug } = req.params;
  const { environment = 'production', version: requestedVersion } = req.query;

  const prompt = await Prompt.findOne({ user: userId, slug }).lean();
  if (!prompt) {
    res.status(404);
    throw new Error('Prompt not found');
  }

  // Determine which version to return
  let versionNum;
  if (requestedVersion) {
    versionNum = parseInt(requestedVersion, 10);
  } else if (prompt.activeVersions && prompt.activeVersions[environment]) {
    versionNum = prompt.activeVersions[environment];
  } else {
    versionNum = prompt.latestVersion;
  }

  const versionDoc = await PromptVersion.findOne({
    prompt: prompt._id,
    version: versionNum,
  }).lean();

  if (!versionDoc) {
    res.status(404);
    throw new Error('Prompt version not found');
  }

  res.json({
    id: prompt._id,
    name: prompt.name,
    slug: prompt.slug,
    version: versionDoc.version,
    content: versionDoc.content,
    variables: versionDoc.variables,
    model: versionDoc.model,
    modelParameters: versionDoc.modelParameters,
  });
});

module.exports = {
  createPrompt,
  getPrompts,
  getPrompt,
  updatePrompt,
  deletePrompt,
  createVersion,
  getVersions,
  getVersion,
  deployVersion,
  getPromptBySlug,
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/controllers/promptController.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/controllers/promptController.js foil/server/tests/__unit__/controllers/promptController.test.js
git commit -m "feat: add prompt controller with CRUD, versioning, deployment, and slug lookup"
```

---

### Task 6: Prompt Controller — Additional Tests (get, update, delete, versions, deploy, slug)

**Files:**
- Modify: `foil/server/tests/__unit__/controllers/promptController.test.js`

- [ ] **Step 1: Add tests for remaining controller methods**

Append to the existing test file:

```javascript
describe('getPrompt', () => {
  it('should return prompt with current version', async () => {
    const mockPrompt = { _id: 'p1', name: 'Test', latestVersion: 2 };
    const mockVersion = { version: 2, content: 'Hello {{name}}' };

    Prompt.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(mockPrompt) });
    PromptVersion.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(mockVersion) });

    const { req, res } = makeReqRes({}, { promptId: 'p1' });
    await getPrompt(req, res);

    expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ name: 'Test', currentVersion: mockVersion }));
  });

  it('should return 404 for non-existent prompt', async () => {
    Prompt.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(null) });

    const { req, res } = makeReqRes({}, { promptId: 'nonexistent' });
    await expect(getPrompt(req, res)).rejects.toThrow('Prompt not found');
    expect(res.status).toHaveBeenCalledWith(404);
  });
});

describe('deletePrompt', () => {
  it('should delete prompt and all versions', async () => {
    Prompt.findOne = jest.fn().mockResolvedValue({ _id: 'p1' });
    PromptVersion.deleteMany = jest.fn().mockResolvedValue({});
    Prompt.deleteOne = jest.fn().mockResolvedValue({});

    const { req, res } = makeReqRes({}, { promptId: 'p1' });
    await deletePrompt(req, res);

    expect(PromptVersion.deleteMany).toHaveBeenCalledWith({ prompt: 'p1' });
    expect(Prompt.deleteOne).toHaveBeenCalledWith({ _id: 'p1' });
    expect(res.json).toHaveBeenCalledWith({ message: 'Prompt deleted' });
  });
});

describe('createVersion', () => {
  it('should create new version and bump latestVersion', async () => {
    const mockPrompt = { _id: 'p1', latestVersion: 2, save: jest.fn() };
    Prompt.findOne = jest.fn().mockResolvedValue(mockPrompt);
    PromptVersion.mockImplementation((data) => ({
      ...data,
      save: jest.fn(),
    }));

    const { req, res } = makeReqRes(
      { content: 'Updated {{name}}', commitMessage: 'fix typo' },
      { promptId: 'p1' }
    );
    await createVersion(req, res);

    expect(mockPrompt.latestVersion).toBe(3);
    expect(res.status).toHaveBeenCalledWith(201);
  });
});

describe('deployVersion', () => {
  it('should set activeVersion for environment', async () => {
    const activeVersions = new Map();
    const mockPrompt = {
      _id: 'p1',
      activeVersions,
      save: jest.fn(),
    };
    Prompt.findOne = jest.fn().mockResolvedValue(mockPrompt);
    PromptVersion.findOne = jest.fn().mockResolvedValue({ version: 2 });

    const { req, res } = makeReqRes(
      { version: 2, environment: 'production' },
      { promptId: 'p1' }
    );
    await deployVersion(req, res);

    expect(activeVersions.get('production')).toBe(2);
    expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ environment: 'production', version: 2 }));
  });

  it('should reject invalid environment', async () => {
    const { req, res } = makeReqRes(
      { version: 1, environment: 'invalid' },
      { promptId: 'p1' }
    );
    await expect(deployVersion(req, res)).rejects.toThrow('Environment must be one of');
    expect(res.status).toHaveBeenCalledWith(400);
  });
});

describe('getPromptBySlug', () => {
  it('should return deployed version for environment', async () => {
    const mockPrompt = {
      _id: 'p1',
      name: 'Test',
      slug: 'test',
      activeVersions: { production: 2 },
      latestVersion: 3,
    };
    const mockVersion = {
      version: 2,
      content: 'Hello {{name}}',
      variables: ['name'],
      model: 'gpt-4o',
      modelParameters: {},
    };

    Prompt.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(mockPrompt) });
    PromptVersion.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(mockVersion) });

    const { req, res } = makeReqRes({}, { slug: 'test' }, { environment: 'production' });
    await getPromptBySlug(req, res);

    const response = res.json.mock.calls[0][0];
    expect(response.version).toBe(2);
    expect(response.content).toBe('Hello {{name}}');
    expect(response.variables).toEqual(['name']);
  });

  it('should fall back to latest version when no deployment exists', async () => {
    const mockPrompt = {
      _id: 'p1',
      name: 'Test',
      slug: 'test',
      activeVersions: {},
      latestVersion: 3,
    };
    const mockVersion = { version: 3, content: 'Latest', variables: [], model: 'gpt-4o-mini', modelParameters: {} };

    Prompt.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(mockPrompt) });
    PromptVersion.findOne = jest.fn().mockReturnValue({ lean: jest.fn().mockResolvedValue(mockVersion) });

    const { req, res } = makeReqRes({}, { slug: 'test' }, {});
    await getPromptBySlug(req, res);

    expect(PromptVersion.findOne).toHaveBeenCalledWith(expect.objectContaining({ version: 3 }));
  });
});
```

- [ ] **Step 2: Run all controller tests**

Run: `cd foil/server && npx jest tests/__unit__/controllers/promptController.test.js --no-coverage`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add foil/server/tests/__unit__/controllers/promptController.test.js
git commit -m "test: add comprehensive prompt controller tests"
```

---

### Task 7: Validation Schemas

**Files:**
- Modify: `foil/server/src/validations/schemas.js`

- [ ] **Step 1: Add prompt validation schemas**

Append to `foil/server/src/validations/schemas.js`:

```javascript
// Prompt schemas
const createPromptSchema = z.object({
  body: z.object({
    name: z.string().min(1, 'Name is required').max(200),
    slug: z.string().min(1, 'Slug is required').max(100).regex(/^[a-z0-9]+(?:-[a-z0-9]+)*$/, 'Slug must be lowercase alphanumeric with hyphens'),
    description: z.string().max(1000).optional(),
    agentId: z.string().optional(),
    tags: z.array(z.string()).optional(),
    content: z.string().min(1, 'Content is required').max(50000),
    model: z.string().optional(),
    modelParameters: z.object({
      temperature: z.number().min(0).max(2).optional(),
      maxTokens: z.number().min(1).optional(),
      topP: z.number().min(0).max(1).optional(),
      frequencyPenalty: z.number().min(-2).max(2).optional(),
      presencePenalty: z.number().min(-2).max(2).optional(),
      stopSequences: z.array(z.string()).optional(),
    }).optional(),
    commitMessage: z.string().max(500).optional(),
  }),
  query: z.object({}).passthrough().optional(),
  params: z.object({}).passthrough().optional(),
});

const updatePromptSchema = z.object({
  body: z.object({
    name: z.string().min(1).max(200).optional(),
    description: z.string().max(1000).optional(),
    agentId: z.string().nullable().optional(),
    tags: z.array(z.string()).optional(),
  }),
  params: z.object({
    promptId: z.string().min(1),
  }),
  query: z.object({}).passthrough().optional(),
});

const promptIdParamSchema = z.object({
  params: z.object({
    promptId: z.string().min(1),
  }),
  body: z.object({}).passthrough().optional(),
  query: z.object({}).passthrough().optional(),
});

const createVersionSchema = z.object({
  body: z.object({
    content: z.string().min(1, 'Content is required').max(50000),
    model: z.string().optional(),
    modelParameters: z.object({
      temperature: z.number().min(0).max(2).optional(),
      maxTokens: z.number().min(1).optional(),
      topP: z.number().min(0).max(1).optional(),
      frequencyPenalty: z.number().min(-2).max(2).optional(),
      presencePenalty: z.number().min(-2).max(2).optional(),
      stopSequences: z.array(z.string()).optional(),
    }).optional(),
    commitMessage: z.string().max(500).optional(),
  }),
  params: z.object({
    promptId: z.string().min(1),
  }),
  query: z.object({}).passthrough().optional(),
});

const deployVersionSchema = z.object({
  body: z.object({
    version: z.number().int().min(1),
    environment: z.enum(['development', 'staging', 'production']),
  }),
  params: z.object({
    promptId: z.string().min(1),
  }),
  query: z.object({}).passthrough().optional(),
});

const getPromptBySlugSchema = z.object({
  params: z.object({
    slug: z.string().min(1),
  }),
  query: z.object({
    environment: z.enum(['development', 'staging', 'production']).optional(),
    version: z.string().optional(),
  }).optional(),
  body: z.object({}).passthrough().optional(),
});
```

Then add all new schemas to the `module.exports` at the bottom of the file.

- [ ] **Step 2: Verify schemas parse correctly**

Run: `cd foil/server && node -e "const s = require('./src/validations/schemas'); console.log(Object.keys(s).filter(k => k.includes('rompt')))"`
Expected: lists all prompt schema names

- [ ] **Step 3: Commit**

```bash
git add foil/server/src/validations/schemas.js
git commit -m "feat: add Zod validation schemas for prompt endpoints"
```

---

### Task 8: Prompt Routes & Server Registration

**Files:**
- Create: `foil/server/src/routes/promptRoutes.js`
- Modify: `foil/server/src/server.js`

- [ ] **Step 1: Create routes file**

```javascript
// foil/server/src/routes/promptRoutes.js
const express = require('express');
const router = express.Router();
const {
  createPrompt,
  getPrompts,
  getPrompt,
  updatePrompt,
  deletePrompt,
  createVersion,
  getVersions,
  getVersion,
  deployVersion,
  getPromptBySlug,
} = require('../controllers/promptController');
const { validate } = require('../middleware/validateMiddleware');
const {
  createPromptSchema,
  updatePromptSchema,
  promptIdParamSchema,
  createVersionSchema,
  deployVersionSchema,
  getPromptBySlugSchema,
} = require('../validations/schemas');

// SDK lookup by slug (most specific route first)
router.get('/by-slug/:slug', validate(getPromptBySlugSchema), getPromptBySlug);

// Prompt CRUD
router.route('/')
  .get(getPrompts)
  .post(validate(createPromptSchema), createPrompt);

router.route('/:promptId')
  .get(validate(promptIdParamSchema), getPrompt)
  .put(validate(updatePromptSchema), updatePrompt)
  .delete(validate(promptIdParamSchema), deletePrompt);

// Version management
router.post('/:promptId/versions', validate(createVersionSchema), createVersion);
router.get('/:promptId/versions', validate(promptIdParamSchema), getVersions);
router.get('/:promptId/versions/:version', getVersion);

// Deployment
router.post('/:promptId/deploy', validate(deployVersionSchema), deployVersion);

module.exports = router;
```

- [ ] **Step 2: Register in server.js**

In `foil/server/src/server.js`, add the import near other route imports:

```javascript
const promptRoutes = require('./routes/promptRoutes');
```

Add the route mount near the other `apiLimiter` routes (around line 261):

```javascript
app.use('/api/prompts', apiLimiter, protect, requireActiveBilling, promptRoutes);
```

- [ ] **Step 3: Verify server starts**

Run: `cd foil/server && node -e "require('./src/routes/promptRoutes'); console.log('Routes loaded OK')"`
Expected: `Routes loaded OK`

- [ ] **Step 4: Commit**

```bash
git add foil/server/src/routes/promptRoutes.js foil/server/src/server.js
git commit -m "feat: add prompt routes and register in server"
```

---

### Task 9: Plan Gating for Prompts

**Files:**
- Modify: `foil/server/src/config/stripe.js` (add `prompts` feature flag)
- Modify: `foil/server/src/middleware/usageMiddleware.js` (add `checkPromptLimit`)
- Modify: `foil/server/src/routes/promptRoutes.js` (apply middleware)

- [ ] **Step 1: Add prompts feature to plan config**

In `foil/server/src/config/stripe.js`, add to each plan's `features` object:

```javascript
// trial:
prompts: 3,  // 3 prompts during trial

// starter:
prompts: 0,  // No prompts on Starter (no LLM features)

// pro:
prompts: -1, // Unlimited prompts
```

- [ ] **Step 2: Add prompt limit constants and middleware**

In `foil/server/src/middleware/usageMiddleware.js`, add:

```javascript
const PROMPT_LIMITS = {
  trial: 3,
  starter: 0,
  pro: -1, // Unlimited
};

const checkPromptLimit = asyncHandler(async (req, res, next) => {
  if (!req.user) return next();

  // Only check on POST (creation)
  if (req.method !== 'POST') return next();

  const subscription = await getCachedSubscription(req.user._id);
  const plan = subscription?.plan || 'trial';
  const limit = PROMPT_LIMITS[plan];

  if (limit === -1) return next();

  if (limit === 0) {
    res.status(403);
    throw new Error('Prompt management is not available on your plan. Please upgrade to Pro.');
  }

  const Prompt = require('../models/promptModel');
  const count = await Prompt.countDocuments({ user: req.user._id });

  if (count >= limit) {
    logger.warn({ userId: req.user._id, count, limit }, 'Prompt limit reached');
    res.status(403);
    throw new Error(`Your plan is limited to ${limit} prompts. Please upgrade for more.`);
  }

  next();
});
```

Add `checkPromptLimit` to the `module.exports`.

- [ ] **Step 3: Apply middleware to routes**

In `foil/server/src/routes/promptRoutes.js`, import and apply:

```javascript
const { checkPromptLimit } = require('../middleware/usageMiddleware');

// Update POST route:
router.route('/')
  .get(getPrompts)
  .post(checkPromptLimit, validate(createPromptSchema), createPrompt);
```

- [ ] **Step 4: Commit**

```bash
git add foil/server/src/config/stripe.js foil/server/src/middleware/usageMiddleware.js foil/server/src/routes/promptRoutes.js
git commit -m "feat: add plan-based prompt limits (trial: 3, starter: 0, pro: unlimited)"
```

---

## Chunk 3: SDK Integration

### Task 10: JavaScript SDK — getPrompt method

**Files:**
- Modify: `foil/sdks/javascript/src/foil.js`
- Test: `foil/sdks/javascript/test/getPrompt.test.js` (create if tests dir exists, otherwise note manual testing)

- [ ] **Step 1: Add getPrompt method to Foil class**

In `foil/sdks/javascript/src/foil.js`, add after the existing `getExperimentVariant` method:

```javascript
/**
 * Get a prompt by slug for use in your application.
 * Returns the deployed version for the specified environment, or latest if no deployment exists.
 *
 * @param {string} slug - The prompt slug
 * @param {Object} [options] - Options
 * @param {string} [options.environment='production'] - Environment to fetch (development, staging, production)
 * @param {number} [options.version] - Specific version number (overrides environment deployment)
 * @returns {Promise<Object>} Prompt data with content, variables, model, and modelParameters
 */
async getPrompt(slug, options = {}) {
  if (!slug || typeof slug !== 'string') {
    throw new Error('slug is required and must be a string');
  }

  try {
    const params = {};
    if (options.environment) params.environment = options.environment;
    if (options.version) params.version = String(options.version);

    const response = await axios.get(
      `${this.baseUrl}/prompts/by-slug/${encodeURIComponent(slug)}`,
      {
        params,
        headers: {
          Authorization: this.apiKey,
        },
      }
    );
    return response.data;
  } catch (error) {
    console.error('Foil Get Prompt Failed:', error.message);
    throw error;
  }
}

/**
 * Compile a prompt template by replacing {{variable}} placeholders with values.
 *
 * @param {string} template - The prompt template string
 * @param {Object} variables - Key-value pairs for variable substitution
 * @returns {string} Compiled prompt
 */
static compilePrompt(template, variables = {}) {
  return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
    if (key in variables) return variables[key];
    return match; // Leave unresolved variables as-is
  });
}
```

- [ ] **Step 2: Update span creation to accept promptId and promptVersion**

In the `endSpan` / span data builder section, ensure `promptId` and `promptVersion` are passed through when provided in span options:

```javascript
// In the span data object passed to endSpan:
promptId: spanData.promptId || undefined,
promptVersion: spanData.promptVersion || undefined,
```

- [ ] **Step 3: Verify SDK loads**

Run: `cd foil/sdks/javascript && node -e "const { Foil } = require('./src/foil'); console.log(typeof new Foil({apiKey: 'test', agentName: 'test'}).getPrompt)"`
Expected: `function`

- [ ] **Step 4: Commit**

```bash
git add foil/sdks/javascript/src/foil.js
git commit -m "feat(sdk): add getPrompt and compilePrompt to JavaScript SDK"
```

---

### Task 11: Python SDK — get_prompt method

**Files:**
- Modify: `foil/sdks/python/foil/client.py`

- [ ] **Step 1: Add get_prompt method to Foil class**

In `foil/sdks/python/foil/client.py`, add after the `get_experiment_variant` method:

```python
def get_prompt(
    self,
    slug: str,
    environment: str = "production",
    version: int = None,
) -> dict:
    """
    Get a prompt by slug for use in your application.
    Returns the deployed version for the specified environment,
    or latest if no deployment exists.

    Args:
        slug: The prompt slug
        environment: Environment to fetch (development, staging, production)
        version: Specific version number (overrides environment deployment)

    Returns:
        Prompt data with content, variables, model, and modelParameters

    Raises:
        requests.exceptions.RequestException: If the API call fails
    """
    try:
        params = {"environment": environment}
        if version is not None:
            params["version"] = str(version)

        response = requests.get(
            f"{self.base_url}/prompts/by-slug/{slug}",
            params=params,
            headers=self._headers(),
        )
        response.raise_for_status()
        return response.json()
    except Exception as e:
        print(f"Foil Get Prompt Failed: {e}")
        raise

@staticmethod
def compile_prompt(template: str, variables: dict = None) -> str:
    """
    Compile a prompt template by replacing {{variable}} placeholders.

    Args:
        template: The prompt template string
        variables: Key-value pairs for variable substitution

    Returns:
        Compiled prompt string
    """
    import re
    if variables is None:
        variables = {}

    def replacer(match):
        key = match.group(1)
        return str(variables[key]) if key in variables else match.group(0)

    return re.sub(r"\{\{(\w+)\}\}", replacer, template)
```

- [ ] **Step 2: Verify SDK loads**

Run: `cd foil/sdks/python && python -c "from foil.client import Foil; print(hasattr(Foil, 'get_prompt'))"`
Expected: `True`

- [ ] **Step 3: Commit**

```bash
git add foil/sdks/python/foil/client.py
git commit -m "feat(sdk): add get_prompt and compile_prompt to Python SDK"
```

---

### Task 12: Run full test suites

- [ ] **Step 1: Run foil/server tests**

Run: `cd foil/server && npm test`
Expected: All tests pass

- [ ] **Step 2: Run foil-ingestion tests**

Run: `cd foil-ingestion && npm run test:unit`
Expected: All tests pass

- [ ] **Step 3: Run ESLint on foil-ingestion**

Run: `cd foil-ingestion && npm run lint`
Expected: No errors

- [ ] **Step 4: Final commit if any fixups needed**

```bash
git commit -m "fix: address test/lint issues from prompt management feature"
```
