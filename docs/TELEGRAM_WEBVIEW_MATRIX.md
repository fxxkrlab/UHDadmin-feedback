# Telegram WebView Compatibility Matrix

Round-9 P1-2: Cross-client Telegram WebView compatibility documentation.

## Platform Matrix

| Platform | initData Available | TG UID Access | Login Path | Recommended Open Method |
|----------|-------------------|---------------|------------|------------------------|
| iOS Telegram | Yes (WebApp) | Via initData | Auto (initData verify) | Menu Button / WebApp Button |
| Android Telegram | Yes (WebApp) | Via initData | Auto (initData verify) | Menu Button / WebApp Button |
| macOS Telegram | Partial | Limited | Password Login + TG Verify | Bot "Get TG Verification" button |
| Windows Telegram | Partial | Limited | Password Login + TG Verify | Bot "Get TG Verification" button |
| Browser (Direct) | No | None | Password Login | Direct URL (no TG binding) |

## initData Availability

### Mobile Clients (iOS/Android)
- **Full initData**: Available when opened via:
  - Bot menu button (MenuButton)
  - Inline `web_app` button
  - `/start` with deep link
- **Contains**: `user.id`, `user.username`, `user.first_name`, `auth_date`, `hash`
- **Verification**: Server-side HMAC validation against bot token

### Desktop Clients (macOS/Windows)
- **Partial initData**: May not trigger WebApp context properly
- **Deep links**: `/start verify_tg` may not activate in desktop clients
- **Workaround**: Use "Get TG Verification" callback button in bot menu

### Browser (Direct URL)
- **No initData**: Telegram context not available
- **Login**: Standard username/password only
- **TG Binding**: Not possible without Telegram client

## Login Paths

### Path 1: initData Auto-Login (Mobile)
```
User opens MiniApp via Telegram Bot
  -> window.Telegram.WebApp.initData available
  -> POST /auth/telegram/miniapp/exchange { init_data: "..." }
  -> Server validates initData hash
  -> Returns JWT token + user info
  -> Auto-login complete
```

### Path 2: Password Login + TG Auto-Bind (Mobile)
```
User opens MiniApp via Telegram Bot (no initData or failed exchange)
  -> User enters username/password
  -> POST /auth/telegram/miniapp/login { username, password, tg_uid? }
  -> If tg_uid from initData available: auto-bind TG account
  -> Returns JWT token
  -> Login complete with TG binding
```

### Path 3: Password Login + Manual TG Verify (Desktop)
```
User opens MiniApp via Telegram Bot (desktop)
  -> initData not available or unreliable
  -> User clicks "Get TG Verification" in bot menu
  -> Bot generates signed TG UID: tg_uid:timestamp:signature
  -> User opens MiniApp with ?tg_verify=... parameter
  -> User enters password to login
  -> POST /auth/telegram/miniapp/login with signed tg_verify
  -> Server validates signature and timestamp (5 min TTL)
  -> Returns JWT token with TG binding
```

### Path 4: Browser-Only Login (No TG)
```
User opens MiniApp URL directly in browser
  -> No Telegram context
  -> User enters username/password
  -> POST /auth/login (standard portal login)
  -> Returns JWT token
  -> Login complete (no TG binding possible)
```

## Known Limitations

### iOS Telegram
- WebApp API fully supported
- `openTelegramLink()` works for deep links
- Back button may close WebApp unexpectedly

### Android Telegram
- WebApp API fully supported
- `openTelegramLink()` works for deep links
- Hardware back button may close WebApp

### macOS Telegram
- WebApp API partially supported
- Deep links from `openTelegramLink()` may not trigger `/start` handler
- Workaround: Use callback buttons instead of deep links
- Bot must provide "Get TG Verification" button

### Windows Telegram
- Similar limitations to macOS
- WebApp support varies by version
- Recommend callback button approach

### Browser
- No Telegram-specific features
- Standard web authentication only
- Cannot access Telegram user info

## Implementation Notes

### Bot Menu Structure (handlers.py)
```python
def get_main_keyboard():
    builder = InlineKeyboardBuilder()
    builder.button(text="View Account", callback_data="view_account")
    builder.button(text="Daily Check-in", callback_data="checkin")
    builder.button(text="Open edgea App", web_app=WebAppInfo(url=MINIAPP_URL))
    # Desktop fallback for TG verification
    builder.button(text="Get TG Verification", callback_data="get_tg_verify")
    builder.adjust(2, 1, 1)
    return builder.as_markup()
```

### Signed TG UID Format
```
{tg_uid}:{timestamp}:{signature}
```
- `tg_uid`: Telegram user ID
- `timestamp`: Unix timestamp (seconds)
- `signature`: HMAC-SHA256(tg_uid:timestamp, SECRET_KEY)[:16]
- TTL: 5 minutes

### MiniApp initData Detection
```javascript
// Check if running inside Telegram WebApp
const tg = window.Telegram?.WebApp
const initData = tg?.initData
const tgUid = tg?.initDataUnsafe?.user?.id

if (initData) {
  // Use initData exchange
  await exchangeInitData(initData)
} else if (route.query.tg_verify) {
  // Use signed TG verification
  await loginWithTgVerify(route.query.tg_verify)
} else {
  // Show password login form
}
```

## Testing Checklist

- [ ] iOS Telegram: WebApp button opens MiniApp, initData available
- [ ] Android Telegram: WebApp button opens MiniApp, initData available
- [ ] macOS Telegram: "Get TG Verification" button works
- [ ] Windows Telegram: "Get TG Verification" button works
- [ ] Browser: Password login works without TG features
- [ ] Signed TG UID expires after 5 minutes
- [ ] Auto-bind works on first password login with TG context

## API Endpoints Summary

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/auth/telegram/miniapp/exchange` | POST | initData | Exchange initData for JWT |
| `/auth/telegram/miniapp/login` | POST | None | Password login with optional TG binding |
| `/auth/telegram/miniapp/restricted-signup` | POST | None | Restricted user registration (seat=0) |
| `/auth/login` | POST | None | Standard portal login (no TG) |

## References

- [Telegram WebApp Documentation](https://core.telegram.org/bots/webapps)
- [Bot API: Web Apps](https://core.telegram.org/bots/api#webapp)
- [initData Validation](https://core.telegram.org/bots/webapps#validating-data-received-via-the-edgea-app)
