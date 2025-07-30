# USSD
USSD API Documentation


# USSD JSON API Specification

## Overview

This document defines a JSON-based API for USSD (Unstructured Supplementary Service Data) services. The API provides a standardized way to handle USSD sessions, menu navigation, and user interactions.

## API Endpoints

### 1. Process USSD Request

**Endpoint:** `POST /api/v1/ussd/process`

Processes USSD requests and returns appropriate responses based on session state and user input.

#### Request Format

```json
{
  "sessionId": "unique-session-identifier",
  "serviceCode": "*123#",
  "phoneNumber": "+260970123456",
  "networkCode": "MTN",
  "input": "1",
  "newSession": true,
  "language": "en",
  "metadata": {
    "imsi": "640020123456789",
    "cellId": "12345",
    "location": {
      "lat": -15.4167,
      "lon": 28.2833
    }
  }
}
```

#### Request Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| sessionId | string | Yes | Unique identifier for the USSD session |
| serviceCode | string | Yes | USSD service code dialed by user |
| phoneNumber | string | Yes | User's phone number in E.164 format |
| networkCode | string | Yes | Mobile network operator code |
| input | string | No | User's input (empty for new sessions) |
| newSession | boolean | Yes | True for new sessions, false for continued |
| language | string | No | Preferred language code (default: "en") |
| metadata | object | No | Additional session metadata |

#### Response Format

```json
{
  "sessionId": "unique-session-identifier",
  "message": "Welcome to Mobile Banking\n1. Check Balance\n2. Send Money\n3. Buy Airtime\n4. Pay Bills\n5. Mini Statement",
  "action": "CONTINUE",
  "metadata": {
    "menuLevel": 1,
    "menuId": "main_menu",
    "sessionData": {
      "attempts": 0,
      "lastAction": "menu_display"
    }
  }
}
```

#### Response Parameters

| Field | Type | Description |
|-------|------|-------------|
| sessionId | string | Session identifier (must match request) |
| message | string | Text to display to the user |
| action | string | Either "CONTINUE" or "END" |
| metadata | object | Optional session metadata |

### 2. Get Session Status

**Endpoint:** `GET /api/v1/ussd/session/{sessionId}`

Retrieves the current status of a USSD session.

#### Response Format

```json
{
  "sessionId": "unique-session-identifier",
  "status": "active",
  "phoneNumber": "+260970123456",
  "startTime": "2024-01-15T10:30:00Z",
  "lastActivity": "2024-01-15T10:31:45Z",
  "menuPath": ["main_menu", "send_money", "enter_amount"],
  "sessionData": {
    "recipient": "0970987654",
    "amount": null
  }
}
```

### 3. Cancel Session

**Endpoint:** `DELETE /api/v1/ussd/session/{sessionId}`

Cancels an active USSD session.

#### Response Format

```json
{
  "sessionId": "unique-session-identifier",
  "status": "cancelled",
  "message": "Session cancelled successfully"
}
```

## Complete Menu Configuration Example

