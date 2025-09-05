# Partner Embed Integration Guide (v1)

This guide describes how to integrate and embed our web application into your mobile app (via WebView) or website (via iframe). The embedded app is served from `https://pixels.persona-ai.ai/partner` and requires JWT-based authentication.

## Overview

Our embeddable web application provides a seamless integration experience for partners. The app:
- Authenticates via HMAC-signed JWT tokens (HS256)
- Communicates via standard JavaScript events
- Supports iOS, Android, and web platforms
- Includes comprehensive error handling and lifecycle management

## URL Format

The embedded app URL follows this structure:

```
https://pixels.persona-ai.ai/partner?token=<JWT>
```

### Required Parameters
- `token` - A valid JWT signed with your shared secret (see Token Specification below)

### Optional Parameters
- `lang` - Language code (e.g., `en`, `es`, `fr`). Default: `en`
- `theme` - UI theme (`light` or `dark`). Default: `light`
- `debug` - Enable debug mode (`true` or `false`). Default: `false`

Example:
```
https://pixels.persona-ai.ai/partner?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...&lang=es&theme=dark
```

## Token Specification

### Algorithm
Tokens must be signed using **HMAC SHA-256 (HS256)** with your shared secret key.

### JWT Structure

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Required Claims

| Claim | Type | Description | Example |
|-------|------|-------------|---------|
| `iss` | string | Issuer - Your partner identifier | `"partner_acme_corp"` |
| `sub` | string | Subject - Unique user identifier | `"user_12345"` |
| `aud` | string | Audience - Must be `"pixels.persona-ai.ai"` | `"pixels.persona-ai.ai"` |
| `iat` | number | Issued at - Unix timestamp | `1704067200` |
| `exp` | number | Expiration - Unix timestamp | `1704070800` |
| `nonce` | string | Unique nonce for replay protection | `"a1b2c3d4e5f6"` |

### Optional Claims

| Claim | Type | Description | Example |
|-------|------|-------------|---------|
| `meta` | object | Additional user metadata | `{"name": "John Doe", "tier": "premium"}` |

### Example Token Payload

```json
{
  "iss": "partner_acme_corp",
  "sub": "user_12345",
  "aud": "pixels.persona-ai.ai",
  "iat": 1704067200,
  "exp": 1704070800,
  "nonce": "a1b2c3d4e5f6",
  "meta": {
    "name": "John Doe",
    "email": "john@example.com",
    "tier": "premium"
  }
}
```

### Token Generation Example (Node.js)

```typescript
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

function generateEmbedToken(userId: string, sharedSecret: string): string {
  const now = Math.floor(Date.now() / 1000);
  
  const payload = {
    iss: 'partner_acme_corp',
    sub: userId,
    aud: 'pixels.persona-ai.ai',
    iat: now,
    exp: now + 3600, // 1 hour TTL
    nonce: crypto.randomBytes(16).toString('hex'),
    meta: {
      name: 'John Doe',
      email: 'john@example.com'
    }
  };
  
  return jwt.sign(payload, sharedSecret, { algorithm: 'HS256' });
}
```

## Validation Rules

The embedded app validates tokens according to these rules:

1. **Signature Verification**: Token must be signed with the correct shared secret
2. **Expiration Check**: Current time must be before `exp` claim
3. **Issued Time Check**: Current time must be after `iat` claim
4. **Audience Validation**: `aud` must equal `"pixels.persona-ai.ai"`
5. **Issuer Validation**: `iss` must match your registered partner identifier
6. **Nonce Uniqueness**: The `nonce` must not have been used within the token's lifetime
7. **Time Window**: Token age (`current_time - iat`) must not exceed 24 hours

### Validation Error Responses

If token validation fails, the app will emit a `session.ended` event with error code `auth_failed` and display an appropriate error message.

## Event API

The embedded app communicates with the parent container via JavaScript events. All events are dispatched on the `window` object.

### Event Structure

```typescript
interface EmbedEvent {
  type: string;
  timestamp: number;
  data?: any;
}
```

### Lifecycle Events

#### `embed.ready`

Emitted when the embedded app has successfully loaded and authenticated.

```typescript
{
  type: 'embed.ready',
  timestamp: 1704067200000,
  data: {
    version: '1.0.0',
    userId: 'user_12345',
    features: ['chat', 'voice', 'video']
  }
}
```

#### `session.started`

Emitted when a user session has been successfully established.

