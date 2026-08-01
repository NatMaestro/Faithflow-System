# Subscription System Walkthrough

## Overview

The subscription system manages church access to FaithFlows based on their plan (trial, basic, standard, premium, enterprise) and subscription status. It includes trial periods, grace periods, and access control.

---

## 1. Database Model (Backend)

### Church Model Fields (`apps/churches/models.py`)

#### Plan Types

```python
plan = models.CharField(
    choices=[
        ('trial', 'Trial'),        # Free 30-day trial
        ('basic', 'Basic'),        # $19/month
        ('standard', 'Standard'),  # $29/month
        ('premium', 'Premium'),   # $79/month
        ('enterprise', 'Enterprise'), # Custom pricing
    ],
    default='trial'
)
```

#### Subscription Status

```python
subscription_status = models.CharField(
    choices=[
        ('active', 'Active'),       # Can access system
        ('inactive', 'Inactive'),   # Not active
        ('suspended', 'Suspended'), # Temporarily blocked
        ('cancelled', 'Cancelled'),  # Permanently cancelled
    ],
    default='active'
)
```

#### Date Fields

- **`trial_started_at`**: When the trial began
- **`trial_end_date`**: When the trial expires (30 days from start)
- **`subscription_start_date`**: When paid subscription started
- **`subscription_end_date`**: When paid subscription expires

#### Control Fields

- **`grace_period_days`**: Days after expiration before blocking (default: 7)
- **`bypass_subscription_check`**: Boolean flag to bypass all checks (for testing)

---

## 2. Subscription Lifecycle

### A. Trial Start (New Church Registration)

**When**: Church completes onboarding
**Where**: `Complete.tsx` → `churchService.startTrial()`

**Process**:

1. User completes registration
2. Backend creates church with `plan='trial'`
3. Frontend calls `POST /api/v1/churches/{id}/start-trial/`
4. Backend sets:
   - `plan = 'trial'`
   - `subscription_status = 'active'`
   - `trial_started_at = now()`
   - `trial_end_date = now() + 30 days`

**Code Flow**:

```typescript
// Frontend: Complete.tsx
await churchService.startTrial(result.church.id);

// Backend: views.py -> start_trial()
church.plan = 'trial'
church.trial_started_at = now
church.trial_end_date = now + 30 days
```

---

### B. Subscription States

#### State 1: Active Trial

- **Plan**: `trial`
- **Status**: `active`
- **Dates**: `trial_end_date` is in the future
- **Access**: ✅ Full access
- **Frontend**: Shows "X days remaining" banner

#### State 2: Trial Expired (Grace Period)

- **Plan**: `trial`
- **Status**: `active`
- **Dates**: `trial_end_date` passed, but within grace period (7 days)
- **Access**: ✅ Still has access (grace period)
- **Frontend**: Shows warning banner "Trial expired, X days to upgrade"
- **Backend**: Returns `status: 'trial_expiring'`, `can_access: True`

#### State 3: Trial Fully Expired

- **Plan**: `trial`
- **Status**: `active` (but effectively expired)
- **Dates**: Past grace period end date
- **Access**: ❌ Blocked
- **Frontend**: Shows "Trial Expired" modal, blocks access
- **Backend**: Returns `status: 'trial_expired'`, `can_access: False`, HTTP 403

#### State 4: Active Subscription

- **Plan**: `basic|standard|premium|enterprise`
- **Status**: `active`
- **Dates**: `subscription_end_date` is in the future
- **Access**: ✅ Full access
- **Frontend**: Shows plan name, no warnings

#### State 5: Subscription Expired (Grace Period)

- **Plan**: `basic|standard|premium|enterprise`
- **Status**: `active`
- **Dates**: `subscription_end_date` passed, but within grace period
- **Access**: ✅ Still has access (grace period)
- **Frontend**: Shows warning banner "Subscription expired, X days to renew"

#### State 6: Subscription Fully Expired

