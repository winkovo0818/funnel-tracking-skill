# funnel-tracking examples

## Example 1: Web checkout funnel

### Input

```text
Please add tracking for our checkout flow. We need visit -> login -> checkout start -> checkout complete -> payment success/fail. Frontend is React, backend is Node.js, and we want one shared event schema.
```

### Expected Outcome

- Defines the minimal event set and step order
- Creates a reusable frontend tracker helper
- Adds `trackEventOnce` for milestone events
- Adds a backend `/api/track` endpoint with validation
- Stores both raw event logs and per-flow summaries

## Example 2: Mini-program form submission flow

### Input

```text
Please follow the funnel-tracking skill to align our mini-program submission flow with the existing web tracking protocol. Keep tracking failures non-blocking and preserve flow_id across resume.
```

### Expected Outcome

- Reuses the existing backend event contract
- Maps platform-specific context into the shared payload
- Keeps `flow_id` stable across resume scenarios
- Adds a retry queue for weak-network situations
- Resets flow-scoped state after terminal success

## Example 3: Abstract existing ad hoc tracking

### Input

```text
We already have scattered analytics calls in frontend and backend. Please refactor them into a general funnel-tracking architecture and document the event schema.
```

### Expected Outcome

- Extracts common context fields and IDs
- Consolidates tracking calls behind one tracker module
- Centralizes server-side validation and enrichment
- Replaces one-off storage writes with raw + summary sinks
- Documents privacy limits and rollout checklist

## Example 4: Feishu Bitable integration with Webhook

### Input

```text
Please implement tracking using Feishu Bitable as storage. Use the single-table approach with Webhook integration. Our Webhook URL is https://tvo7pfzu3em.feishu.cn/wiki/OIUewKdMyi2iLtksofmcgTMLnLf?table=tbly1FGwdiaBrwOM&view=vewwAzhS2P and requires Bearer token authentication.
```

### Expected Outcome

**Backend Implementation (Node.js)**:

```javascript
const axios = require('axios');

// Environment configuration
const FEISHU_WEBHOOK_URL = 'https://tvo7pfzu3em.feishu.cn/wiki/OIUewKdMyi2iLtksofmcgTMLnLf?table=tbly1FGwdiaBrwOM&view=vewwAzhS2P';
const FEISHU_WEBHOOK_TOKEN = process.env.FEISHU_WEBHOOK_TOKEN; // Store securely

// Event step order mapping
const STEP_ORDER_MAP = {
  'visit': 1,
  'login_success': 2,
  'assessment_start': 3,
  'assessment_complete': 4,
  'submit_success': 5,
  'submit_fail': 5,
};

async function trackEvent(eventData) {
  try {
    const payload = {
      event_id: eventData.event_id,
      event_time: eventData.event_time,
      event_date: eventData.event_time.slice(0, 10), // Extract date "2026-03-13"
      event_name: eventData.event_name,
      step_order: STEP_ORDER_MAP[eventData.event_name] || 0,
      flow_id: eventData.flow_id,
      session_id: eventData.session_id,
      anonymous_id: eventData.anonymous_id,
      user_id: eventData.user_id || '',
      page_path: eventData.page_path || '',
      page_title: eventData.page_title || '',
      source: eventData.source || 'direct',
      device_type: eventData.device_type || 'desktop',
      browser: eventData.browser || '',
      os: eventData.os || '',
      ip_city: eventData.ip_city || '',
      assessment_id: eventData.assessment_id || '',
      submit_id: eventData.submit_id || '',
      is_success: eventData.is_success !== false, // Default true
      error_code: eventData.error_code || '',
      error_msg: eventData.error_msg || '',
      duration_ms: eventData.duration_ms || 0,
      extra: JSON.stringify(eventData.extra || {}),
    };

    const response = await axios.post(FEISHU_WEBHOOK_URL, payload, {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${FEISHU_WEBHOOK_TOKEN}`,
      },
      timeout: 5000, // 5 second timeout
    });

    console.log('[Tracking] Success:', response.data);
    return { success: true };
  } catch (error) {
    console.error('[Tracking] Failed:', error.message);
    // Failure should not block main flow
    return { success: false, error: error.message };
  }
}