```typescript
{
  type: 'session.started',
  timestamp: 1704067210000,
  data: {
    sessionId: 'session_abc123',
    userId: 'user_12345'
  }
}
```

#### `session.ended`

Emitted when the session ends, either normally or due to an error.

```typescript
{
  type: 'session.ended',
  timestamp: 1704067800000,
  data: {
    sessionId: 'session_abc123',
    reason: 'normal_exit', // See error codes below
    message: 'Session ended by user'
  }
}
```

### Error Codes

The `session.ended` event includes standardized error codes:

| Code | Description |
|------|-------------|
| `normal_exit` | User ended session normally |
| `network_loss` | Network connection lost |
| `auth_failed` | Authentication failure or token expired |
| `server_error` | Server-side error occurred |
| `timeout` | Session timed out due to inactivity |
| `unknown` | Unknown error occurred |

### Listening to Events

```javascript
window.addEventListener('message', (event) => {
  // Validate origin
  if (event.origin !== 'https://pixels.persona-ai.ai') {
    return;
  }
  
  // Handle events
  switch (event.data.type) {
    case 'embed.ready':
      console.log('Embed ready:', event.data);
      break;
    case 'session.started':
      console.log('Session started:', event.data);
      break;
    case 'session.ended':
      console.log('Session ended:', event.data);
      handleSessionEnd(event.data.data.reason);
      break;
  }
});
```

## Platform Integration Examples

### iOS (WKWebView)

```swift
import WebKit
import UIKit

class EmbedViewController: UIViewController, WKScriptMessageHandler {
    var webView: WKWebView!
    let sharedSecret = "YOUR_SHARED_SECRET"
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupWebView()
        loadEmbed()
    }
    
    func setupWebView() {
        let config = WKWebViewConfiguration()
        
        // Add message handler for events
        let contentController = WKUserContentController()
        
        // Inject JavaScript to capture events
        let script = """
            window.addEventListener('message', function(e) {
                if (e.origin === 'https://pixels.persona-ai.ai') {
                    window.webkit.messageHandlers.embedEvent.postMessage(e.data);
                }
            });
        """
        
        let userScript = WKUserScript(
            source: script,
            injectionTime: .atDocumentEnd,
            forMainFrameOnly: false
        )
        
        contentController.addUserScript(userScript)
        contentController.add(self, name: "embedEvent")
        config.userContentController = contentController
        
        webView = WKWebView(frame: view.bounds, configuration: config)
        webView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        view.addSubview(webView)
    }
    
    func loadEmbed() {
        let token = generateToken(userId: "user_12345")
        let urlString = "https://pixels.persona-ai.ai/partner?token=\(token)"
        
        if let url = URL(string: urlString) {
            let request = URLRequest(url: url)
            webView.load(request)
        }
    }
    
    // Handle messages from JavaScript
    func userContentController(_ userContentController: WKUserContentController,
                              didReceive message: WKScriptMessage) {
        guard let dict = message.body as? [String: Any],
              let type = dict["type"] as? String else {
            return
        }
        
        switch type {
        case "embed.ready":
            print("Embed ready")
        case "session.started":
            print("Session started")
        case "session.ended":
            if let data = dict["data"] as? [String: Any],
               let reason = data["reason"] as? String {
                handleSessionEnd(reason: reason)
            }
        default:
            break
        }
    }
    
    func handleSessionEnd(reason: String) {
        switch reason {
        case "auth_failed":
            // Refresh token and reload
            loadEmbed()
        case "network_loss":
            // Show network error
            showAlert(title: "Network Error", message: "Connection lost")
        default:
            // Handle other cases
            break
        }
    }
}
```

### Android (WebView)

