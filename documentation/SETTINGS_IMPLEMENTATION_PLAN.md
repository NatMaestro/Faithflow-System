# Settings Implementation Plan

## Overview
This document outlines how settings will affect different parts of the application and how subscription/trial management will be implemented.

## Settings Categorization

### 1. **System-Wide Settings** (Affects entire application)
- **General Settings**
  - Church name → Used in headers, emails, notifications
  - Contact info → Displayed in footer, contact pages
  - Website → Links throughout the app
  
- **Subscription/Trial Management**
  - Plan type (trial/basic/standard/premium/enterprise)
  - Trial end date
  - Subscription end date
  - Subscription status (active/inactive/suspended/cancelled)
  - **Access Control**: Blocks access when expired

### 2. **Admin Dashboard Settings** (Affects admin interface only)
- **Member Settings**
  - Auto-approval → Controls member request workflow
  - Email verification → Affects member registration process
  - Self-registration → Controls if members can register themselves
  - Member ID prefix/numbering → Used when creating new members
  - Photo requirements → Enforced in admin member management
  
- **Payment Settings**
  - Payment gateway → Controls donation processing
  - Currency → Affects all payment displays
  - Receipt settings → Controls email receipts

### 3. **Member Dashboard Settings** (Affects member interface)
- **Branding Settings**
  - Logo → Displayed in member dashboard header
  - Banner → Shown on member dashboard
  - Tagline → Displayed in member dashboard
  - Theme colors → Applied to member dashboard UI
  - Language → Affects member dashboard text
  
- **Member Settings** (Member-facing)
  - Allow self-edit → Controls if members can edit their own profiles
  - Photo sharing visibility → Controls member directory
  - Birthday visibility → Controls birthday displays
  - Contact info visibility → Controls member directory

### 4. **Both Dashboards** (Affects admin and member)
- **Branding Settings**
  - Logo, banner, tagline → Used in both dashboards
  - Timezone → Affects date/time displays everywhere
  - Social media links → Footer links in both dashboards

## Subscription/Trial Management Implementation

### Backend Changes

1. **Add fields to Church model:**
   ```python
   trial_started_at = models.DateTimeField(null=True, blank=True)
   trial_end_date = models.DateTimeField(null=True, blank=True)
   subscription_start_date = models.DateTimeField(null=True, blank=True)
   subscription_end_date = models.DateTimeField(null=True, blank=True)
   ```

2. **Create Subscription Middleware:**
   - Check trial/subscription status on every request
   - Block access if trial expired and no active subscription
   - Return appropriate error messages
   - Allow grace period (e.g., 7 days after expiration)

3. **Subscription Status Logic:**
   ```python
   def get_subscription_status(self):
       now = timezone.now()
       
       # Check if trial expired
       if self.plan == 'trial' and self.trial_end_date:
           if now > self.trial_end_date:
               return 'trial_expired'
       
       # Check if subscription expired
       if self.subscription_end_date:
           if now > self.subscription_end_date:
               return 'subscription_expired'
       
       # Check subscription status
       if self.subscription_status != 'active':
           return self.subscription_status
       
       return 'active'
   ```

### Frontend Changes

1. **Create Subscription Hook:**
   ```typescript
   useSubscriptionStatus() {
     - Checks church.plan, trial_end_date, subscription_end_date
     - Returns: { status, daysRemaining, isExpired, canAccess }
     - Shows warnings when < 7 days remaining
     - Blocks access when expired
   }
   ```

2. **Create Subscription Guard Component:**
   - Wraps protected routes
   - Shows upgrade modal when trial/subscription expired
   - Redirects to upgrade page if needed

3. **Subscription Banner:**
   - Shows at top of admin dashboard when trial/subscription expiring soon
   - Shows countdown timer
   - "Upgrade" button

## Settings Integration Points

### General Settings → System-Wide
- **Church Name**: 
  - Admin dashboard header
  - Member dashboard header
  - Email signatures
  - Page titles
  
- **Contact Info**:
  - Footer component (both dashboards)
  - Contact page
  - Email templates

### Branding Settings → Both Dashboards
- **Logo**: 
  - Admin sidebar header
  - Member dashboard header
  - Email templates
  
- **Banner**: 
  - Member dashboard hero section
  - Admin dashboard (optional)
  
- **Theme Colors**: 
  - Applied via CSS variables
  - Both dashboards use theme
  
- **Timezone**: 
  - All date/time displays
  - Event scheduling
  - Notification timing

### Member Settings → Admin & Member Dashboards
- **Auto-approval**: 
  - Admin: Controls pending members workflow
  - Member: Affects registration confirmation message
  
- **Self-registration**: 
  - Admin: Controls if registration form is available
  - Member: Shows/hides registration option
  
- **Self-edit**: 
  - Member: Enables/disables profile edit button
  - Admin: Always has edit access

### Payment Settings → System-Wide
- **Payment Gateway**: 
  - Donation forms (member dashboard)
  - Payment processing (admin dashboard)
  
- **Currency**: 
  - All payment displays
  - Reports and analytics
  
- **Receipt Settings**: 
  - Email templates
  - Auto-send logic

## Implementation Steps

1. ✅ Add subscription fields to backend model
2. ✅ Create migration for subscription fields
3. ✅ Create subscription middleware
4. ✅ Create frontend subscription hook
5. ✅ Create subscription guard component
6. ✅ Connect General settings to app components
7. ✅ Connect Branding settings to dashboards
8. ✅ Connect Member settings to workflows
9. ✅ Connect Payment settings to payment features
10. ✅ Add subscription banner/warnings
11. ✅ Implement settings loading/saving with proper error handling