```json
{
  "menuConfig": {
    "main_menu": {
      "type": "menu",
      "message": "Welcome to Mobile Banking\n1. Check Balance\n2. Send Money\n3. Buy Airtime\n4. Pay Bills\n5. Mini Statement",
      "options": {
        "1": "check_balance",
        "2": "send_money",
        "3": "buy_airtime",
        "4": "pay_bills",
        "5": "mini_statement"
      }
    },
    
    "check_balance": {
      "type": "menu",
      "message": "Select Account\n1. Savings Account\n2. Current Account\n3. Mobile Wallet\n0. Back",
      "options": {
        "1": "savings_balance",
        "2": "current_balance",
        "3": "wallet_balance",
        "0": "main_menu"
      }
    },
    
    "savings_balance": {
      "type": "display",
      "message": "Savings Account\nBalance: ${balance}\nAvailable: ${available}\n\n0. Main Menu",
      "dataRequired": ["balance", "available"],
      "options": {
        "0": "main_menu"
      }
    },
    
    "send_money": {
      "type": "input",
      "message": "Enter recipient phone number:",
      "inputName": "recipient",
      "validation": {
        "type": "phone",
        "pattern": "^[0-9]{10}$",
        "errorMessage": "Invalid phone number. Please enter 10 digits."
      },
      "nextMenu": "send_money_amount"
    },
    
    "send_money_amount": {
      "type": "input",
      "message": "Enter amount to send:",
      "inputName": "amount",
      "validation": {
        "type": "numeric",
        "min": 1,
        "max": 10000,
        "errorMessage": "Amount must be between 1 and 10,000"
      },
      "nextMenu": "send_money_confirm"
    },
    
    "send_money_confirm": {
      "type": "menu",
      "message": "Confirm Transfer\nTo: ${recipient}\nAmount: $${amount}\nFee: $${fee}\nTotal: $${total}\n\n1. Confirm\n2. Cancel",
      "dataRequired": ["recipient", "amount", "fee", "total"],
      "options": {
        "1": "send_money_pin",
        "2": "main_menu"
      }
    },
    
    "send_money_pin": {
      "type": "input",
      "message": "Enter your PIN:",
      "inputName": "pin",
      "validation": {
        "type": "pin",
        "length": 4,
        "masked": true,
        "errorMessage": "Invalid PIN"
      },
      "nextMenu": "send_money_process"
    },
    
    "send_money_process": {
      "type": "action",
      "action": "process_transfer",
      "successMenu": "send_money_success",
      "failureMenu": "send_money_failed"
    },
    
    "send_money_success": {
      "type": "end",
      "message": "Transfer Successful!\nAmount: $${amount}\nTo: ${recipient}\nRef: ${reference}\nNew Balance: $${newBalance}\n\nThank you!"
    },
    
    "send_money_failed": {
      "type": "end",
      "message": "Transfer Failed\n${errorMessage}\n\nPlease try again later."
    }
  }
}
```

## Implementation Example (Node.js/Express)

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// In-memory session storage (use Redis in production)
const sessions = new Map();

// Menu configuration
const menuConfig = {
  main_menu: {
    type: "menu",
    message: "Welcome to Mobile Banking\n1. Check Balance\n2. Send Money\n3. Buy Airtime\n4. Pay Bills\n5. Mini Statement",
    options: {
      "1": "check_balance",
      "2": "send_money",
      "3": "buy_airtime",
      "4": "pay_bills",
      "5": "mini_statement"
    }
  },
  // ... rest of menu config
};

// Process USSD request
app.post('/api/v1/ussd/process', async (req, res) => {
  try {
    const {
      sessionId,
      serviceCode,
      phoneNumber,
      networkCode,
      input,
      newSession,
      language = 'en'
    } = req.body;

    let session;
    
    // Handle new session
    if (newSession) {
      session = {
        id: sessionId,
        phoneNumber,
        networkCode,
        language,
        currentMenu: 'main_menu',
        data: {},
        startTime: new Date(),
        lastActivity: new Date()
      };
      
      // Handle direct service codes
      if (serviceCode === '*123*1#') {
        session.currentMenu = 'check_balance';
      } else if (serviceCode === '*123*2#') {
        session.currentMenu = 'send_money';
      }
      
      sessions.set(sessionId, session);
    } else {
      // Get existing session
      session = sessions.get(sessionId);
      
      if (!session) {
        return res.status(400).json({
          error: 'Session not found',
          code: 'SESSION_EXPIRED'
        });
      }
      
      session.lastActivity = new Date();
    }
    
    // Process input and get response
    const response = await processMenu(session, input);
    
    // Save session if continuing
    if (response.action === 'CONTINUE') {
      sessions.set(sessionId, session);
    } else {
      sessions.delete(sessionId);
    }
    
    res.json(response);
    
  } catch (error) {
    console.error('Error processing USSD request:', error);
    res.status(500).json({
      error: 'Internal server error',
      code: 'PROCESSING_ERROR'
    });
  }
});