```kotlin
import android.webkit.WebView
import android.webkit.WebViewClient
import android.webkit.JavascriptInterface
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import org.json.JSONObject

class EmbedActivity : AppCompatActivity() {
    private lateinit var webView: WebView
    private val sharedSecret = "YOUR_SHARED_SECRET"
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_embed)
        
        setupWebView()
        loadEmbed()
    }
    
    private fun setupWebView() {
        webView = findViewById(R.id.webView)
        
        webView.settings.apply {
            javaScriptEnabled = true
            domStorageEnabled = true
        }
        
        // Add JavaScript interface
        webView.addJavascriptInterface(WebAppInterface(), "Android")
        
        webView.webViewClient = object : WebViewClient() {
            override fun onPageFinished(view: WebView?, url: String?) {
                super.onPageFinished(view, url)
                injectEventListener()
            }
        }
    }
    
    private fun loadEmbed() {
        val token = generateToken("user_12345")
        val url = "https://pixels.persona-ai.ai/partner?token=$token"
        webView.loadUrl(url)
    }
    
    private fun injectEventListener() {
        val javascript = """
            window.addEventListener('message', function(e) {
                if (e.origin === 'https://pixels.persona-ai.ai') {
                    Android.handleEvent(JSON.stringify(e.data));
                }
            });
        """
        webView.evaluateJavascript(javascript, null)
    }
    
    inner class WebAppInterface {
        @JavascriptInterface
        fun handleEvent(eventJson: String) {
            val event = JSONObject(eventJson)
            val type = event.getString("type")
            
            runOnUiThread {
                when (type) {
                    "embed.ready" -> {
                        // Handle embed ready
                        println("Embed ready")
                    }
                    "session.started" -> {
                        // Handle session started
                        println("Session started")
                    }
                    "session.ended" -> {
                        val data = event.getJSONObject("data")
                        val reason = data.getString("reason")
                        handleSessionEnd(reason)
                    }
                }
            }
        }
    }
    
    private fun handleSessionEnd(reason: String) {
        when (reason) {
            "auth_failed" -> {
                // Refresh token and reload
                loadEmbed()
            }
            "network_loss" -> {
                // Show network error
                showError("Network connection lost")
            }
            else -> {
                // Handle other cases
            }
        }
    }
}
```

### Web (iframe)

```html
<!DOCTYPE html>
<html>
<head>
    <title>Partner Embed Example</title>
    <style>
        #embed-container {
            width: 100%;
            height: 600px;
            border: 1px solid #ccc;
            border-radius: 8px;
            overflow: hidden;
        }
        
        #embed-iframe {
            width: 100%;
            height: 100%;
            border: none;
        }
        
        .status {
            padding: 10px;
            margin: 10px 0;
            border-radius: 4px;
        }
        
        .status.ready { background: #d4edda; color: #155724; }
        .status.error { background: #f8d7da; color: #721c24; }
    </style>
</head>
<body>
    <h1>Embedded App Demo</h1>
    <div id="status"></div>
    <div id="embed-container">
        <iframe id="embed-iframe"></iframe>
    </div>
    
    <script>
        const EMBED_DOMAIN = 'https://pixels.persona-ai.ai';
        const SHARED_SECRET = 'YOUR_SHARED_SECRET';
        
        // Generate token (this should be done server-side in production)
        async function generateToken(userId) {
            // In production, make an API call to your backend
            const response = await fetch('/api/generate-embed-token', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ userId })
            });
            const data = await response.json();
            return data.token;
        }
        
        // Initialize embed
        async function initEmbed() {
            const userId = 'user_12345';
            const token = await generateToken(userId);
            
            const iframe = document.getElementById('embed-iframe');
            iframe.src = `${EMBED_DOMAIN}/partner?token=${token}`;
        }
        
        // Listen for events
        window.addEventListener('message', (event) => {
            // Validate origin
            if (event.origin !== EMBED_DOMAIN) {
                return;
            }
            
            const statusDiv = document.getElementById('status');
            
            switch (event.data.type) {
                case 'embed.ready':
                    console.log('Embed ready:', event.data);
                    statusDiv.className = 'status ready';
                    statusDiv.textContent = 'Embed loaded successfully';
                    break;
                    
                case 'session.started':
                    console.log('Session started:', event.data);
                    statusDiv.className = 'status ready';
                    statusDiv.textContent = `Session started: ${event.data.data.sessionId}`;
                    break;
                    
                case 'session.ended':
                    console.log('Session ended:', event.data);
                    const reason = event.data.data.reason;
                    
                    if (reason === 'normal_exit') {
                        statusDiv.className = 'status';
                        statusDiv.textContent = 'Session ended normally';
                    } else {
                        statusDiv.className = 'status error';
                        statusDiv.textContent = `Session ended: ${reason}`;
                        
                        // Handle specific error cases
                        if (reason === 'auth_failed') {
                            // Refresh token and reload
                            setTimeout(() => initEmbed(), 2000);
                        }
                    }
                    break;
            }
        });
        
        // Start loading
        initEmbed();
    </script>
</body>
</html>
```

## Security Recommendations

### Token Security

1. **Short TTL**: Use short token lifetimes (recommended: 1 hour maximum)
2. **Secure Storage**: Never store shared secrets in client-side code
3. **Server-Side Generation**: Always generate tokens on your backend server
4. **HTTPS Only**: Always use HTTPS for all communications
5. **Key Rotation**: Implement regular rotation of shared secrets (recommended: every 90 days)

