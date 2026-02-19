# AllMap Monetization & Revenue Share Spec Sheet

**Target Audience:** AI Coding Agent / Senior Backend Engineer
**Goal:** Implement a "Creator Economy" monetization system where map owners can charge subscriptions, and revenue is automatically distributed to contributors based on their "Karma" (contribution quality).

## 1. Architecture Overview

*   **Platform:** Django (Backend), Stripe Connect (Payments & Payouts).
*   **Revenue Model:**
    *   **Inflow:** End-user pays Monthly/Yearly Subscription for Map Access.
    *   **Split:** Platform Fee (e.g., 20%) + Stripe Fees -> Remaining Pot.
    *   **Outflow:** Pot distributed to Contributors proportional to their Karma share on that specific map.
*   **Key Constraint:** Payouts must be automated via Stripe Connect Express accounts to handle KYC and tax forms for influencers.

## 2. Database Schema (New App: `monetization`)

Create a new Django app `monetization`.

### Model 1: `MapSubscriptionPlan`
Controls the pricing for a specific map.
```python
class MapSubscriptionPlan(models.Model):
    map = models.OneToOneField('maps.Map', related_name='subscription_plan')
    stripe_product_id = models.CharField(max_length=100)
    stripe_price_id_monthly = models.CharField(max_length=100)
    stripe_price_id_yearly = models.CharField(max_length=100)
    price_monthly = models.DecimalField(max_digits=6, decimal_places=2)
    price_yearly = models.DecimalField(max_digits=6, decimal_places=2)
    platform_fee_percent = models.DecimalField(default=20.00) # Platform take
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Model 2: `PayoutRecipient`
Links a User to a Map with their Stripe Connect ID.
```python
class PayoutRecipient(models.Model):
    map = models.ForeignKey('maps.Map', related_name='payout_recipients')
    user = models.ForeignKey(User, related_name='payout_settings')
    stripe_connect_id = models.CharField(max_length=100, help_text="acct_...")
    is_onboarding_complete = models.BooleanField(default=False)
    
    # Optional override for "Boss" users who negotiate fixed %
    fixed_share_percent = models.DecimalField(null=True, blank=True)
    
    class Meta:
        unique_together = ['map', 'user']
```

### Model 3: `MonthlyRevenueSnapshot`
Records the "Pot" for a given month before distribution.
```python
class MonthlyRevenueSnapshot(models.Model):
    map = models.ForeignKey('maps.Map')
    period_start = models.DateField()
    period_end = models.DateField()
    gross_revenue = models.DecimalField(...)
    platform_fee_amount = models.DecimalField(...)
    stripe_fee_estimated = models.DecimalField(...)
    distributable_amount = models.DecimalField(...) # The pot to split
    status = models.CharField(choices=['calculating', 'ready', 'paid'])
```

### Model 4: `PayoutTransaction`
The actual record of money sent to a user.
```python
class PayoutTransaction(models.Model):
    snapshot = models.ForeignKey(MonthlyRevenueSnapshot)
    recipient = models.ForeignKey(PayoutRecipient)
    amount = models.DecimalField(...)
    karma_score_at_time = models.IntegerField(help_text="User's karma on map at calculation time")
    share_percent = models.DecimalField(...)
    stripe_transfer_id = models.CharField(...)
    status = models.CharField(choices=['pending', 'success', 'failed'])
```

## 3. Step-by-Step Implementation Plan

### Phase 1: Stripe Connect Setup (Infrastructure)
1.  **Configure Stripe:** Enable "Connect" in Stripe Dashboard. Choose "Express" accounts (best for influencers).
2.  **Env Variables:** Add `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_CONNECT_CLIENT_ID`.
3.  **Onboarding API:** Create endpoint `POST /api/monetization/onboard/`.
    *   Calls `stripe.Account.create(type='express')`.
    *   Returns `account_link` URL for the user to complete KYC on Stripe.
4.  **Webhook Handler:** Create listener for `account.updated` to mark `PayoutRecipient.is_onboarding_complete = True`.

### Phase 2: Subscription Gating (The Paywall)
1.  **Create Plan API:** Endpoint for Map Owners to set price.
    *   Logic: Create Product/Price objects in Stripe via API. Save `MapSubscriptionPlan`.
2.  **Purchase API:** Endpoint `POST /api/monetization/subscribe/`.
    *   Input: `map_id`, `plan_type` (monthly/yearly).
    *   Logic: Create Stripe Checkout Session. Mode: `subscription`.
    *   **Crucial:** Do NOT use `transfer_data` (destination charges) here. We need to hold funds and split them later (Separate Charges and Transfers).
3.  **Access Control Middleware:**
    *   Update `Map.can_view(user)`: If map has active `MapSubscriptionPlan`, check if `request.user` has active Subscription object.

### Phase 3: The Karma Engine (The Calculator)
*Create a service class `RevenueCalculator`.*

**Logic Flow:**
1.  **Trigger:** Cron job runs on 1st of month.
2.  **Fetch Revenue:** Query Stripe for all charges related to this Map's `product_id` in the last 30 days.
3.  **Calculate Pot:** `Gross - Platform Fee = Net Pot`.
4.  **Fetch Contributors:** Get all `PayoutRecipient`s for this map.
5.  **Calculate Shares:**
    *   `TotalMapKarma` = Sum of `karma_awarded` for ALL locations on map.
    *   For each User: `UserMapKarma` = Sum of `karma_awarded` for THEIR locations on map.
    *   `UserShare` = `UserMapKarma / TotalMapKarma`.
    *   *Edge Case:* If `fixed_share_percent` exists, reserve that slice first, distribute remainder by Karma.

### Phase 4: Payout Execution (The Transfer)
*Create a management command `python manage.py process_monthly_payouts`.*

1.  Loop through `MonthlyRevenueSnapshot` where status='ready'.
2.  Loop through calculated `PayoutTransaction`s.
3.  **Stripe Transfer:**
    ```python
    stripe.Transfer.create(
      amount=user_amount_cents,
      currency="usd",
      destination=recipient.stripe_connect_id,
      transfer_group=f"payout_{snapshot.id}"
    )
    ```
4.  Update `PayoutTransaction` status to `success`.
5.  Send email notification to Influencer ("You got paid!").

## 4. Critical Rules & Edge Cases

*   **Refund Clawbacks:** If a user refunds, the map owner owes the platform. Keep a "Reserve" (rolling 10%) if refund rates are high.
*   **Minimum Payout:** Don't transfer less than $2.00 (Stripe fees make it worthless). Accumulate balance instead.
*   **Karma Gaming:**
    *   Implement `Location.is_verified`. Only verified locations count toward revenue share.
    *   Map Owner must "Approve" pins for them to count toward Karma.
*   **Inactive Users:** If a user deletes their account, their share redistributes to the remaining pool.

## 5. UI/Frontend Requirements (Brief)

1.  **"Monetize" Tab (Owner):** Connect Stripe, Set Price, View Total Revenue.
2.  **"Contributor" Dashboard (Influencer):**
    *   "Connect Payout Account" button.
    *   Graph: "Your Karma Share" (e.g., 4.5%).
    *   List: "Past Payouts".
3.  **Map Access (Subscriber):** Credit Card form integration.