// Usage example
trackEvent({
  event_id: 'evt_' + Date.now(),
  event_time: new Date().toISOString(),
  event_name: 'submit_success',
  flow_id: 'flow_test123',
  session_id: 'sess_test456',
  anonymous_id: 'anon_test789',
  user_id: 'user_10086',
  page_path: '/submit',
  page_title: 'Submit Page',
  source: 'wechat',
  device_type: 'mobile',
  browser: 'wechat-miniprogram',
  os: 'iOS 18.3',
  ip_city: 'Shanghai',
  assessment_id: 'assess_001',
  submit_id: 'submit_001',
  is_success: true,
  duration_ms: 32500,
  extra: { client_platform: 'miniprogram', version: '1.2.3' },
});
```

**Complete HTTP Request Example**:

```bash
POST https://tvo7pfzu3em.feishu.cn/wiki/OIUewKdMyi2iLtksofmcgTMLnLf?table=tbly1FGwdiaBrwOM&view=vewwAzhS2P

Headers:
Content-Type: application/json
Authorization: Bearer xxxxxxx

Body:
{
  "event_id": "evt_20260313_abc123",
  "event_time": "2026-03-13T10:30:45.000Z",
  "event_date": "2026-03-13",
  "event_name": "submit_success",
  "step_order": 5,
  "flow_id": "flow_xyz789",
  "session_id": "sess_def456",
  "anonymous_id": "anon_u83hd2k9",
  "user_id": "user_10086",
  "page_path": "/submit",
  "page_title": "Submit Page",
  "source": "wechat",
  "device_type": "mobile",
  "browser": "wechat-miniprogram",
  "os": "iOS 18.3",
  "ip_city": "Shanghai",
  "assessment_id": "assess_001",
  "submit_id": "submit_001",
  "is_success": true,
  "error_code": "",
  "error_msg": "",
  "duration_ms": 32500,
  "extra": "{\"client_platform\":\"miniprogram\",\"version\":\"1.2.3\"}"
}
```

**Key Features**:

- Single-table approach (event_logs only)
- Simple Webhook integration (no token management)
- Non-blocking error handling
- 5-second timeout to prevent blocking
- Complete field mapping
- Environment variable for token security

**Feishu Bitable Setup**:

1. Create `event_logs` table with all fields
2. Go to "Automation" → "Create Automation"
3. Trigger: "When Webhook request received"
4. Action: "Add record"
5. Map Webhook payload fields to table columns
6. Copy Webhook URL and token

**Recommended Views**:

- Today's Events: Filter `event_date = today`
- Failed Events: Filter `is_success = false`
- By Flow: Group by `flow_id`, sort by `event_time`
- By Source: Group by `source`, count events

## Example 5: Automatic Feishu Bitable Setup with API

### Input

```text
Please automatically create a Feishu Bitable table for tracking. I don't want to manually create 23 fields.

Feishu App Configuration:
app_id: cli_a1b2c3d4e5f6g7h8
app_secret: xxxxxxxxxxxxxxxxxxxx

Business Flow:
1. Visit homepage (visit)
2. Login (login_success)
3. Start assessment (assessment_start)
4. Complete assessment (assessment_complete)
5. Submit result (submit_success / submit_fail)

Option: Create new Bitable (or provide existing app_token)
```

### Expected Outcome

**Complete Setup Script (Node.js/TypeScript)**:

```typescript
import axios from 'axios';

const FEISHU_APP_ID = 'cli_a1b2c3d4e5f6g7h8';
const FEISHU_APP_SECRET = 'xxxxxxxxxxxxxxxxxxxx';

const FIELD_TYPE = {
  TEXT: 1,
  NUMBER: 2,
  SELECT: 3,
  DATE: 5,
  CHECKBOX: 7,
  MULTILINE: 1, // 多行文本也是文本类型
};

// Step 1: Get tenant_access_token
async function getTenantAccessToken() {
  const response = await axios.post(
    'https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal/',
    {
      app_id: FEISHU_APP_ID,
      app_secret: FEISHU_APP_SECRET,
    }
  );
  return response.data.tenant_access_token;
}