### Implementation Best Practices

```typescript
// ✅ Good: Server-side token generation
app.post('/api/generate-embed-token', authenticate, async (req, res) => {
  const token = generateSecureToken(req.user.id, process.env.SHARED_SECRET);
  res.json({ token });
});

// ❌ Bad: Client-side token generation
// Never expose your shared secret in client code!
```

### Content Security Policy

For web integrations, implement appropriate CSP headers:

```http
Content-Security-Policy: frame-src https://pixels.persona-ai.ai; 
                         connect-src https://pixels.persona-ai.ai;
```

### Nonce Management

- Use cryptographically secure random values for nonces
- Store used nonces with their expiration times
- Reject tokens with previously used nonces
- Clean up expired nonces regularly

```typescript
// Example nonce generation
import crypto from 'crypto';

function generateNonce(): string {
  return crypto.randomBytes(16).toString('hex');
}

// Example nonce validation
const usedNonces = new Map<string, number>();

function validateNonce(nonce: string, exp: number): boolean {
  const now = Date.now() / 1000;
  
  // Check if nonce was already used
  if (usedNonces.has(nonce)) {
    return false;
  }
  
  // Store nonce with expiration
  usedNonces.set(nonce, exp);
  
  // Clean up expired nonces
  for (const [n, e] of usedNonces.entries()) {
    if (e < now) {
      usedNonces.delete(n);
    }
  }
  
  return true;
}
```

## Versioning

### API Version

Current API version: **v1**

The API version is included in the `embed.ready` event data:

```json
{
  "type": "embed.ready",
  "data": {
    "version": "1.0.0"
  }
}
```

### Semantic Versioning

We follow semantic versioning (semver) for our embed API:

- **Major version** (1.x.x): Breaking changes
- **Minor version** (x.1.x): New features, backward compatible
- **Patch version** (x.x.1): Bug fixes, backward compatible

### Version Compatibility

| Embed Version | Minimum Token Version | Notes |
|--------------|----------------------|--------|
| 1.0.x | 1.0 | Initial release |
| 1.1.x | 1.0 | Added optional metadata fields |
| 2.0.x | 2.0 | Breaking changes (future) |

### Deprecation Policy

- Deprecated features will be marked with warnings for at least 90 days
- Breaking changes will be announced at least 30 days in advance
- Legacy versions will be supported for at least 6 months after deprecation

## Test Checklist

Before going to production, ensure you have tested:

### Authentication
- [ ] Token generation with all required claims
- [ ] Token expiration handling
- [ ] Invalid token rejection
- [ ] Nonce uniqueness validation
- [ ] Signature verification

### Integration
- [ ] Embed loads successfully in WebView/iframe
- [ ] All lifecycle events are received
- [ ] Event data is correctly parsed
- [ ] Cross-origin communication works

### Error Handling
- [ ] Network disconnection recovery
- [ ] Token expiration refresh flow
- [ ] Server error handling
- [ ] Timeout scenarios
- [ ] Invalid origin rejection

### Platform-Specific
- [ ] iOS: WKWebView integration and event handling
- [ ] Android: WebView JavaScript interface
- [ ] Web: iframe CSP and sandboxing

### Performance
- [ ] Load time under 3 seconds
- [ ] Smooth scrolling and interactions
- [ ] Memory usage within limits
- [ ] No memory leaks during long sessions

### Security
- [ ] HTTPS enforcement
- [ ] Token transmitted securely
- [ ] No sensitive data in URLs
- [ ] CSP headers configured
- [ ] Origin validation

## Troubleshooting

### Common Issues

#### Token Validation Failures

**Symptom**: `auth_failed` event immediately after loading

**Solutions**:
- Verify token is properly signed with shared secret
- Check token expiration time
- Ensure all required claims are present
- Validate audience claim matches exactly

#### Events Not Received

**Symptom**: No events firing in parent container

**Solutions**:
- Verify origin validation in event listener
- Check browser console for errors
- Ensure JavaScript is enabled
- Verify correct event listener setup

#### Session Timeouts

**Symptom**: Unexpected `session.ended` with `timeout` reason

**Solutions**:
- Implement keep-alive mechanism
- Increase token TTL if appropriate
- Handle timeout gracefully with auto-reconnect

---

*Last Updated: January 2025*  
*Version: 1.0.0*
