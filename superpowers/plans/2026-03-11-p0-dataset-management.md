# Dataset Management — Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add dataset storage and curation so users can save production traces as reusable test sets, manage dataset items, export as JSONL for fine-tuning, and run evaluations across datasets to compare prompt/model performance.

**Architecture:** New MongoDB models (Dataset, DatasetItem, DatasetRun, DatasetRunItem) in foil/server. CRUD API for datasets and items. A "from-traces" endpoint that pulls span data from ClickHouse/S3 to populate datasets. JSONL export endpoint. Dataset runs execute prompts via a direct LLM proxy call (reusing the wizard proxy pattern) and store results. Plan-gated: trial gets 1 dataset (100 items), starter 0, pro unlimited.

**Tech Stack:** Express 5, Mongoose 9, Zod validation, Jest tests (foil/server), ClickHouse reads via SpanRepository

**Depends on:** Prompt Management plan (Task 5's controller is used by dataset runs to resolve prompts)

---

## Chunk 1: Data Models

### Task 1: Dataset MongoDB Model

**Files:**
- Create: `foil/server/src/models/datasetModel.js`
- Test: `foil/server/tests/__unit__/models/datasetModel.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/models/datasetModel.test.js
const mongoose = require('mongoose');

let Dataset;

beforeAll(() => {
  Dataset = require('../../../src/models/datasetModel');
});

describe('Dataset Model', () => {
  it('should require user field', () => {
    const d = new Dataset({ name: 'test' });
    const err = d.validateSync();
    expect(err.errors.user).toBeDefined();
  });

  it('should require name field', () => {
    const d = new Dataset({ user: new mongoose.Types.ObjectId() });
    const err = d.validateSync();
    expect(err.errors.name).toBeDefined();
  });

  it('should default itemCount to 0', () => {
    const d = new Dataset({
      user: new mongoose.Types.ObjectId(),
      name: 'My Dataset',
    });
    expect(d.itemCount).toBe(0);
  });

  it('should default tags to empty array', () => {
    const d = new Dataset({
      user: new mongoose.Types.ObjectId(),
      name: 'My Dataset',
    });
    expect(d.tags).toEqual([]);
  });

  it('should accept valid dataset data', () => {
    const d = new Dataset({
      user: new mongoose.Types.ObjectId(),
      name: 'Evaluation Set',
      description: 'Golden test cases',
      agentId: new mongoose.Types.ObjectId(),
      tags: ['golden', 'v2'],
    });
    const err = d.validateSync();
    expect(err).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/models/datasetModel.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the Dataset model**

```javascript
// foil/server/src/models/datasetModel.js
const mongoose = require('mongoose');

const datasetSchema = mongoose.Schema(
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
    description: {
      type: String,
      trim: true,
      maxlength: 2000,
    },
    itemCount: {
      type: Number,
      default: 0,
      min: 0,
    },
    tags: {
      type: [String],
      default: [],
    },
  },
  { timestamps: true }
);

datasetSchema.index({ user: 1, createdAt: -1 });

const Dataset = mongoose.model('Dataset', datasetSchema);
module.exports = Dataset;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/models/datasetModel.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/models/datasetModel.js foil/server/tests/__unit__/models/datasetModel.test.js
git commit -m "feat: add Dataset model for dataset management"
```

---

### Task 2: DatasetItem MongoDB Model

**Files:**
- Create: `foil/server/src/models/datasetItemModel.js`
- Test: `foil/server/tests/__unit__/models/datasetItemModel.test.js`

- [ ] **Step 1: Write the failing test**

```javascript
// foil/server/tests/__unit__/models/datasetItemModel.test.js
const mongoose = require('mongoose');

let DatasetItem;

beforeAll(() => {
  DatasetItem = require('../../../src/models/datasetItemModel');
});

