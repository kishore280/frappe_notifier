# Frappe Notifier (Modified — kishore280/frappe_notifier)

Fork of [tridz-dev/frappe_notifier](https://github.com/tridz-dev/frappe_notifier) with the following changes on branch `fix-android-background-notification-delivery`.

---

## What Changed

### 1. Firebase Admin SDK — proper service account auth (`utils/firebase.py`)

The original code passed the Firebase web config (`apiKey`, `authDomain`, etc.) as `options=` to `initialize_app()`. The Admin SDK ignores that argument for auth — it is a client-side web config, not server credentials.

**Fix:** `firebase_config` field in Frappe Notifier Settings must now contain the **service account JSON** (downloaded from Firebase Console → Project Settings → Service accounts → Generate new private key). It is read and passed as `credentials.Certificate()`.

```python
cred = credentials.Certificate(service_account_info)
initialize_app(cred)
```

Benefits:
- No dependency on `GOOGLE_APPLICATION_CREDENTIALS` env var or files on disk
- Each Frappe site stores its own service account JSON → multi-site setups can use different Firebase projects

### 2. Android background delivery — FCM data payload + high priority (`api/send_notification.py`)

Android kills notification-only FCM messages when the app is backgrounded or in Doze mode. The fix:

- **`AndroidConfig(priority="high")`** — bypasses Doze, wakes the device immediately
- **Data payload** (`data=`) carries `title`, `body`, `click_action`, `base_url`, and any caller-supplied fields so the Flutter client can reconstruct the notification even when the app is not in the foreground
- **All values cast to `str`** — FCM HTTP v1 rejects non-string values in the data map:
  ```python
  data_dict = {k: str(v) for k, v in data_dict.items()}
  ```
- **`extra_data` param added** to `send_notification()` so callers can inject arbitrary string fields into the FCM data payload without touching the function signature for every use case

### 3. Notification Log hook — direct FCM delivery (`controllers/system_notifications.py`)

The original `send_notification_to_user` helper routed through `frappe.push_notification.PushNotification` (the Frappe relay), which uses a different token store and was not wiring up the correct project/site filters.

**Fix:** Replaced with direct calls to `get_user_tokens` + `send_notification` from `frappe_notifier`:

```python
tokens = get_user_tokens(
    project_name="grozfy",
    site_name=frappe.local.site,
    user_id=doc.for_user,
)
send_notification(
    tokens=tokens,
    title=title,
    body=body,
    click_action=None,          # not a URL — keep out of WebpushFCMOptions.link
    deactivate_invalid_tokens=True,
    extra_data={
        "document_type": ...,
        "document_name": ...,
        "notification_type": ...,
        "notification_log_name": ...,
        "click_action": "FLUTTER_NOTIFICATION_CLICK",  # Flutter reads from data payload
        "base_url": frappe.utils.get_url(),
    },
)
```

Key points:
- `click_action` is passed as `None` to `send_notification` so it never reaches `WebpushFCMOptions.link` (which expects a URL, not a Flutter action string)
- `FLUTTER_NOTIFICATION_CLICK` lives only in the FCM data payload where the Flutter SDK reads it
- `base_url` included so the client knows which site to open

### 4. Hooks — no change needed (`hooks.py`)

The `Notification Log → after_insert` hook was already wired correctly:

```python
doc_events = {
    "Notification Log": {
        "after_insert": "frappe_notifier.controllers.system_notifications.on_insert"
    }
}
```

---

## Setup

### Firebase Config field

Paste the **service account JSON** (not the web app config) into **Frappe Notifier Settings → Firebase Config**:

```json
{
  "type": "service_account",
  "project_id": "your-project-id",
  "private_key_id": "...",
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----\n",
  "client_email": "firebase-adminsdk-xxx@your-project.iam.gserviceaccount.com",
  "client_id": "...",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

Everything else in the original [README.md](README.md) (installation, Vapid key, site config) applies unchanged.

---

## License

MIT
