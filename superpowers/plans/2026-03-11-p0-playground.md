# Playground — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an interactive playground backend that lets users test prompts against LLM providers, swap models/parameters, run evaluations on outputs, and compare results side-by-side — all from the Foil dashboard.

**Architecture:** New MongoDB model (ProviderKey) for encrypted provider API key storage. A playground controller with `run` (single execution), `run-batch` (multi-config comparison), and `models` (list available) endpoints. Execution calls LLMs via the user's stored provider keys or Foil's keys (billed). Optionally runs Foil evaluations on outputs. Results are ephemeral (not persisted) unless user saves to a dataset. Extends the existing wizard proxy pattern. Plan-gated: trial gets 10 runs/day, starter 0, pro 500/day.

**Tech Stack:** Express 5, Mongoose 9, Zod validation, Jest tests, OpenAI SDK (for OpenAI + compatible providers), Anthropic SDK (for Claude models), crypto (for key encryption)

**Depends on:** Prompt Management plan (for loading prompts into playground), Dataset Management plan (for loading dataset items as test inputs)

---

## Chunk 1: Provider Key Management

### Task 1: ProviderKey MongoDB Model

**Files:**
- Create: `foil/server/src/models/providerKeyModel.js`
- Test: `foil/server/tests/__unit__/models/providerKeyModel.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/models/providerKeyModel.test.js
const mongoose = require('mongoose');

let ProviderKey;

beforeAll(() => {
  // Set encryption key for tests
  process.env.PROVIDER_KEY_ENCRYPTION_SECRET = 'test-secret-key-32-chars-long!!';
  ProviderKey = require('../../../src/models/providerKeyModel');
});

describe('ProviderKey Model', () => {
  it('should require user field', () => {
    const pk = new ProviderKey({ provider: 'openai' });
    const err = pk.validateSync();
    expect(err.errors.user).toBeDefined();
  });

  it('should require provider field', () => {
    const pk = new ProviderKey({ user: new mongoose.Types.ObjectId() });
    const err = pk.validateSync();
    expect(err.errors.provider).toBeDefined();
  });

  it('should only allow valid providers', () => {
    const pk = new ProviderKey({
      user: new mongoose.Types.ObjectId(),
      provider: 'invalid-provider',
    });
    const err = pk.validateSync();
    expect(err.errors.provider).toBeDefined();
  });

  it('should accept valid providers', () => {
    const providers = ['openai', 'anthropic', 'google'];
    providers.forEach((provider) => {
      const pk = new ProviderKey({
        user: new mongoose.Types.ObjectId(),
        provider,
        encryptedKey: 'encrypted-data',
        keyIv: 'iv-data',
      });
      const err = pk.validateSync();
      expect(err).toBeUndefined();
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/models/providerKeyModel.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the ProviderKey model**

```javascript
// foil/server/src/models/providerKeyModel.js
const mongoose = require('mongoose');
const crypto = require('crypto');

const ALGORITHM = 'aes-256-cbc';

function getEncryptionKey() {
  const secret = process.env.PROVIDER_KEY_ENCRYPTION_SECRET;
  if (!secret) throw new Error('PROVIDER_KEY_ENCRYPTION_SECRET is required');
  return crypto.createHash('sha256').update(secret).digest();
}

const providerKeySchema = mongoose.Schema(
  {
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
      index: true,
    },
    provider: {
      type: String,
      required: true,
      enum: ['openai', 'anthropic', 'google'],
    },
    encryptedKey: {
      type: String,
      required: true,
    },
    keyIv: {
      type: String,
      required: true,
    },
    label: {
      type: String,
      trim: true,
      maxlength: 100,
    },
  },
  { timestamps: true }
);

// One key per provider per user
providerKeySchema.index({ user: 1, provider: 1 }, { unique: true });