// Step 2: Create Bitable
async function createBitable(token: string) {
  const response = await axios.post(
    'https://open.feishu.cn/open-apis/bitable/v1/apps',
    { name: '埋点数据收集' },
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    }
  );
  return response.data.data.app.app_token;
}

// Step 3: Create event_logs table
async function createTable(token: string, appToken: string) {
  const response = await axios.post(
    `https://open.feishu.cn/open-apis/bitable/v1/apps/${appToken}/tables`,
    {
      table: {
        name: 'event_logs',
        default_view_name: '全部事件',
      },
    },
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    }
  );
  return response.data.data.table_id;
}

// Step 4: Add fields
async function addField(
  token: string,
  appToken: string,
  tableId: string,
  fieldConfig: any
) {
  await axios.post(
    `https://open.feishu.cn/open-apis/bitable/v1/apps/${appToken}/tables/${tableId}/fields`,
    fieldConfig,
    {
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    }
  );
  // Add delay to avoid rate limiting
  await new Promise(resolve => setTimeout(resolve, 200));
}

// Main setup function
async function setupFeishuBitable() {
  console.log('🚀 Starting Feishu Bitable setup...\n');

  // Step 1
  console.log('1️⃣ Getting access token...');
  const token = await getTenantAccessToken();
  console.log('   ✅ Token obtained\n');

  // Step 2
  console.log('2️⃣ Creating Bitable...');
  const appToken = await createBitable(token);
  console.log(`   ✅ Bitable created: https://xxx.feishu.cn/base/${appToken}\n`);

  // Step 3
  console.log('3️⃣ Creating event_logs table...');
  const tableId = await createTable(token, appToken);
  console.log('   ✅ Table created\n');

  // Step 4
  console.log('4️⃣ Adding fields (23 fields)...');

  const fields = [
    { field_name: 'event_id', type: FIELD_TYPE.TEXT },
    {
      field_name: 'event_time',
      type: FIELD_TYPE.DATE,
      property: { date_format: 'yyyy-MM-dd HH:mm:ss' },
    },
    {
      field_name: 'event_date',
      type: FIELD_TYPE.DATE,
      property: { date_format: 'yyyy-MM-dd' },
    },
    {
      field_name: 'event_name',
      type: FIELD_TYPE.SELECT,
      property: {
        options: [
          { name: 'visit' },
          { name: 'login_success' },
          { name: 'assessment_start' },
          { name: 'assessment_complete' },
          { name: 'submit_success' },
          { name: 'submit_fail' },
        ],
      },
    },
    { field_name: 'step_order', type: FIELD_TYPE.NUMBER },
    { field_name: 'flow_id', type: FIELD_TYPE.TEXT },
    { field_name: 'session_id', type: FIELD_TYPE.TEXT },
    { field_name: 'anonymous_id', type: FIELD_TYPE.TEXT },
    { field_name: 'user_id', type: FIELD_TYPE.TEXT },
    { field_name: 'page_path', type: FIELD_TYPE.TEXT },
    { field_name: 'page_title', type: FIELD_TYPE.TEXT },
    {
      field_name: 'source',
      type: FIELD_TYPE.SELECT,
      property: {
        options: [
          { name: 'direct' },
          { name: 'wechat' },
          { name: 'alipay' },
          { name: 'feishu' },
          { name: 'search' },
          { name: 'ad' },
          { name: 'other' },
        ],
      },
    },
    {
      field_name: 'device_type',
      type: FIELD_TYPE.SELECT,
      property: {
        options: [{ name: 'mobile' }, { name: 'desktop' }],
      },
    },
    { field_name: 'browser', type: FIELD_TYPE.TEXT },
    { field_name: 'os', type: FIELD_TYPE.TEXT },
    { field_name: 'ip_city', type: FIELD_TYPE.TEXT },
    { field_name: 'assessment_id', type: FIELD_TYPE.TEXT },
    { field_name: 'submit_id', type: FIELD_TYPE.TEXT },
    { field_name: 'is_success', type: FIELD_TYPE.CHECKBOX },
    { field_name: 'error_code', type: FIELD_TYPE.TEXT },
    { field_name: 'error_msg', type: FIELD_TYPE.TEXT },
    { field_name: 'duration_ms', type: FIELD_TYPE.NUMBER },
    { field_name: 'extra', type: FIELD_TYPE.TEXT },
  ];

  for (let i = 0; i < fields.length; i++) {
    await addField(token, appToken, tableId, fields[i]);
    console.log(`   ✅ Added field ${i + 1}/23: ${fields[i].field_name}`);
  }

  console.log('\n5️⃣ Setup complete! Next steps:\n');
  console.log('   📋 Bitable URL: https://xxx.feishu.cn/base/' + appToken);
  console.log('   📋 Table ID: ' + tableId);
  console.log('\n   ⚠️  Manual step required:');
  console.log('   1. Open the Bitable in browser');
  console.log('   2. Click "自动化" → "创建自动化"');
  console.log('   3. Trigger: "当收到 Webhook 请求时"');
  console.log('   4. Action: "新增记录" to event_logs table');
  console.log('   5. Map all fields from Webhook payload');
  console.log('   6. Save and copy Webhook URL + Token');
  console.log('\n   Then add to .env:');
  console.log('   FEISHU_WEBHOOK_URL=<your_webhook_url>');
  console.log('   FEISHU_WEBHOOK_TOKEN=<your_webhook_token>');

  return { appToken, tableId };
}