- **Plan**: `basic|standard|premium|enterprise`
- **Status**: `active`
- **Dates**: Past grace period end date
- **Access**: ❌ Blocked
- **Backend**: Returns `status: 'subscription_expired'`, `can_access: False`, HTTP 403

#### State 7: Suspended/Cancelled

- **Status**: `suspended` or `cancelled`
- **Access**: ❌ Blocked (regardless of dates)
- **Backend**: Returns error immediately, no date checks

---

## 3. Access Control Flow

### Backend Middleware (`core/middleware/subscription.py`)

**Order of Checks**:

1. **Public Routes** → Skip check

   - `/api/v1/auth/register/`
   - `/api/v1/auth/login/`
   - `/api/v1/auth/refresh/`
   - `/health/`
   - `/api/docs/`

2. **Bypass Flag** → Allow access

   - If `bypass_subscription_check = True` → ✅ Access granted
   - If subdomain is `apostolic` or `apostolicchurch` → ✅ Access granted

3. **Subscription Status Check**:

   ```python
   # Check cancelled/suspended first
   if subscription_status in ['cancelled', 'suspended']:
       return 403 Forbidden

   # Check trial expiration
   if plan == 'trial' and trial_end_date:
       grace_end = trial_end_date + grace_period_days
       if now > grace_end:
           return 403 Forbidden  # Fully expired
       if now > trial_end_date:
           return 200 OK  # Grace period (warn but allow)

   # Check subscription expiration
   if subscription_end_date:
       grace_end = subscription_end_date + grace_period_days
       if now > grace_end:
           return 403 Forbidden  # Fully expired
       if now > subscription_end_date:
           return 200 OK  # Grace period (warn but allow)

   # Active
   return 200 OK
   ```

---

### Frontend Hook (`hooks/useSubscription.ts`)

**Process**:

1. Gets `currentChurch` from Redux state
2. Checks bypass flags (same as backend)
3. Calculates expiration dates
4. Determines status and access

**Returns**:

```typescript
{
  status: 'active' | 'trial_expiring' | 'trial_expired' |
          'subscription_expiring' | 'subscription_expired' |
          'suspended' | 'cancelled',
  daysRemaining: number | null,
  isExpired: boolean,
  canAccess: boolean,
  isTrial: boolean,
  isInGracePeriod: boolean,
  expirationDate: string | null,
  plan: string
}
```

---

### Frontend Guard (`components/subscription/SubscriptionGuard.tsx`)

**Process**:

1. Uses `useSubscription()` hook
2. If `canAccess = false` → Shows "Subscription Expired" modal
3. If `canAccess = true` → Shows children (normal app)

**Bypass Check**:

- Also checks bypass flags directly as safety measure
- If bypass active → Always show children

---

## 4. Plan Selection & Upgrade

### Landing Page → Onboarding

1. User clicks "Start Free Trial" on landing page
2. Goes through onboarding flow
3. On completion → Trial automatically starts (30 days)

### Upgrade Flow

1. User clicks "Upgrade Plan" button (from banner or guard)
2. Navigates to `/subscription/upgrade`
3. Selects plan (Basic, Standard, Premium)
4. Selects duration (Monthly or Yearly)
5. Clicks "Upgrade Now"
6. Frontend calls `POST /api/v1/churches/{id}/upgrade-subscription/`
7. Backend updates:
   ```python
   church.plan = selected_plan  # 'basic', 'standard', 'premium'
   church.subscription_status = 'active'
   church.subscription_start_date = now()
   church.subscription_end_date = now() + (30 days or 365 days)
   church.trial_end_date = None  # Clear trial
   ```

---

## 5. Status Determination Logic

### Backend (`models.py` → `get_subscription_status()`)

