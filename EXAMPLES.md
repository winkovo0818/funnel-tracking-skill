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