// Run setup
setupFeishuBitable()
  .then(() => {
    console.log('\n✅ All done!');
    process.exit(0);
  })
  .catch((error) => {
    console.error('\n❌ Setup failed:', error.message);
    if (error.response) {
      console.error('   API Error:', error.response.data);
    }
    process.exit(1);
  });
```

**Usage**:

```bash
# Install dependencies
npm install axios

# Set environment variables
export FEISHU_APP_ID=cli_a1b2c3d4e5f6g7h8
export FEISHU_APP_SECRET=xxxxxxxxxxxxxxxxxxxx

# Run setup script
npx ts-node setup-feishu-bitable.ts
```

**Expected Output**:

```
🚀 Starting Feishu Bitable setup...

1️⃣ Getting access token...
   ✅ Token obtained

2️⃣ Creating Bitable...
   ✅ Bitable created: https://xxx.feishu.cn/base/bascnxxxxxx

3️⃣ Creating event_logs table...
   ✅ Table created

4️⃣ Adding fields (23 fields)...
   ✅ Added field 1/23: event_id
   ✅ Added field 2/23: event_time
   ✅ Added field 3/23: event_date
   ...
   ✅ Added field 23/23: extra

5️⃣ Setup complete! Next steps:

   📋 Bitable URL: https://xxx.feishu.cn/base/bascnxxxxxx
   📋 Table ID: tblxxxxxx

   ⚠️  Manual step required:
   1. Open the Bitable in browser
   2. Click "自动化" → "创建自动化"
   3. Trigger: "当收到 Webhook 请求时"
   4. Action: "新增记录" to event_logs table
   5. Map all fields from Webhook payload
   6. Save and copy Webhook URL + Token

   Then add to .env:
   FEISHU_WEBHOOK_URL=<your_webhook_url>
   FEISHU_WEBHOOK_TOKEN=<your_webhook_token>

✅ All done!
```

**Key Features**:

- Automatic Bitable and table creation
- All 23 fields configured with correct types
- Single-select options pre-configured
- Rate limiting handled (200ms delay between field creation)
- Clear next-step instructions for Webhook setup
- Error handling with detailed API error messages

**Limitations**:

- Webhook automation must be configured manually (Feishu API doesn't support it yet)
- Requires Feishu app with `bitable:bitable` permission
- `tenant_access_token` expires in 2 hours (refresh if needed)
- API rate limit: ~100 requests/minute

**When to Use**:

- Multi-environment deployment (dev/test/prod)
- Multiple projects with similar tracking needs
- Team wants reproducible setup process
- Prefer code-based infrastructure over manual clicks