// Process menu logic
async function processMenu(session, input) {
  const currentMenu = menuConfig[session.currentMenu];
  
  if (!currentMenu) {
    return {
      sessionId: session.id,
      message: "Invalid menu. Please try again.",
      action: "END"
    };
  }
  
  // Handle different menu types
  switch (currentMenu.type) {
    case 'menu':
      if (input && currentMenu.options[input]) {
        session.currentMenu = currentMenu.options[input];
        const nextMenu = menuConfig[session.currentMenu];
        const message = formatMessage(nextMenu.message, session.data);
        
        return {
          sessionId: session.id,
          message,
          action: nextMenu.type === 'end' ? 'END' : 'CONTINUE',
          metadata: {
            menuId: session.currentMenu,
            menuLevel: getMenuLevel(session.currentMenu)
          }
        };
      } else if (input) {
        // Invalid option
        return {
          sessionId: session.id,
          message: `Invalid option. Please try again.\n\n${currentMenu.message}`,
          action: 'CONTINUE'
        };
      } else {
        // Display current menu
        return {
          sessionId: session.id,
          message: formatMessage(currentMenu.message, session.data),
          action: 'CONTINUE'
        };
      }
      
    case 'input':
      if (input) {
        // Validate input
        const validation = validateInput(input, currentMenu.validation);
        
        if (validation.valid) {
          session.data[currentMenu.inputName] = input;
          session.currentMenu = currentMenu.nextMenu;
          
          const nextMenu = menuConfig[session.currentMenu];
          const message = formatMessage(nextMenu.message, session.data);
          
          return {
            sessionId: session.id,
            message,
            action: 'CONTINUE'
          };
        } else {
          return {
            sessionId: session.id,
            message: validation.errorMessage || currentMenu.validation.errorMessage,
            action: 'CONTINUE'
          };
        }
      } else {
        return {
          sessionId: session.id,
          message: currentMenu.message,
          action: 'CONTINUE'
        };
      }
      
    case 'action':
      // Execute action (e.g., process payment)
      const result = await executeAction(currentMenu.action, session.data);
      
      if (result.success) {
        session.data = { ...session.data, ...result.data };
        session.currentMenu = currentMenu.successMenu;
      } else {
        session.data.errorMessage = result.errorMessage;
        session.currentMenu = currentMenu.failureMenu;
      }
      
      const nextMenu = menuConfig[session.currentMenu];
      const message = formatMessage(nextMenu.message, session.data);
      
      return {
        sessionId: session.id,
        message,
        action: nextMenu.type === 'end' ? 'END' : 'CONTINUE'
      };
      
    case 'display':
    case 'end':
      return {
        sessionId: session.id,
        message: formatMessage(currentMenu.message, session.data),
        action: currentMenu.type === 'end' ? 'END' : 'CONTINUE'
      };
      
    default:
      return {
        sessionId: session.id,
        message: "Service error. Please try again.",
        action: "END"
      };
  }
}

// Helper functions
function formatMessage(template, data) {
  return template.replace(/\${(\w+)}/g, (match, key) => {
    return data[key] || match;
  });
}

function validateInput(input, rules) {
  if (!rules) return { valid: true };
  
  switch (rules.type) {
    case 'phone':
      return {
        valid: new RegExp(rules.pattern).test(input),
        errorMessage: rules.errorMessage
      };
      
    case 'numeric':
      const num = parseFloat(input);
      return {
        valid: !isNaN(num) && num >= rules.min && num <= rules.max,
        errorMessage: rules.errorMessage
      };
      
    case 'pin':
      return {
        valid: input.length === rules.length && /^\d+$/.test(input),
        errorMessage: rules.errorMessage
      };
      
    default:
      return { valid: true };
  }
}

async function executeAction(action, data) {
  // Implement actual business logic here
  switch (action) {
    case 'process_transfer':
      // Simulate transfer processing
      return {
        success: true,
        data: {
          reference: 'TXN' + Date.now(),
          newBalance: 4500.00
        }
      };
      
    default:
      return {
        success: false,
        errorMessage: 'Service temporarily unavailable'
      };
  }
}

function getMenuLevel(menuId) {
  // Calculate menu depth for analytics
  const levels = {
    'main_menu': 0,
    'check_balance': 1,
    'send_money': 1,
    'savings_balance': 2,
    'send_money_amount': 2,
    'send_money_confirm': 3
  };
  return levels[menuId] || 0;
}

// Session cleanup (run periodically)
setInterval(() => {
  const timeout = 5 * 60 * 1000; // 5 minutes
  const now = Date.now();
  
  for (const [sessionId, session] of sessions.entries()) {
    if (now - session.lastActivity.getTime() > timeout) {
      sessions.delete(sessionId);
    }
  }
}, 60 * 1000); // Run every minute

