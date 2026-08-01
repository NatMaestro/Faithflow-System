# Subscription Control Guide

## Overview
This guide explains how to manually control subscription checks for specific tenants (churches) in the code.

## Method 1: Using Django Shell (Recommended for Manual Control)

### Enable Subscription Bypass for a Specific Church

```python
# Open Django shell
python manage.py shell

# Import the Church model
from apps.churches.models import Church

# Find the church by subdomain or name
church = Church.objects.get(subdomain='your-church-subdomain')
# OR
church = Church.objects.get(name='Your Church Name')

# Enable bypass
church.bypass_subscription_check = True
church.save()

print(f"Subscription check disabled for: {church.name}")
```

### Disable Subscription Bypass (Re-enable Checks)

```python
# In Django shell
from apps.churches.models import Church

church = Church.objects.get(subdomain='your-church-subdomain')
church.bypass_subscription_check = False
church.save()

print(f"Subscription check re-enabled for: {church.name}")
```

### Check Current Bypass Status

```python
from apps.churches.models import Church

church = Church.objects.get(subdomain='your-church-subdomain')
print(f"Bypass enabled: {church.bypass_subscription_check}")
```

## Method 2: Using Django Admin (If Admin is Set Up)

1. Go to Django admin: `http://localhost:8000/admin/`
2. Navigate to **Churches** → **Churches**
3. Find and click on the church you want to modify
4. Check/uncheck the **Bypass subscription check** field
5. Save

## Method 3: Direct Database Update (Advanced)

```python
# In Django shell
from apps.churches.models import Church
from django.db import connection

# Update directly via SQL (if needed)
church_id = 1  # Replace with actual church ID
Church.objects.filter(id=church_id).update(bypass_subscription_check=True)
```

## Method 4: Bulk Operations

### Disable Subscription Checks for Multiple Churches

```python
from apps.churches.models import Church

# Example: Disable for all churches with 'test' in subdomain
Church.objects.filter(subdomain__icontains='test').update(bypass_subscription_check=True)

# Example: Disable for specific churches by ID
Church.objects.filter(id__in=[1, 2, 3]).update(bypass_subscription_check=True)

# Example: Disable for churches with specific plan
Church.objects.filter(plan='enterprise').update(bypass_subscription_check=True)
```

### Re-enable Checks for Multiple Churches

```python
from apps.churches.models import Church

# Re-enable for all churches
Church.objects.all().update(bypass_subscription_check=False)

# Re-enable for specific subset
Church.objects.filter(subdomain__icontains='test').update(bypass_subscription_check=False)
```

## Method 5: Using Management Command (Future Implementation)

Create a management command for easier control:

```python
# Create: apps/churches/management/commands/toggle_subscription_bypass.py
from django.core.management.base import BaseCommand
from apps.churches.models import Church

class Command(BaseCommand):
    help = 'Toggle subscription bypass for a church'

    def add_arguments(self, parser):
        parser.add_argument('subdomain', type=str, help='Church subdomain')
        parser.add_argument('--enable', action='store_true', help='Enable bypass')
        parser.add_argument('--disable', action='store_true', help='Disable bypass')

    def handle(self, *args, **options):
        try:
            church = Church.objects.get(subdomain=options['subdomain'])
            
            if options['enable']:
                church.bypass_subscription_check = True
                church.save()
                self.stdout.write(self.style.SUCCESS(f'Subscription bypass ENABLED for {church.name}'))
            elif options['disable']:
                church.bypass_subscription_check = False
                church.save()
                self.stdout.write(self.style.SUCCESS(f'Subscription bypass DISABLED for {church.name}'))
            else:
                status = 'ENABLED' if church.bypass_subscription_check else 'DISABLED'
                self.stdout.write(f'Current status for {church.name}: {status}')
        except Church.DoesNotExist:
            self.stdout.write(self.style.ERROR(f'Church with subdomain "{options["subdomain"]}" not found'))
```

Usage:
```bash
# Enable bypass
python manage.py toggle_subscription_bypass your-church-subdomain --enable

# Disable bypass
python manage.py toggle_subscription_bypass your-church-subdomain --disable

# Check status
python manage.py toggle_subscription_bypass your-church-subdomain
```

## Method 6: Programmatic Control in Code

### In Views or Services

```python
from apps.churches.models import Church

# Enable bypass programmatically
def enable_subscription_bypass(church_id):
    church = Church.objects.get(id=church_id)
    church.bypass_subscription_check = True
    church.save()
    return church

# Disable bypass programmatically
def disable_subscription_bypass(church_id):
    church = Church.objects.get(id=church_id)
    church.bypass_subscription_check = False
    church.save()
    return church
```

## How It Works

1. **Middleware Check**: The `SubscriptionMiddleware` checks the `bypass_subscription_check` flag first
2. **If Enabled**: All subscription/trial expiration checks are skipped
3. **If Disabled**: Normal subscription checks apply (trial expiration, subscription expiration, etc.)

## Important Notes

⚠️ **Security Warning**: 
- Only use bypass for testing, development, or special cases
- Always document why a church has bypass enabled
- Consider adding an expiration date for bypass flags in the future

✅ **Best Practices**:
- Use bypass sparingly
- Keep a log of which churches have bypass enabled and why
- Re-enable checks once testing/development is complete
- Use this for internal testing churches, not production

## Future: Superadmin Dashboard Control

When implementing the superadmin dashboard, you can add:

1. **Church Management Page**: List all churches with bypass status
2. **Toggle Button**: Enable/disable bypass with one click
3. **Audit Log**: Track who enabled/disabled bypass and when
4. **Bulk Operations**: Enable/disable for multiple churches at once

Example UI:
```
Church Name          | Subdomain        | Bypass Status | Actions
--------------------|------------------|---------------|----------
Grace Community     | gracecommunity   | ❌ Disabled   | [Enable]
Test Church         | testchurch      | ✅ Enabled    | [Disable]
```

## Database Migration

After adding the `bypass_subscription_check` field, run:

```bash
python manage.py makemigrations churches
python manage.py migrate
```

## Verification

To verify bypass is working:

1. **Enable bypass for a test church**
2. **Set trial_end_date to past date**
3. **Make API request** - should succeed (bypassed)
4. **Disable bypass**
5. **Make API request** - should fail with 403 (subscription expired)


