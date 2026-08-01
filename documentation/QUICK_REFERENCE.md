# Quick Reference: Subscription Control

## 🚀 Quick Start

### Enable Subscription Bypass (Disable Checks)
```bash
cd faithflow-backend
python manage.py toggle_subscription_bypass your-church-subdomain --enable
```

### Disable Subscription Bypass (Re-enable Checks)
```bash
python manage.py toggle_subscription_bypass your-church-subdomain --disable
```

### Check Current Status
```bash
python manage.py toggle_subscription_bypass your-church-subdomain
```

### List All Churches with Bypass Enabled
```bash
python manage.py toggle_subscription_bypass --list-all
```

## 📝 Django Shell Examples

### Enable Bypass
```python
python manage.py shell
>>> from apps.churches.models import Church
>>> church = Church.objects.get(subdomain='your-church-subdomain')
>>> church.bypass_subscription_check = True
>>> church.save()
>>> print(f"Bypass enabled for: {church.name}")
```

### Disable Bypass
```python
>>> church.bypass_subscription_check = False
>>> church.save()
```

### Check Status
```python
>>> print(f"Bypass: {church.bypass_subscription_check}")
```

## 🔍 How It Works

1. **Field Added**: `bypass_subscription_check` (Boolean) in Church model
2. **Middleware Check**: SubscriptionMiddleware checks this flag first
3. **If True**: All subscription/trial checks are skipped
4. **If False**: Normal subscription checks apply

## ⚠️ Important Notes

- Use bypass only for testing/development or special cases
- Always document why a church has bypass enabled
- Re-enable checks when done testing
- This bypasses ALL subscription checks (trial, subscription, grace period)

## 🎯 Use Cases

- **Testing**: Test features without subscription restrictions
- **Development**: Develop without worrying about trial expiration
- **Special Cases**: Partner churches, beta testers, etc.
- **Emergency**: Temporarily restore access if subscription system has issues