// Start server
app.listen(3000, () => {
  console.log('USSD API running on port 3000');
});
```

## Complete Flow Examples

### Example 1: Check Balance Flow

```json
// 1. New session - User dials *123*1#
POST /api/v1/ussd/process
{
  "sessionId": "sess-001",
  "serviceCode": "*123*1#",
  "phoneNumber": "+260970123456",
  "networkCode": "MTN",
  "input": "",
  "newSession": true
}

Response:
{
  "sessionId": "sess-001",
  "message": "Select Account\n1. Savings Account\n2. Current Account\n3. Mobile Wallet\n0. Back",
  "action": "CONTINUE"
}

// 2. User selects Savings Account
POST /api/v1/ussd/process
{
  "sessionId": "sess-001",
  "input": "1",
  "newSession": false
}

Response:
{
  "sessionId": "sess-001",
  "message": "Savings Account\nBalance: $1,234.56\nAvailable: $1,234.56\n\n0. Main Menu",
  "action": "CONTINUE"
}

// 3. User selects 0 to end
POST /api/v1/ussd/process
{
  "sessionId": "sess-001",
  "input": "0",
  "newSession": false
}

Response:
{
  "sessionId": "sess-001",
  "message": "Thank you for using Mobile Banking!",
  "action": "END"
}
```

### Example 2: Send Money Flow

```json
// 1. Start send money
POST /api/v1/ussd/process
{
  "sessionId": "sess-002",
  "serviceCode": "*123*2#",
  "phoneNumber": "+260970123456",
  "networkCode": "AIRTEL",
  "input": "",
  "newSession": true
}

Response:
{
  "sessionId": "sess-002",
  "message": "Enter recipient phone number:",
  "action": "CONTINUE"
}

// 2. Enter recipient
POST /api/v1/ussd/process
{
  "sessionId": "sess-002",
  "input": "0970987654",
  "newSession": false
}

Response:
{
  "sessionId": "sess-002",
  "message": "Enter amount to send:",
  "action": "CONTINUE"
}

// 3. Enter amount
POST /api/v1/ussd/process
{
  "sessionId": "sess-002",
  "input": "100",
  "newSession": false
}

Response:
{
  "sessionId": "sess-002",
  "message": "Confirm Transfer\nTo: 0970987654\nAmount: $100\nFee: $2\nTotal: $102\n\n1. Confirm\n2. Cancel",
  "action": "CONTINUE"
}

// 4. Confirm
POST /api/v1/ussd/process
{
  "sessionId": "sess-002",
  "input": "1",
  "newSession": false
}

Response:
{
  "sessionId": "sess-002",
  "message": "Enter your PIN:",
  "action": "CONTINUE"
}

// 5. Enter PIN
POST /api/v1/ussd/process
{
  "sessionId": "sess-002",
  "input": "1234",
  "newSession": false
}