```python
def get_subscription_status(self):
    # 1. Check if cancelled/suspended
    if subscription_status in ['cancelled', 'suspended']:
        return subscription_status

    # 2. Check trial expiration
    if plan == 'trial' and trial_end_date:
        grace_end = trial_end_date + grace_period_days
        if now > grace_end:
            return 'trial_expired'  # Fully expired
        if now > trial_end_date:
            return 'trial_expiring'  # In grace period

    # 3. Check subscription expiration
    if subscription_end_date:
        grace_end = subscription_end_date + grace_period_days
        if now > grace_end:
            return 'subscription_expired'  # Fully expired
        if now > subscription_end_date:
            return 'subscription_expiring'  # In grace period

    # 4. Active
    return 'active'
```

### Frontend (`useSubscription.ts`)

Same logic, but in JavaScript:

1. Check bypass flags
2. Check cancelled/suspended
3. Check trial expiration (with grace period)
4. Check subscription expiration (with grace period)
5. Return 'active' if all checks pass

---

## 6. Bypass Mechanisms

### For Testing/Development

#### Method 1: Database Flag

```python
church.bypass_subscription_check = True
church.save()
```

#### Method 2: Subdomain Check (Hardcoded)

- Subdomain: `apostolic` or `apostolicchurch`
- Automatically bypasses all checks
- Works in both frontend and backend

#### Method 3: Management Command

```bash
python manage.py toggle_subscription_bypass apostolic --enable
```

---

## 7. API Endpoints

### Get Subscription Status

```
GET /api/v1/churches/{id}/subscription-status/
```

Returns: plan, status, dates, days remaining, can_access

### Start Trial

```
POST /api/v1/churches/{id}/start-trial/
```

Sets: trial_started_at, trial_end_date (30 days from now)

### Upgrade Subscription

```
POST /api/v1/churches/{id}/upgrade-subscription/
Body: {
    "plan": "basic|standard|premium|enterprise",
    "duration": "monthly|yearly"
}
```

Sets: plan, subscription_start_date, subscription_end_date

---

## 8. Visual Indicators

### SubscriptionBanner Component

- Shows when trial/subscription is expiring
- Displays days remaining
- "Upgrade Now" button → `/subscription/upgrade`

### SubscriptionGuard Component

- Blocks access when fully expired
- Shows "Subscription Expired" modal
- "Upgrade Plan" and "Logout" buttons

---

## 9. Key Points

1. **Trial vs Subscription**:

   - Trial uses `trial_end_date`
   - Paid plans use `subscription_end_date`
   - When upgrading, `trial_end_date` is cleared

2. **Grace Period**:

   - Default: 7 days after expiration
   - User can still access during grace period
   - Shows warnings but doesn't block

3. **Access Control**:

   - Backend middleware blocks at API level (HTTP 403)
   - Frontend guard blocks at UI level (shows modal)
   - Both use same logic for consistency

4. **Bypass**:

   - Works in both frontend and backend
   - Multiple ways to enable (flag, subdomain, command)
   - Useful for testing and special cases

5. **Status Priority**:
   - Cancelled/Suspended → Highest priority (immediate block)
   - Expired (past grace) → Block
   - Expired (grace period) → Warn but allow
   - Active → Full access

---

## 10. Example Scenarios

### Scenario 1: New Church

1. Church registers → `plan='trial'`, `trial_started_at=now`, `trial_end_date=now+30d`
2. Status: `active`, Access: ✅
3. After 30 days → Status: `trial_expiring`, Access: ✅ (grace period)
4. After 37 days → Status: `trial_expired`, Access: ❌

### Scenario 2: Upgrade During Trial

1. Church on trial (day 15)
2. Upgrades to Standard → `plan='standard'`, `subscription_start_date=now`, `subscription_end_date=now+30d`, `trial_end_date=None`
3. Status: `active`, Access: ✅

### Scenario 3: Subscription Renewal

1. Church on Standard (expires in 5 days)
2. Renews → `subscription_end_date=now+30d` (or +365d for yearly)
3. Status: `active`, Access: ✅

### Scenario 4: Manual Suspension

1. Admin sets `subscription_status='suspended'`
2. Status: `suspended`, Access: ❌ (immediate, regardless of dates)

---

This system ensures churches can only access the system when they have an active trial or subscription, with grace periods for expired accounts and bypass mechanisms for testing.