describe('DatasetItem Model', () => {
  it('should require dataset field', () => {
    const item = new DatasetItem({ input: 'hello' });
    const err = item.validateSync();
    expect(err.errors.dataset).toBeDefined();
  });

  it('should require input field', () => {
    const item = new DatasetItem({ dataset: new mongoose.Types.ObjectId() });
    const err = item.validateSync();
    expect(err.errors.input).toBeDefined();
  });

  it('should accept expectedOutput', () => {
    const item = new DatasetItem({
      dataset: new mongoose.Types.ObjectId(),
      input: 'What is 2+2?',
      expectedOutput: '4',
    });
    expect(item.expectedOutput).toBe('4');
  });

  it('should store sourceTraceId', () => {
    const item = new DatasetItem({
      dataset: new mongoose.Types.ObjectId(),
      input: 'hello',
      sourceTraceId: 'trace-123',
      sourceSpanId: 'span-456',
    });
    expect(item.sourceTraceId).toBe('trace-123');
  });

  it('should default metadata to empty object', () => {
    const item = new DatasetItem({
      dataset: new mongoose.Types.ObjectId(),
      input: 'test',
    });
    expect(item.metadata).toEqual({});
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/models/datasetItemModel.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the DatasetItem model**

```javascript
// foil/server/src/models/datasetItemModel.js
const mongoose = require('mongoose');

const datasetItemSchema = mongoose.Schema(
  {
    dataset: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Dataset',
      required: true,
      index: true,
    },
    input: {
      type: String,
      required: [true, 'Input is required'],
      maxlength: 100000,
    },
    expectedOutput: {
      type: String,
      maxlength: 100000,
    },
    sourceTraceId: {
      type: String,
    },
    sourceSpanId: {
      type: String,
    },
    metadata: {
      type: mongoose.Schema.Types.Mixed,
      default: () => ({}),
    },
  },
  { timestamps: true }
);

datasetItemSchema.index({ dataset: 1, createdAt: -1 });

const DatasetItem = mongoose.model('DatasetItem', datasetItemSchema);
module.exports = DatasetItem;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/models/datasetItemModel.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/models/datasetItemModel.js foil/server/tests/__unit__/models/datasetItemModel.test.js
git commit -m "feat: add DatasetItem model for dataset management"
```

---

## Chunk 2: Dataset CRUD Controller & Routes

### Task 3: Dataset Controller

**Files:**
- Create: `foil/server/src/controllers/datasetController.js`
- Test: `foil/server/tests/__unit__/controllers/datasetController.test.js`

- [ ] **Step 1: Write failing tests**

```javascript
// foil/server/tests/__unit__/controllers/datasetController.test.js
jest.mock('../../../src/models/datasetModel');
jest.mock('../../../src/models/datasetItemModel');
jest.mock('../../../src/utils/logger', () => ({
  info: jest.fn(), warn: jest.fn(), error: jest.fn(), debug: jest.fn(),
}));

const Dataset = require('../../../src/models/datasetModel');
const DatasetItem = require('../../../src/models/datasetItemModel');
const {
  createDataset,
  getDatasets,
  getDataset,
  deleteDataset,
  addItems,
  getItems,
  deleteItem,
} = require('../../../src/controllers/datasetController');

function makeReqRes(body = {}, params = {}, query = {}) {
  return {
    req: { user: { _id: 'user-1' }, body, params, query },
    res: { status: jest.fn().mockReturnThis(), json: jest.fn() },
  };
}

describe('datasetController', () => {
  beforeEach(() => jest.clearAllMocks());

  describe('createDataset', () => {
    it('should create dataset and return 201', async () => {
      Dataset.mockImplementation((data) => ({
        ...data,
        _id: 'ds-1',
        itemCount: 0,
        save: jest.fn(),
      }));

      const { req, res } = makeReqRes({ name: 'Golden Set', description: 'Test cases' });
      await createDataset(req, res);

      expect(res.status).toHaveBeenCalledWith(201);
      expect(res.json).toHaveBeenCalled();
    });
  });

  describe('getDatasets', () => {
    it('should return paginated datasets', async () => {
      Dataset.find = jest.fn().mockReturnValue({
        limit: jest.fn().mockReturnThis(),
        skip: jest.fn().mockReturnThis(),
        sort: jest.fn().mockReturnThis(),
        lean: jest.fn().mockResolvedValue([{ _id: 'ds-1', name: 'Set 1' }]),
      });
      Dataset.countDocuments = jest.fn().mockResolvedValue(1);

      const { req, res } = makeReqRes({}, {}, { page: '1', limit: '20' });
      await getDatasets(req, res);

      expect(res.json).toHaveBeenCalledWith(expect.objectContaining({ total: 1 }));
    });
  });

  describe('addItems', () => {
    it('should add items and update itemCount', async () => {
      const mockDataset = { _id: 'ds-1', user: 'user-1', itemCount: 0, save: jest.fn() };
      Dataset.findOne = jest.fn().mockResolvedValue(mockDataset);
      DatasetItem.insertMany = jest.fn().mockResolvedValue([
        { _id: 'item-1', input: 'hello' },
        { _id: 'item-2', input: 'world' },
      ]);
      DatasetItem.countDocuments = jest.fn().mockResolvedValue(2);

      const { req, res } = makeReqRes(
        { items: [{ input: 'hello' }, { input: 'world' }] },
        { datasetId: 'ds-1' }
      );
      await addItems(req, res);

      expect(DatasetItem.insertMany).toHaveBeenCalled();
      expect(mockDataset.save).toHaveBeenCalled();
      expect(res.status).toHaveBeenCalledWith(201);
    });
  });

  describe('deleteDataset', () => {
    it('should delete dataset and all items', async () => {
      Dataset.findOne = jest.fn().mockResolvedValue({ _id: 'ds-1' });
      DatasetItem.deleteMany = jest.fn().mockResolvedValue({});
      Dataset.deleteOne = jest.fn().mockResolvedValue({});

      const { req, res } = makeReqRes({}, { datasetId: 'ds-1' });
      await deleteDataset(req, res);

      expect(DatasetItem.deleteMany).toHaveBeenCalledWith({ dataset: 'ds-1' });
      expect(Dataset.deleteOne).toHaveBeenCalledWith({ _id: 'ds-1' });
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd foil/server && npx jest tests/__unit__/controllers/datasetController.test.js --no-coverage`
Expected: FAIL

- [ ] **Step 3: Write the controller**

```javascript
// foil/server/src/controllers/datasetController.js
const asyncHandler = require('express-async-handler');
const Dataset = require('../models/datasetModel');
const DatasetItem = require('../models/datasetItemModel');
const logger = require('../utils/logger');

const createDataset = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { name, description, agentId, tags } = req.body;

  const dataset = new Dataset({
    user: userId,
    name,
    description,
    agentId,
    tags,
  });

  await dataset.save();
  logger.info({ datasetId: dataset._id, name }, 'Dataset created');
  res.status(201).json(dataset);
});

const getDatasets = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { page = 1, limit = 20, agentId } = req.query;

  const query = { user: userId };
  if (agentId) query.agentId = agentId;

  const pageNum = parseInt(page, 10);
  const limitNum = parseInt(limit, 10);

  const [datasets, total] = await Promise.all([
    Dataset.find(query)
      .limit(limitNum)
      .skip((pageNum - 1) * limitNum)
      .sort({ updatedAt: -1 })
      .lean(),
    Dataset.countDocuments(query),
  ]);

  res.json({ datasets, total, page: pageNum, pages: Math.ceil(total / limitNum) });
});

const getDataset = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId }).lean();
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  res.json(dataset);
});

const updateDataset = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;
  const { name, description, agentId, tags } = req.body;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  if (name !== undefined) dataset.name = name;
  if (description !== undefined) dataset.description = description;
  if (agentId !== undefined) dataset.agentId = agentId;
  if (tags !== undefined) dataset.tags = tags;

  await dataset.save();
  logger.info({ datasetId }, 'Dataset updated');
  res.json(dataset);
});

const deleteDataset = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  await DatasetItem.deleteMany({ dataset: datasetId });
  await Dataset.deleteOne({ _id: datasetId });

  logger.info({ datasetId }, 'Dataset deleted');
  res.json({ message: 'Dataset deleted' });
});

// --- Item management ---

const addItems = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;
  const { items } = req.body;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  const docs = items.map((item) => ({
    dataset: datasetId,
    input: item.input,
    expectedOutput: item.expectedOutput,
    sourceTraceId: item.sourceTraceId,
    sourceSpanId: item.sourceSpanId,
    metadata: item.metadata,
  }));

  const inserted = await DatasetItem.insertMany(docs);

  // Update item count
  dataset.itemCount = await DatasetItem.countDocuments({ dataset: datasetId });
  await dataset.save();

  logger.info({ datasetId, added: inserted.length }, 'Dataset items added');
  res.status(201).json({ added: inserted.length, items: inserted });
});

const getItems = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;
  const { page = 1, limit = 50 } = req.query;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  const pageNum = parseInt(page, 10);
  const limitNum = parseInt(limit, 10);

  const [items, total] = await Promise.all([
    DatasetItem.find({ dataset: datasetId })
      .limit(limitNum)
      .skip((pageNum - 1) * limitNum)
      .sort({ createdAt: -1 })
      .lean(),
    DatasetItem.countDocuments({ dataset: datasetId }),
  ]);

  res.json({ items, total, page: pageNum, pages: Math.ceil(total / limitNum) });
});

const updateItem = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId, itemId } = req.params;
  const { input, expectedOutput, metadata } = req.body;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  const item = await DatasetItem.findOne({ _id: itemId, dataset: datasetId });
  if (!item) {
    res.status(404);
    throw new Error('Item not found');
  }

  if (input !== undefined) item.input = input;
  if (expectedOutput !== undefined) item.expectedOutput = expectedOutput;
  if (metadata !== undefined) item.metadata = metadata;

  await item.save();
  res.json(item);
});

const deleteItem = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId, itemId } = req.params;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  const result = await DatasetItem.deleteOne({ _id: itemId, dataset: datasetId });
  if (result.deletedCount === 0) {
    res.status(404);
    throw new Error('Item not found');
  }

  dataset.itemCount = await DatasetItem.countDocuments({ dataset: datasetId });
  await dataset.save();

  res.json({ message: 'Item deleted' });
});

// --- From traces ---

const addItemsFromTraces = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;
  const { traceIds } = req.body;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  // Get SpanRepository from app
  const spanRepository = req.app.get('spanRepository');
  if (!spanRepository) {
    res.status(500);
    throw new Error('SpanRepository not available');
  }

  const items = [];
  for (const traceId of traceIds) {
    const spans = await spanRepository.findByTraceId(userId.toString(), traceId);
    if (!spans || spans.length === 0) continue;

    // Use the root span (first span or agent type)
    const rootSpan = spans.find((s) => s.spanType === 'agent') || spans[0];

    items.push({
      dataset: datasetId,
      input: rootSpan.input || '',
      expectedOutput: rootSpan.output || '',
      sourceTraceId: traceId,
      sourceSpanId: rootSpan.spanId,
      metadata: {
        model: rootSpan.model,
        spanType: rootSpan.spanType,
        importedAt: new Date().toISOString(),
      },
    });
  }

  if (items.length === 0) {
    res.status(400);
    throw new Error('No valid traces found for the provided IDs');
  }

  const inserted = await DatasetItem.insertMany(items);

  dataset.itemCount = await DatasetItem.countDocuments({ dataset: datasetId });
  await dataset.save();

  logger.info({ datasetId, added: inserted.length, traceCount: traceIds.length }, 'Items added from traces');
  res.status(201).json({ added: inserted.length, items: inserted });
});

// --- Export ---

const exportDataset = asyncHandler(async (req, res) => {
  const userId = req.user._id;
  const { datasetId } = req.params;
  const { format = 'jsonl' } = req.query;

  const dataset = await Dataset.findOne({ _id: datasetId, user: userId });
  if (!dataset) {
    res.status(404);
    throw new Error('Dataset not found');
  }

  const items = await DatasetItem.find({ dataset: datasetId }).lean();

  if (format === 'jsonl') {
    res.setHeader('Content-Type', 'application/jsonl');
    res.setHeader('Content-Disposition', `attachment; filename="${dataset.name.replace(/[^a-z0-9]/gi, '_')}.jsonl"`);

    const lines = items.map((item) =>
      JSON.stringify({
        input: item.input,
        expected_output: item.expectedOutput || null,
        metadata: item.metadata || {},
      })
    );

    res.send(lines.join('\n'));
  } else {
    res.status(400);
    throw new Error('Unsupported format. Use "jsonl".');
  }
});

module.exports = {
  createDataset,
  getDatasets,
  getDataset,
  updateDataset,
  deleteDataset,
  addItems,
  getItems,
  updateItem,
  deleteItem,
  addItemsFromTraces,
  exportDataset,
};
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd foil/server && npx jest tests/__unit__/controllers/datasetController.test.js --no-coverage`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add foil/server/src/controllers/datasetController.js foil/server/tests/__unit__/controllers/datasetController.test.js
git commit -m "feat: add dataset controller with CRUD, items, from-traces, and JSONL export"
```

---

### Task 4: Dataset Validation Schemas

**Files:**
- Modify: `foil/server/src/validations/schemas.js`

- [ ] **Step 1: Add dataset schemas**

Append to `foil/server/src/validations/schemas.js`:

```javascript
// Dataset schemas
const createDatasetSchema = z.object({
  body: z.object({
    name: z.string().min(1, 'Name is required').max(200),
    description: z.string().max(2000).optional(),
    agentId: z.string().optional(),
    tags: z.array(z.string()).optional(),
  }),
  query: z.object({}).passthrough().optional(),
  params: z.object({}).passthrough().optional(),
});

const updateDatasetSchema = z.object({
  body: z.object({
    name: z.string().min(1).max(200).optional(),
    description: z.string().max(2000).optional(),
    agentId: z.string().nullable().optional(),
    tags: z.array(z.string()).optional(),
  }),
  params: z.object({ datasetId: z.string().min(1) }),
  query: z.object({}).passthrough().optional(),
});

const datasetIdParamSchema = z.object({
  params: z.object({ datasetId: z.string().min(1) }),
  body: z.object({}).passthrough().optional(),
  query: z.object({}).passthrough().optional(),
});

const addItemsSchema = z.object({
  body: z.object({
    items: z.array(z.object({
      input: z.string().min(1, 'Input is required').max(100000),
      expectedOutput: z.string().max(100000).optional(),
      metadata: z.record(z.unknown()).optional(),
    })).min(1, 'At least one item is required').max(100, 'Maximum 100 items per batch'),
  }),
  params: z.object({ datasetId: z.string().min(1) }),
  query: z.object({}).passthrough().optional(),
});

const updateItemSchema = z.object({
  body: z.object({
    input: z.string().min(1).max(100000).optional(),
    expectedOutput: z.string().max(100000).nullable().optional(),
    metadata: z.record(z.unknown()).optional(),
  }),
  params: z.object({
    datasetId: z.string().min(1),
    itemId: z.string().min(1),
  }),
  query: z.object({}).passthrough().optional(),
});

const addItemsFromTracesSchema = z.object({
  body: z.object({
    traceIds: z.array(z.string().min(1)).min(1).max(50, 'Maximum 50 traces per batch'),
  }),
  params: z.object({ datasetId: z.string().min(1) }),
  query: z.object({}).passthrough().optional(),
});

const exportDatasetSchema = z.object({
  params: z.object({ datasetId: z.string().min(1) }),
  query: z.object({
    format: z.enum(['jsonl']).optional(),
  }).optional(),
  body: z.object({}).passthrough().optional(),
});
```

Add all new schemas to the `module.exports`.

- [ ] **Step 2: Verify schemas load**

Run: `cd foil/server && node -e "const s = require('./src/validations/schemas'); console.log(Object.keys(s).filter(k => k.includes('ataset')))"`
Expected: lists all dataset schema names

- [ ] **Step 3: Commit**

```bash
git add foil/server/src/validations/schemas.js
git commit -m "feat: add Zod validation schemas for dataset endpoints"
```

---

### Task 5: Dataset Routes & Server Registration

**Files:**
- Create: `foil/server/src/routes/datasetRoutes.js`
- Modify: `foil/server/src/server.js`

- [ ] **Step 1: Create routes file**

```javascript
// foil/server/src/routes/datasetRoutes.js
const express = require('express');
const router = express.Router();
const {
  createDataset,
  getDatasets,
  getDataset,
  updateDataset,
  deleteDataset,
  addItems,
  getItems,
  updateItem,
  deleteItem,
  addItemsFromTraces,
  exportDataset,
} = require('../controllers/datasetController');
const { validate } = require('../middleware/validateMiddleware');
const {
  createDatasetSchema,
  updateDatasetSchema,
  datasetIdParamSchema,
  addItemsSchema,
  updateItemSchema,
  addItemsFromTracesSchema,
  exportDatasetSchema,
} = require('../validations/schemas');

// Dataset CRUD
router.route('/')
  .get(getDatasets)
  .post(validate(createDatasetSchema), createDataset);

router.route('/:datasetId')
  .get(validate(datasetIdParamSchema), getDataset)
  .put(validate(updateDatasetSchema), updateDataset)
  .delete(validate(datasetIdParamSchema), deleteDataset);

// Item management
router.post('/:datasetId/items', validate(addItemsSchema), addItems);
router.get('/:datasetId/items', validate(datasetIdParamSchema), getItems);
router.put('/:datasetId/items/:itemId', validate(updateItemSchema), updateItem);
router.delete('/:datasetId/items/:itemId', deleteItem);

// From traces
router.post('/:datasetId/from-traces', validate(addItemsFromTracesSchema), addItemsFromTraces);

// Export
router.get('/:datasetId/export', validate(exportDatasetSchema), exportDataset);

module.exports = router;
```

- [ ] **Step 2: Register in server.js**

In `foil/server/src/server.js`, add:

```javascript
const datasetRoutes = require('./routes/datasetRoutes');
```

Add near the other apiLimiter routes:

```javascript
app.use('/api/datasets', apiLimiter, protect, requireActiveBilling, datasetRoutes);
```

- [ ] **Step 3: Verify routes load**

Run: `cd foil/server && node -e "require('./src/routes/datasetRoutes'); console.log('Routes loaded OK')"`
Expected: `Routes loaded OK`

- [ ] **Step 4: Commit**

```bash
git add foil/server/src/routes/datasetRoutes.js foil/server/src/server.js
git commit -m "feat: add dataset routes and register in server"
```

---

### Task 6: Plan Gating for Datasets

**Files:**
- Modify: `foil/server/src/config/stripe.js`
- Modify: `foil/server/src/middleware/usageMiddleware.js`
- Modify: `foil/server/src/routes/datasetRoutes.js`

- [ ] **Step 1: Add datasets feature to plans**

In `foil/server/src/config/stripe.js`, add to each plan's `features`:

```javascript
// trial:
datasets: 1,   // 1 dataset during trial

// starter:
datasets: 0,   // No datasets on Starter

// pro:
datasets: -1,  // Unlimited datasets
```

- [ ] **Step 2: Add dataset limit middleware**

In `foil/server/src/middleware/usageMiddleware.js`:

```javascript
const DATASET_LIMITS = {
  trial: 1,
  starter: 0,
  pro: -1,
};

const DATASET_ITEM_LIMITS = {
  trial: 100,
  starter: 0,
  pro: -1,
};

const checkDatasetLimit = asyncHandler(async (req, res, next) => {
  if (!req.user || req.method !== 'POST') return next();

  const subscription = await getCachedSubscription(req.user._id);
  const plan = subscription?.plan || 'trial';
  const limit = DATASET_LIMITS[plan];

  if (limit === -1) return next();

  if (limit === 0) {
    res.status(403);
    throw new Error('Datasets are not available on your plan. Please upgrade to Pro.');
  }

  const Dataset = require('../models/datasetModel');
  const count = await Dataset.countDocuments({ user: req.user._id });

  if (count >= limit) {
    res.status(403);
    throw new Error(`Your plan is limited to ${limit} dataset(s). Please upgrade for more.`);
  }

  next();
});

const checkDatasetItemLimit = asyncHandler(async (req, res, next) => {
  if (!req.user || req.method !== 'POST') return next();

  const subscription = await getCachedSubscription(req.user._id);
  const plan = subscription?.plan || 'trial';
  const limit = DATASET_ITEM_LIMITS[plan];

  if (limit === -1) return next();

  if (limit === 0) {
    res.status(403);
    throw new Error('Datasets are not available on your plan. Please upgrade to Pro.');
  }

  const DatasetItem = require('../models/datasetItemModel');
  const { datasetId } = req.params;
  const count = await DatasetItem.countDocuments({ dataset: datasetId });
  const adding = req.body?.items?.length || req.body?.traceIds?.length || 0;

  if (count + adding > limit) {
    res.status(403);
    throw new Error(`Dataset item limit is ${limit} on your plan. Current: ${count}, adding: ${adding}. Please upgrade.`);
  }

  next();
});
```

Export both. Apply in routes:

```javascript
const { checkDatasetLimit, checkDatasetItemLimit } = require('../middleware/usageMiddleware');

router.route('/')
  .get(getDatasets)
  .post(checkDatasetLimit, validate(createDatasetSchema), createDataset);

router.post('/:datasetId/items', checkDatasetItemLimit, validate(addItemsSchema), addItems);
router.post('/:datasetId/from-traces', checkDatasetItemLimit, validate(addItemsFromTracesSchema), addItemsFromTraces);
```

- [ ] **Step 3: Commit**

```bash
git add foil/server/src/config/stripe.js foil/server/src/middleware/usageMiddleware.js foil/server/src/routes/datasetRoutes.js
git commit -m "feat: add plan-based dataset limits (trial: 1/100 items, starter: 0, pro: unlimited)"
```

---

### Task 7: Run full test suites

- [ ] **Step 1: Run foil/server tests**

Run: `cd foil/server && npm test`
Expected: All tests pass

- [ ] **Step 2: Commit if fixups needed**

```bash
git commit -m "fix: address test issues from dataset management feature"
```