Response:
{
  "sessionId": "sess-002",
  "message": "Transfer Successful!\nAmount: $100\nTo: 0970987654\nRef: TXN1642345678\nNew Balance: $4500.00\n\nThank you!",
  "action": "END"
}
```

## Error Handling

### Error Response Format

```json
{
  "error": "Error description",
  "code": "ERROR_CODE",
  "details": {
    "field": "Additional error details"
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| SESSION_EXPIRED | 400 | Session has expired or not found |
| INVALID_INPUT | 400 | User input validation failed |
| SERVICE_UNAVAILABLE | 503 | Backend service temporarily unavailable |
| PROCESSING_ERROR | 500 | Internal server error during processing |
| INVALID_SERVICE_CODE | 400 | Unknown or invalid USSD service code |
| RATE_LIMIT_EXCEEDED | 429 | Too many requests from same number |

## Security Considerations

1. **Session Management**
   - Use secure session tokens
   - Implement session timeouts (5 minutes recommended)
   - Clear sensitive data after session ends

2. **Input Validation**
   - Validate all user inputs
   - Sanitize data to prevent injection attacks
   - Implement rate limiting per phone number

3. **Data Protection**
   - Encrypt sensitive data in transit (HTTPS)
   - Mask sensitive information (PINs, passwords)
   - Log security events for audit trails

4. **Authentication**
   - API key authentication for USSD gateway
   - PIN verification for sensitive operations
   - Two-factor authentication for high-value transactions

## Testing

### Test Environment

```bash
# Health check
GET https://api-test.example.com/api/v1/health

# Test USSD endpoint
POST https://api-test.example.com/api/v1/ussd/process
```

### Sample Test Cases

```json
{
  "testCases": [
    {
      "name": "New Session",
      "request": {
        "sessionId": "test-001",
        "serviceCode": "*123#",
        "phoneNumber": "+260970123456",
        "networkCode": "TEST",
        "input": "",
        "newSession": true
      },
      "expectedResponse": {
        "action": "CONTINUE",
        "messageContains": "Welcome to Mobile Banking"
      }
    },
    {
      "name": "Invalid Menu Option",
      "request": {
        "sessionId": "test-002",
        "input": "9",
        "newSession": false
      },
      "expectedResponse": {
        "action": "CONTINUE",
        "messageContains": "Invalid option"
      }
    }
  ]
}
```

## Performance Metrics

### Response Time Requirements

- Average response time: < 200ms
- 95th percentile: < 500ms
- 99th percentile: < 1000ms

### Capacity Planning

- Sessions per second: 1000
- Concurrent sessions: 50,000
- Session timeout: 5 minutes
- Message size limit: 160 characters

## Integration with USSD Gateways

### Africastalking Format

```json
{
  "sessionId": "ATUid_1234567890",
  "phoneNumber": "+260970123456",
  "networkCode": "63902",
  "serviceCode": "*123#",
  "text": "1*2*100"
}
```

### Flares Format

```json
{
  "session_id": "flares_1234567890",
  "msisdn": "260970123456",
  "ussd_string": "*123#",
  "input": "1",
  "new_session": true
}
```

### Adapter Example

```javascript
// Convert from gateway format to standard API format
function adaptGatewayRequest(gateway, request) {
  switch (gateway) {
    case 'africastalking':
      const inputs = request.text ? request.text.split('*') : [];
      return {
        sessionId: request.sessionId,
        serviceCode: request.serviceCode,
        phoneNumber: request.phoneNumber,
        networkCode: request.networkCode,
        input: inputs[inputs.length - 1] || '',
        newSession: request.text === ''
      };
      
    case 'flares':
      return {
        sessionId: request.session_id,
        serviceCode: request.ussd_string,
        phoneNumber: '+' + request.msisdn,
        networkCode: 'UNKNOWN',
        input: request.input,
        newSession: request.new_session
      };
      
    default:
      throw new Error('Unknown gateway');
  }
}
```

## Monitoring and Analytics

### Key Metrics to Track

```json
{
  "metrics": {
    "sessions": {
      "total": 15234,
      "completed": 12456,
      "abandoned": 2778,
      "averageDuration": 125.4
    },
    "menuUsage": {
      "main_menu": 15234,
      "check_balance": 8456,
      "send_money": 4532,
      "buy_airtime": 2246
    },
    "errors": {
      "session_expired": 234,
      "invalid_input": 567,
      "service_unavailable": 12
    },
    "performance": {
      "avgResponseTime": 145,
      "p95ResponseTime": 456,
      "p99ResponseTime": 892
    }
  }
}
```

This JSON API specification provides a complete, standardized approach to building USSD services with proper session management, error handling, and integration capabilities.



{
  "info": {
    "name": "USSD API Collection",
    "description": "Complete Postman collection for testing USSD API endpoints. This collection includes all API endpoints, test scenarios, and example flows for USSD services.",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    {
      "key": "baseUrl",
      "value": "https://api.example.com/api/v1",
      "type": "string",
      "description": "Base URL for the USSD API"
    },
    {
      "key": "sessionId",
      "value": "",
      "type": "string",
      "description": "Current session ID (automatically set by tests)"
    },
    {
      "key": "phoneNumber",
      "value": "+260970123456",
      "type": "string",
      "description": "Test phone number"
    },
    {
      "key": "serviceCode",
      "value": "*123#",
      "type": "string",
      "description": "USSD service code"
    },
    {
      "key": "networkCode",
      "value": "MTN",
      "type": "string",
      "description": "Mobile network operator code"
    }
  ],
  "item": [
    {
      "name": "1. Basic Operations",
      "item": [
        {
          "name": "Health Check",
          "request": {
            "method": "GET",
            "header": [],
            "url": {
              "raw": "{{baseUrl}}/health",
              "host": ["{{baseUrl}}"],
              "path": ["health"]
            },
            "description": "Check if the API service is running"
          },
          "response": []
        },
        {
          "name": "Start New Session - Main Menu",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "// Test response",
                  "pm.test(\"Status code is 200\", function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test(\"Response has required fields\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData).to.have.property('sessionId');",
                  "    pm.expect(jsonData).to.have.property('message');",
                  "    pm.expect(jsonData).to.have.property('action');",
                  "});",
                  "",
                  "pm.test(\"Action is CONTINUE\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.action).to.eql('CONTINUE');",
                  "});",
                  "",
                  "pm.test(\"Main menu is displayed\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.message).to.include('Welcome to Mobile Banking');",
                  "});",
                  "",
                  "// Save session ID for next requests",
                  "const jsonData = pm.response.json();",
                  "pm.collectionVariables.set(\"sessionId\", jsonData.sessionId);"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"{{$guid}}\",\n  \"serviceCode\": \"{{serviceCode}}\",\n  \"phoneNumber\": \"{{phoneNumber}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true,\n  \"language\": \"en\"\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Start a new USSD session with main menu"
          },
          "response": []
        },
        {
          "name": "Get Session Status",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Status code is 200\", function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test(\"Session is active\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.status).to.eql('active');",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "GET",
            "header": [],
            "url": {
              "raw": "{{baseUrl}}/ussd/session/{{sessionId}}",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "session", "{{sessionId}}"]
            },
            "description": "Get the current status of a USSD session"
          },
          "response": []
        },
        {
          "name": "Cancel Session",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Status code is 200\", function () {",
                  "    pm.response.to.have.status(200);",
                  "});",
                  "",
                  "pm.test(\"Session is cancelled\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.status).to.eql('cancelled');",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "DELETE",
            "header": [],
            "url": {
              "raw": "{{baseUrl}}/ussd/session/{{sessionId}}",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "session", "{{sessionId}}"]
            },
            "description": "Cancel an active USSD session"
          },
          "response": []
        }
      ]
    },
    {
      "name": "2. Direct Service Codes",
      "item": [
        {
          "name": "Direct - Check Balance (*123*1#)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Direct access to check balance menu\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.message).to.include('Select Account');",
                  "});",
                  "",
                  "// Save session ID",
                  "const jsonData = pm.response.json();",
                  "pm.collectionVariables.set(\"sessionId\", jsonData.sessionId);"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"{{$guid}}\",\n  \"serviceCode\": \"*123*1#\",\n  \"phoneNumber\": \"{{phoneNumber}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Direct access to Check Balance menu using *123*1#"
          },
          "response": []
        },
        {
          "name": "Direct - Send Money (*123*2#)",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Direct access to send money\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.message).to.include('Enter recipient phone number');",
                  "});",
                  "",
                  "// Save session ID",
                  "const jsonData = pm.response.json();",
                  "pm.collectionVariables.set(\"sessionId\", jsonData.sessionId);"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"{{$guid}}\",\n  \"serviceCode\": \"*123*2#\",\n  \"phoneNumber\": \"{{phoneNumber}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Direct access to Send Money using *123*2#"
          },
          "response": []
        },
        {
          "name": "Direct - Buy Airtime (*123*3#)",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"{{$guid}}\",\n  \"serviceCode\": \"*123*3#\",\n  \"phoneNumber\": \"{{phoneNumber}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Direct access to Buy Airtime using *123*3#"
          },
          "response": []
        }
      ]
    },
    {
      "name": "3. Complete Flows",
      "item": [
        {
          "name": "Check Balance Flow",
          "item": [
            {
              "name": "1. Start Session",
              "event": [
                {
                  "listen": "test",
                  "script": {
                    "exec": [
                      "const jsonData = pm.response.json();",
                      "pm.collectionVariables.set(\"sessionId\", jsonData.sessionId);",
                      "pm.collectionVariables.set(\"currentFlow\", \"checkBalance\");"
                    ],
                    "type": "text/javascript"
                  }
                }
              ],
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"check-balance-{{$timestamp}}\",\n  \"serviceCode\": \"*123#\",\n  \"phoneNumber\": \"{{phoneNumber}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "2. Select Check Balance (Option 1)",
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"1\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "3. Select Savings Account (Option 1)",
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"1\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "4. Return to Main Menu (Option 0)",
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"0\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            }
          ]
        },
        {
          "name": "Send Money Flow",
          "item": [
            {
              "name": "1. Start Send Money",
              "event": [
                {
                  "listen": "test",
                  "script": {
                    "exec": [
                      "const jsonData = pm.response.json();",
                      "pm.collectionVariables.set(\"sessionId\", jsonData.sessionId);",
                      "pm.collectionVariables.set(\"currentFlow\", \"sendMoney\");"
                    ],
                    "type": "text/javascript"
                  }
                }
              ],
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"send-money-{{$timestamp}}\",\n  \"serviceCode\": \"*123*2#\",\n  \"phoneNumber\": \"{{phoneNumber}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "2. Enter Recipient Number",
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"0970987654\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "3. Enter Amount",
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"100\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "4. Confirm Transaction",
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"1\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            },
            {
              "name": "5. Enter PIN",
              "event": [
                {
                  "listen": "test",
                  "script": {
                    "exec": [
                      "pm.test(\"Transaction should complete successfully\", function () {",
                      "    const jsonData = pm.response.json();",
                      "    pm.expect(jsonData.action).to.eql('END');",
                      "    pm.expect(jsonData.message).to.include('Transfer Successful');",
                      "});"
                    ],
                    "type": "text/javascript"
                  }
                }
              ],
              "request": {
                "method": "POST",
                "header": [
                  {
                    "key": "Content-Type",
                    "value": "application/json"
                  }
                ],
                "body": {
                  "mode": "raw",
                  "raw": "{\n  \"sessionId\": \"{{sessionId}}\",\n  \"input\": \"1234\",\n  \"newSession\": false\n}"
                },
                "url": {
                  "raw": "{{baseUrl}}/ussd/process",
                  "host": ["{{baseUrl}}"],
                  "path": ["ussd", "process"]
                }
              },
              "response": []
            }
          ]
        }
      ]
    },
    {
      "name": "4. Error Scenarios",
      "item": [
        {
          "name": "Invalid Menu Option",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Should return error message\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.message).to.include('Invalid option');",
                  "    pm.expect(jsonData.action).to.eql('CONTINUE');",
                  "});"
                ],
                "type": "text/javascript"
              }
            },
            {
              "listen": "prerequest",
              "script": {
                "exec": [
                  "// First create a session",
                  "const sessionId = 'error-test-' + Date.now();",
                  "pm.collectionVariables.set('errorSessionId', sessionId);"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"{{errorSessionId}}\",\n  \"input\": \"99\",\n  \"newSession\": false\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Test invalid menu option handling"
          },
          "response": []
        },
        {
          "name": "Invalid Phone Number",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Should return validation error\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.message).to.include('Invalid phone number');",
                  "    pm.expect(jsonData.action).to.eql('CONTINUE');",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"phone-validation-{{$timestamp}}\",\n  \"input\": \"123\",\n  \"newSession\": false\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Test phone number validation"
          },
          "response": []
        },
        {
          "name": "Invalid Amount",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Should return amount validation error\", function () {",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.message).to.include('Amount must be between');",
                  "    pm.expect(jsonData.action).to.eql('CONTINUE');",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"amount-validation-{{$timestamp}}\",\n  \"input\": \"50000\",\n  \"newSession\": false\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Test amount validation (exceeds maximum)"
          },
          "response": []
        },
        {
          "name": "Session Not Found",
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "pm.test(\"Should return session error\", function () {",
                  "    pm.response.to.have.status(400);",
                  "    const jsonData = pm.response.json();",
                  "    pm.expect(jsonData.error).to.exist;",
                  "    pm.expect(jsonData.code).to.eql('SESSION_EXPIRED');",
                  "});"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"non-existent-session\",\n  \"input\": \"1\",\n  \"newSession\": false\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Test behavior when session doesn't exist"
          },
          "response": []
        }
      ]
    },
    {
      "name": "5. Load Testing",
      "item": [
        {
          "name": "Concurrent Sessions",
          "event": [
            {
              "listen": "prerequest",
              "script": {
                "exec": [
                  "// Generate unique session ID for load testing",
                  "pm.variables.set('loadTestSessionId', 'load-test-' + Date.now() + '-' + Math.random());"
                ],
                "type": "text/javascript"
              }
            }
          ],
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"{{loadTestSessionId}}\",\n  \"serviceCode\": \"*123#\",\n  \"phoneNumber\": \"+26097{{$randomInt}}\",\n  \"networkCode\": \"{{networkCode}}\",\n  \"input\": \"\",\n  \"newSession\": true\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/process",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "process"]
            },
            "description": "Use with Postman Runner to test multiple concurrent sessions"
          },
          "response": []
        }
      ]
    },
    {
      "name": "6. Gateway Integration Tests",
      "item": [
        {
          "name": "AfricasTalking Format",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"sessionId\": \"ATUid_{{$timestamp}}\",\n  \"phoneNumber\": \"+260970123456\",\n  \"networkCode\": \"63902\",\n  \"serviceCode\": \"*123#\",\n  \"text\": \"1*2*100\"\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/gateway/africastalking",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "gateway", "africastalking"]
            },
            "description": "Test AfricasTalking gateway format adapter"
          },
          "response": []
        },
        {
          "name": "Flares Format",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"session_id\": \"flares_{{$timestamp}}\",\n  \"msisdn\": \"260970123456\",\n  \"ussd_string\": \"*123#\",\n  \"input\": \"1\",\n  \"new_session\": true\n}"
            },
            "url": {
              "raw": "{{baseUrl}}/ussd/gateway/flares",
              "host": ["{{baseUrl}}"],
              "path": ["ussd", "gateway", "flares"]
            },
            "description": "Test Flares gateway format adapter"
          },
          "response": []
        }
      ]
    }
  ],
  "event": [
    {
      "listen": "prerequest",
      "script": {
        "type": "text/javascript",
        "exec": [
          "// Global pre-request script",
          "console.log('Request URL:', pm.request.url.toString());",
          "console.log('Request Method:', pm.request.method);",
          "",
          "// Add timestamp to all requests",
          "pm.request.headers.add({",
          "    key: 'X-Request-Time',",
          "    value: new Date().toISOString()",
          "});",
          "",
          "// Add API key if configured",
          "if (pm.environment.get('apiKey')) {",
          "    pm.request.headers.add({",
          "        key: 'X-API-Key',",
          "        value: pm.environment.get('apiKey')",
          "    });",
          "}"
        ]
      }
    },
    {
      "listen": "test",
      "script": {
        "type": "text/javascript",
        "exec": [
          "// Global test script",
          "const responseTime = pm.response.responseTime;",
          "",
          "// Log response time",
          "console.log('Response time:', responseTime + 'ms');",
          "",
          "// Check response time performance",
          "pm.test('Response time is less than 500ms', function () {",
          "    pm.expect(responseTime).to.be.below(500);",
          "});",
          "",
          "// Check for valid JSON response",
          "pm.test('Response is valid JSON', function () {",
          "    pm.response.to.be.json;",
          "});",
          "",
          "// Store response time for analytics",
          "if (pm.collectionVariables.has('totalResponseTime')) {",
          "    const total = pm.collectionVariables.get('totalResponseTime');",
          "    pm.collectionVariables.set('totalResponseTime', total + responseTime);",
          "} else {",
          "    pm.collectionVariables.set('totalResponseTime', responseTime);",
          "}",
          "",
          "// Increment request counter",
          "if (pm.collectionVariables.has('requestCount')) {",
          "    const count = pm.collectionVariables.get('requestCount');",
          "    pm.collectionVariables.set('requestCount', count + 1);",
          "} else {",
          "    pm.collectionVariables.set('requestCount', 1);",
          "}"
        ]
      }
    }
  ]
}