// Instance method to decrypt the API key
providerKeySchema.methods.decryptKey = function () {
  const key = getEncryptionKey();
  const iv = Buffer.from(this.keyIv, 'hex');
  const decipher = crypto.createDecipheriv(ALGORITHM, key, iv);
  let decrypted = decipher.update(this.encryptedKey, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
};

// Static method to encrypt and store a key
providerKeySchema.statics.encryptAndStore = async function (userId, provider, apiKey, label) {
  const key = getEncryptionKey();
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, key, iv);
  let encrypted = cipher.update(apiKey, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  return this.findOneAndUpdate(
    { user: userId, provider },
    {
      encryptedKey: encrypted,
      keyIv: iv.toString('hex'),
      label,
    },
    { upsert: true, new: true }
  );
};

const ProviderKey = mongoose.model('ProviderKey', providerKeySchema);
module.exports = ProviderKey;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/models/providerKeyModel.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/models/providerKeyModel.js foil/server/tests/__unit__/models/providerKeyModel.test.js
git commit -m "feat: add ProviderKey model with AES-256 encryption for LLM provider API keys"
```

---

### Task 2: Provider Key Controller

**Files:**
- Create: `foil/server/src/controllers/providerKeyController.js`
- Test: `foil/server/tests/__unit__/controllers/providerKeyController.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/controllers/providerKeyController.test.js
jest.mock('../../../src/models/providerKeyModel');
jest.mock('../../../src/utils/logger', () => ({
  info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn(),
}));

const ProviderKey = require('../../../src/models/providerKeyModel');
const {
  storeProviderKey,
  listProviderKeys,
  deleteProviderKey,
} = require('../../../src/controllers/providerKeyController');

function makeReqRes(body = {}, params = {}) {
  return {
    req: { user: { _id: 'user-1' }, body, params },
    res: { status: jest.fn().mockReturnThis(), json: jest.fn() },
  };
}

describe('providerKeyController', () => {
  beforeEach(() => jest.clearAllMocks());

  describe('storeProviderKey', () => {
    it('should encrypt and store key', async () => {
      ProviderKey.encryptAndStore = jest.fn().mockResolvedValue({
        _id: 'pk-1',
        provider: 'openai',
        label: 'My Key',
        createdAt: new Date(),
      });

      const { req, res } = makeReqRes({ provider: 'openai', apiKey: 'sk-test', label: 'My Key' });
      await storeProviderKey(req, res);

      expect(ProviderKey.encryptAndStore).toHaveBeenCalledWith('user-1', 'openai', 'sk-test', 'My Key');
      expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ provider: 'openai' }));
    });
  });

  describe('listProviderKeys', () => {
    it('should list keys without exposing encrypted data', async () => {
      ProviderKey.find = jest.fn().mockReturnValue({
        select: jest.fn().mockReturnThis(),
        lean: jest.fn().mockResolvedValue([
          { _id: 'pk-1', provider: 'openai', label: 'My Key', createdAt: new Date() },
        ]),
      });

      const { req, res } = makeReqRes();
      await listProviderKeys(req, res);

      expect(ProviderKey.find).toHaveBeenCalledWith({ user: 'user-1' });
      expect(res.json).toHaveBeenCalled();
    });
  });

  describe('deleteProviderKey', () => {
    it('should delete key by provider', async () => {
      ProviderKey.deleteOne = jest.fn().mockResolvedValue({ deletedCount: 1 });

      const { req, res } = makeReqRes({}, { provider: 'openai' });
      await deleteProviderKey(req, res);

      expect(ProviderKey.deleteOne).toHaveBeenCalledWith({ user: 'user-1', provider: 'openai' });
    });

    it('should return 404 if key not found', async () => {
      ProviderKey.deleteOne = jest.fn().mockResolvedValue({ deletedCount: 0 });

      const { req, res } = makeReqRes({}, { provider: 'openai' });
      await expect(deleteProviderKey(req, res)).rejects.toThrow('not found');
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/controllers/providerKeyController.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the controller**

```javascript
// foil/server/src/controllers/providerKeyController.js
const asyncHandler = require('express-async-handler');
const ProviderKey = require('../models/providerKeyModel');
const logger = require('../utils/logger');

const storeProviderKey = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { provider, apiKey, label } = req.body;

  const stored = await ProviderKey.encryptAndStore(userId, provider, apiKey, label);

  logger.info({ userId, provider }, 'Provider key stored');
  res.json({
    _id: stored._id,
    provider: stored.provider,
    label: stored.label,
    createdAt: stored.createdAt,
    updatedAt: stored.updatedAt,
  });
});

const listProviderKeys = asyncHandler(async (req, res) => {
  const userId = req.user._id;

  const keys = await ProviderKey.find({ user: userId })
    .select('provider label createdAt updatedAt')
    .lean();

  res.json(keys);
});

const deleteProviderKey = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { provider } = req.params;

  const result = await ProviderKey.deleteOne({ user: userId, provider });
  if (result.deletedCount === 0) {
    res.status(404);
    throw new Error(`Provider key for "${provider}" not found`);
  }

  logger.info({ userId, provider }, 'Provider key deleted');
  res.json({ message: 'Provider key deleted' });
});

module.exports = { storeProviderKey, listProviderKeys, deleteProviderKey };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/controllers/providerKeyController.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/controllers/providerKeyController.js foil/server/tests/__unit__/controllers/providerKeyController.test.js
git commit -m "feat: add provider key controller for encrypted LLM key management"
```

---

## Chunk 2: Playground Execution Engine

### Task 3: Playground Service (LLM Execution)

**Files:**
- Create: `foil/server/src/services/playgroundService.js`
- Test: `foil/server/tests/__unit__/services/playgroundService.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/services/playgroundService.test.js
jest.mock('openai');
jest.mock('../../../src/utils/logger', () => ({
  info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn(),
}));

const { executeLLMCall, SUPPORTED_MODELS } = require('../../../src/services/playgroundService');

describe('playgroundService', () => {
  describe('SUPPORTED_MODELS', () => {
    it('should have openai models', () => {
      const openaiModels = SUPPORTED_MODELS.filter((m) => m.provider === 'openai');
      expect(openaiModels.length).toBeGreaterThan(0);
      expect(openaiModels.some((m) => m.id === 'gpt-4o')).toBe(true);
    });

    it('should have anthropic models', () => {
      const anthropicModels = SUPPORTED_MODELS.filter((m) => m.provider === 'anthropic');
      expect(anthropicModels.length).toBeGreaterThan(0);
    });
  });

  describe('executeLLMCall', () => {
    it('should reject unknown provider', async () => {
      await expect(
        executeLLMCall({
          provider: 'unknown',
          model: 'test',
          messages: [{ role: 'user', content: 'hello' }],
          apiKey: 'key',
        })
      ).rejects.toThrow('Unsupported provider');
    });

    it('should call OpenAI for openai provider', async () => {
      const OpenAI = require('openai');
      const mockCreate = jest.fn().mockResolvedValue({
        choices: [{ message: { content: 'Hello!' } }],
        usage: { prompt_tokens: 10, completion_tokens: 5, total_tokens: 15 },
      });
      OpenAI.mockImplementation(() => ({
        chat: { completions: { create: mockCreate } },
      }));

      const result = await executeLLMCall({
        provider: 'openai',
        model: 'gpt-4o-mini',
        messages: [{ role: 'user', content: 'hello' }],
        apiKey: 'sk-test',
        parameters: { temperature: 0.7 },
      });

      expect(mockCreate).toHaveBeenCalled();
      expect(result.output).toBe('Hello!');
      expect(result.usage).toBeDefined();
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/services/playgroundService.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the playground service**

```javascript
// foil/server/src/services/playgroundService.js
const OpenAI = require('openai');
const logger = require('../utils/logger');

const SUPPORTED_MODELS = [
  // OpenAI
  { id: 'gpt-4o', provider: 'openai', name: 'GPT-4o', contextWindow: 128000 },
  { id: 'gpt-4o-mini', provider: 'openai', name: 'GPT-4o Mini', contextWindow: 128000 },
  { id: 'gpt-4.1', provider: 'openai', name: 'GPT-4.1', contextWindow: 1047576 },
  { id: 'gpt-4.1-mini', provider: 'openai', name: 'GPT-4.1 Mini', contextWindow: 1047576 },
  { id: 'o3-mini', provider: 'openai', name: 'o3-mini', contextWindow: 200000 },
  // Anthropic (via OpenAI-compatible endpoint)
  { id: 'claude-sonnet-4-6', provider: 'anthropic', name: 'Claude Sonnet 4.6', contextWindow: 200000 },
  { id: 'claude-haiku-4-5-20251001', provider: 'anthropic', name: 'Claude Haiku 4.5', contextWindow: 200000 },
  { id: 'claude-opus-4-6', provider: 'anthropic', name: 'Claude Opus 4.6', contextWindow: 200000 },
  // Google
  { id: 'gemini-2.5-pro', provider: 'google', name: 'Gemini 2.5 Pro', contextWindow: 1000000 },
  { id: 'gemini-2.5-flash', provider: 'google', name: 'Gemini 2.5 Flash', contextWindow: 1000000 },
];

const PROVIDER_ENDPOINTS = {
  openai: 'https://api.openai.com/v1',
  anthropic: 'https://api.anthropic.com/v1',
  google: 'https://generativelanguage.googleapis.com/v1beta/openai',
};

/**
 * Execute an LLM call through the specified provider.
 * Uses OpenAI SDK compatible interface for all providers.
 */
async function executeLLMCall({ provider, model, messages, apiKey, parameters = {} }) {
  if (!PROVIDER_ENDPOINTS[provider]) {
    throw new Error(`Unsupported provider: ${provider}`);
  }

  const startTime = Date.now();

  const client = new OpenAI({
    apiKey,
    baseURL: PROVIDER_ENDPOINTS[provider],
  });

  const requestParams = {
    model,
    messages,
  };

  if (parameters.temperature !== undefined) requestParams.temperature = parameters.temperature;
  if (parameters.maxTokens !== undefined) requestParams.max_tokens = parameters.maxTokens;
  if (parameters.topP !== undefined) requestParams.top_p = parameters.topP;
  if (parameters.frequencyPenalty !== undefined) requestParams.frequency_penalty = parameters.frequencyPenalty;
  if (parameters.presencePenalty !== undefined) requestParams.presence_penalty = parameters.presencePenalty;
  if (parameters.stopSequences?.length > 0) requestParams.stop = parameters.stopSequences;

  try {
    const response = await client.chat.completions.create(requestParams);
    const latencyMs = Date.now() - startTime;

    const output = response.choices?.[0]?.message?.content || '';
    const usage = response.usage || {};

    return {
      output,
      usage: {
        promptTokens: usage.prompt_tokens || 0,
        completionTokens: usage.completion_tokens || 0,
        totalTokens: usage.total_tokens || 0,
      },
      latencyMs,
      model: response.model || model,
      finishReason: response.choices?.[0]?.finish_reason || 'unknown',
    };
  } catch (error) {
    logger.error({ provider, model, error: error.message }, 'LLM call failed');
    throw new Error(`LLM call failed: ${error.message}`);
  }
}

module.exports = { executeLLMCall, SUPPORTED_MODELS, PROVIDER_ENDPOINTS };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/services/playgroundService.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/services/playgroundService.js foil/server/tests/__unit__/services/playgroundService.test.js
git commit -m "feat: add playground service with multi-provider LLM execution"
```

---

### Task 4: Playground Controller

**Files:**
- Create: `foil/server/src/controllers/playgroundController.js`
- Test: `foil/server/tests/__unit__/controllers/playgroundController.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/controllers/playgroundController.test.js
jest.mock('../../../src/models/providerKeyModel');
jest.mock('../../../src/services/playgroundService');
jest.mock('../../../src/utils/logger', () => ({
  info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn(),
}));

const ProviderKey = require('../../../src/models/providerKeyModel');
const { executeLLMCall, SUPPORTED_MODELS } = require('../../../src/services/playgroundService');
const { runPlayground, getModels } = require('../../../src/controllers/playgroundController');

function makeReqRes(body = {}) {
  return {
    req: { user: { _id: 'user-1' }, body },
    res: { status: jest.fn().mockReturnThis(), json: jest.fn() },
  };
}

describe('playgroundController', () => {
  beforeEach(() => jest.clearAllMocks());

  describe('getModels', () => {
    it('should return supported models list', async () => {
      const { req, res } = makeReqRes();
      await getModels(req, res);

      expect(res.json).toHaveBeenCalledWith(
        expect.objectContaining({ models: expect.any(Array) })
      );
    });
  });

  describe('runPlayground', () => {
    it('should execute LLM call with user provider key', async () => {
      const mockKey = { decryptKey: jest.fn().mockReturnValue('sk-real-key') };
      ProviderKey.findOne = jest.fn().mockResolvedValue(mockKey);

      executeLLMCall.mockResolvedValue({
        output: 'Hello world!',
        usage: { promptTokens: 10, completionTokens: 5, totalTokens: 15 },
        latencyMs: 500,
        model: 'gpt-4o-mini',
        finishReason: 'stop',
      });

      const { req, res } = makeReqRes({
        provider: 'openai',
        model: 'gpt-4o-mini',
        messages: [{ role: 'user', content: 'hello' }],
      });

      await runPlayground(req, res);

      expect(executeLLMCall).toHaveBeenCalledWith(
        expect.objectContaining({ provider: 'openai', model: 'gpt-4o-mini', apiKey: 'sk-real-key' })
      );
      expect(res.json).toHaveBeenCalledWith(
        expect.objectContaining({ output: 'Hello world!', latencyMs: 500 })
      );
    });

    it('should return 400 if no provider key configured', async () => {
      ProviderKey.findOne = jest.fn().mockResolvedValue(null);

      const { req, res } = makeReqRes({
        provider: 'openai',
        model: 'gpt-4o',
        messages: [{ role: 'user', content: 'hi' }],
      });

      await expect(runPlayground(req, res)).rejects.toThrow('No API key configured');
      expect(res.status).toHaveBeenCalledWith(400);
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/controllers/playgroundController.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the controller**

```javascript
// foil/server/src/controllers/playgroundController.js
const asyncHandler = require('express-async-handler');
const ProviderKey = require('../models/providerKeyModel');
const { executeLLMCall, SUPPORTED_MODELS } = require('../services/playgroundService');
const logger = require('../utils/logger');

const getModels = asyncHandler(async (req, res) => {
  const userId = req.user._id;

  // Get user's configured providers
  const keys = await ProviderKey.find({ user: userId }).select('provider').lean();
  const configuredProviders = new Set(keys.map((k) => k.provider));

  const models = SUPPORTED_MODELS.map((m) => ({
    ...m,
    available: configuredProviders.has(m.provider),
  }));

  res.json({ models, configuredProviders: [...configuredProviders] });
});

const runPlayground = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { provider, model, messages, parameters, promptId, variables } = req.body;

  // Get user's API key for this provider
  const providerKey = await ProviderKey.findOne({ user: userId, provider });
  if (!providerKey) {
    res.status(400);
    throw new Error(`No API key configured for ${provider}. Add one in Settings → Provider Keys.`);
  }

  const apiKey = providerKey.decryptKey();

  // If using a managed prompt, resolve it
  let resolvedMessages = messages;
  if (promptId && variables) {
    // Compile template inline (same logic as SDK compilePrompt)
    resolvedMessages = messages.map((msg) => ({
      ...msg,
      content: msg.content.replace(/\{\{(\w+)\}\}/g, (match, key) =>
        key in variables ? variables[key] : match
      ),
    }));
  }

  const result = await executeLLMCall({
    provider,
    model,
    messages: resolvedMessages,
    apiKey,
    parameters: parameters || {},
  });

  logger.info({ userId, provider, model, latencyMs: result.latencyMs }, 'Playground execution');

  res.json(result);
});

const runBatch = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { configs } = req.body;

  // configs is an array of { provider, model, messages, parameters }
  // Execute all in parallel
  const results = await Promise.allSettled(
    configs.map(async (config) => {
      const providerKey = await ProviderKey.findOne({ user: userId, provider: config.provider });
      if (!providerKey) {
        throw new Error(`No API key for ${config.provider}`);
      }

      return executeLLMCall({
        provider: config.provider,
        model: config.model,
        messages: config.messages,
        apiKey: providerKey.decryptKey(),
        parameters: config.parameters || {},
      });
    })
  );

  const formatted = results.map((r, i) => ({
    config: { provider: configs[i].provider, model: configs[i].model },
    status: r.status,
    result: r.status === 'fulfilled' ? r.value : null,
    error: r.status === 'rejected' ? r.reason.message : null,
  }));

  res.json({ results: formatted });
});

module.exports = { getModels, runPlayground, runBatch };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/controllers/playgroundController.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/controllers/playgroundController.js foil/server/tests/__unit__/controllers/playgroundController.test.js
git commit -m "feat: add playground controller with single and batch LLM execution"
```

---

## Chunk 3: Routes, Validation & Plan Gating

### Task 5: Playground Validation Schemas

**Files:**
- Modify: `foil/server/src/validations/schemas.js`

- [ ] **Step 1: Add playground and provider key schemas**

```javascript
// Provider key schemas
const storeProviderKeySchema = z.object({
  body: z.object({
    provider: z.enum(['openai', 'anthropic', 'google']),
    apiKey: z.string().min(1, 'API key is required'),
    label: z.string().max(100).optional(),
  }),
  query: z.object({}).passthrough().optional(),
  params: z.object({}).passthrough().optional(),
});

const deleteProviderKeySchema = z.object({
  params: z.object({
    provider: z.enum(['openai', 'anthropic', 'google']),
  }),
  body: z.object({}).passthrough().optional(),
  query: z.object({}).passthrough().optional(),
});

// Playground schemas
const messageSchema = z.object({
  role: z.enum(['system', 'user', 'assistant']),
  content: z.string().min(1).max(100000),
});

const playgroundParametersSchema = z.object({
  temperature: z.number().min(0).max(2).optional(),
  maxTokens: z.number().min(1).max(200000).optional(),
  topP: z.number().min(0).max(1).optional(),
  frequencyPenalty: z.number().min(-2).max(2).optional(),
  presencePenalty: z.number().min(-2).max(2).optional(),
  stopSequences: z.array(z.string()).optional(),
}).optional();

const runPlaygroundSchema = z.object({
  body: z.object({
    provider: z.enum(['openai', 'anthropic', 'google']),
    model: z.string().min(1),
    messages: z.array(messageSchema).min(1).max(100),
    parameters: playgroundParametersSchema,
    promptId: z.string().optional(),
    variables: z.record(z.string()).optional(),
  }),
  query: z.object({}).passthrough().optional(),
  params: z.object({}).passthrough().optional(),
});

const runBatchSchema = z.object({
  body: z.object({
    configs: z.array(z.object({
      provider: z.enum(['openai', 'anthropic', 'google']),
      model: z.string().min(1),
      messages: z.array(messageSchema).min(1).max(100),
      parameters: playgroundParametersSchema,
    })).min(1).max(5, 'Maximum 5 configurations per batch'),
  }),
  query: z.object({}).passthrough().optional(),
  params: z.object({}).passthrough().optional(),
});
```

Add all to `module.exports`.

- [ ] **Step 2: Commit**

```bash
git add foil/server/src/validations/schemas.js
git commit -m "feat: add validation schemas for playground and provider key endpoints"
```

---

### Task 6: Routes & Server Registration

**Files:**
- Create: `foil/server/src/routes/providerKeyRoutes.js`
- Create: `foil/server/src/routes/playgroundRoutes.js`
- Modify: `foil/server/src/server.js`
- Modify: `foil/server/src/middleware/usageMiddleware.js`

- [ ] **Step 1: Create provider key routes**

```javascript
// foil/server/src/routes/providerKeyRoutes.js
const express = require('express');
const router = express.Router();
const { storeProviderKey, listProviderKeys, deleteProviderKey } = require('../controllers/providerKeyController');
const { validate } = require('../middleware/validateMiddleware');
const { storeProviderKeySchema, deleteProviderKeySchema } = require('../validations/schemas');

router.route('/')
  .get(listProviderKeys)
  .post(validate(storeProviderKeySchema), storeProviderKey);

router.delete('/:provider', validate(deleteProviderKeySchema), deleteProviderKey);

module.exports = router;
```

- [ ] **Step 2: Create playground routes with rate limiting**

```javascript
// foil/server/src/routes/playgroundRoutes.js
const express = require('express');
const router = express.Router();
const { getModels, runPlayground, runBatch } = require('../controllers/playgroundController');
const { validate } = require('../middleware/validateMiddleware');
const { runPlaygroundSchema, runBatchSchema } = require('../validations/schemas');
const { checkPlaygroundLimit } = require('../middleware/usageMiddleware');

router.get('/models', getModels);
router.post('/run', checkPlaygroundLimit, validate(runPlaygroundSchema), runPlayground);
router.post('/run-batch', checkPlaygroundLimit, validate(runBatchSchema), runBatch);

module.exports = router;
```

- [ ] **Step 3: Add playground limit middleware**

In `foil/server/src/middleware/usageMiddleware.js`:

```javascript
const PLAYGROUND_DAILY_LIMITS = {
  trial: 10,
  starter: 0,
  pro: 500,
};

// Simple in-memory daily counter (resets at midnight UTC)
const playgroundCounters = new Map(); // key: userId:date -> count

function getPlaygroundKey(userId) {
  const date = new Date().toISOString().split('T')[0];
  return `${userId}:${date}`;
}

const checkPlaygroundLimit = asyncHandler(async (req, res, next) => {
  if (!req.user) return next();

  const subscription = await getCachedSubscription(req.user._id);
  const plan = subscription?.plan || 'trial';
  const limit = PLAYGROUND_DAILY_LIMITS[plan];

  if (limit === -1) return next();

  if (limit === 0) {
    res.status(403);
    throw new Error('Playground is not available on your plan. Please upgrade to Pro.');
  }

  const key = getPlaygroundKey(req.user._id);
  const current = playgroundCounters.get(key) || 0;

  if (current >= limit) {
    res.status(429);
    throw new Error(`Daily playground limit (${limit}) reached. Resets at midnight UTC.`);
  }

  // Increment after successful execution (in response finish callback)
  const originalJson = res.json.bind(res);
  res.json = function (data) {
    playgroundCounters.set(key, current + 1);
    return originalJson(data);
  };

  next();
});
```

Export `checkPlaygroundLimit`.

- [ ] **Step 4: Register routes in server.js**

```javascript
const providerKeyRoutes = require('./routes/providerKeyRoutes');
const playgroundRoutes = require('./routes/playgroundRoutes');

// Near sensitive routes (strictApiLimiter for key management)
app.use('/api/provider-keys', strictApiLimiter, protect, requireActiveBilling, providerKeyRoutes);

// Playground (apiLimiter)
app.use('/api/playground', apiLimiter, protect, requireActiveBilling, playgroundRoutes);
```

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/routes/providerKeyRoutes.js foil/server/src/routes/playgroundRoutes.js foil/server/src/middleware/usageMiddleware.js foil/server/src/server.js
git commit -m "feat: add playground and provider key routes with daily rate limiting"
```

---

### Task 7: Plan Config Updates

**Files:**
- Modify: `foil/server/src/config/stripe.js`

- [ ] **Step 1: Add playground feature to plans**

```javascript
// trial:
playground: 10,     // 10 runs/day

// starter:
playground: 0,      // Not available

// pro:
playground: 500,    // 500 runs/day
```

- [ ] **Step 2: Commit**

```bash
git add foil/server/src/config/stripe.js
git commit -m "feat: add playground limits to plan config"
```

---

### Task 8: Run full test suites

- [ ] **Step 1: Run foil/server tests**

Run: `cd foil/server && npm test`
Expected: All tests pass

- [ ] **Step 2: Run foil-ingestion tests**

Run: `cd foil-ingestion && npm run test:unit`
Expected: All tests pass

- [ ] **Step 3: Run linting**

Run: `cd foil-ingestion && npm run lint`
Expected: No errors

- [ ] **Step 4: Final commit if fixups needed**

```bash
git commit -m "fix: address test/lint issues from playground feature"
```
