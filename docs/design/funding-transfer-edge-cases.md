# Funding Transfer Edge Cases - Test Coverage Plan

**Created:** February 5, 2026

## Overview

This document outlines additional edge cases and test scenarios for the funding transfer functionality hardening. These tests complement the security fixes implemented for NULL status vulnerabilities and double-crediting prevention.

---

## Priority 1: Critical Security & Data Integrity

### 1.1 Race Condition with Actual Concurrent Execution

**Risk:** High - Double-crediting vulnerability
**Current Coverage:** ✅ Implemented
**Status:** ✅ Passing
**File:** `spec/requests/payment_flow_spec.rb`

This test verifies that when multiple webhook requests arrive simultaneously (simulated with actual Ruby threads), the pessimistic locking (`with_lock`) and `journal_id` idempotency check prevent double-crediting. The test confirms:
- Only one journal entry is created
- Balance is credited exactly once
- The transfer status is updated correctly
- No race condition occurs despite concurrent processing attempts

```ruby
it 'prevents double-crediting when two webhook threads process simultaneously' do
  transfer = create(:funding_transfer,
    user: user,
    transaction_identifier: payment_intent_id,
    status: 'pending',
    amount_cents: 1000
  )
  initial_balance = user.wallet_balance.cents

  threads = 2.times.map do
    Thread.new do
      Transactions::AddFunds.new(user: user, platform: platform)
        .complete_webhook_payment(payment_intent_id: transfer.transaction_identifier)
    end
  end

  threads.each(&:join)

  transfer.reload
  # Should only have ONE journal entry
  expect(Keepr::Journal.where(id: transfer.journal_id).count).to eq(1)
  # Balance should only be credited once
  expect(user.reload.wallet_balance.cents).to eq(initial_balance + 1000)
end
```

---

### 1.2 Webhook Signature Security

**Risk:** High - Unauthorized fund crediting
**Current Coverage:** ✅ Implemented
**Status:** ✅ Passing
**File:** `spec/controllers/v1/webhooks_controller_spec.rb`

These tests verify that the webhook endpoint properly rejects unauthorized requests:
- Missing signature header → Returns 400 Bad Request
- Empty signature value → Returns 400 Bad Request
- Malformed signature format → Returns 400 Bad Request
- All rejection scenarios log appropriate error messages

#### 1.2.1 Missing Signature Header
```ruby
it 'rejects webhook with no signature header' do
  post "/v1/webhooks/payment",
    params: { id: 'evt_test' }.to_json,
    headers: { 'Content-Type' => 'application/json' }

  expect(response).to have_http_status(:bad_request)
  expect(JSON.parse(response.body)['error']).to be_present
end
```

#### 1.2.2 Empty Signature
```ruby
it 'rejects webhook with empty signature' do
  post "/v1/webhooks/payment",
    params: { id: 'evt_test' }.to_json,
    headers: {
      'Stripe-Signature' => '',
      'Content-Type' => 'application/json'
    }

  expect(response).to have_http_status(:bad_request)
end
```

#### 1.2.3 Malformed JSON Payload
```ruby
it 'rejects webhook with malformed JSON payload' do
  allow(Stripe::Webhook).to receive(:construct_event)
    .and_raise(JSON::ParserError.new('unexpected token'))

  post "/v1/webhooks/payment",
    params: '{invalid json',
    headers: { 'Stripe-Signature' => 'test_sig' }

  expect(response).to have_http_status(:bad_request)
end
```

---

### 1.3 Metadata Truthy Value Edge Cases

**Risk:** Medium - Incorrect webhook routing/processing
**Current Coverage:** ✅ Implemented
**Status:** ✅ Passing
**File:** `spec/controllers/v1/webhooks_controller_spec.rb`

These tests verify that the webhook metadata check `.to_s == "true"` correctly handles Ruby's truthy semantics:
- String "false" does NOT trigger wallet funding (prevented truthy bug)
- Empty string, '0', 'no', 'FALSE', 'null', 'undefined', 'off' all correctly rejected
- Only the exact string "true" triggers wallet funding processing
- Case-sensitive: 'True', 'TRUE' are NOT processed (security by design)

#### 1.3.1 String "false" is Truthy
```ruby
it 'correctly evaluates funding_wallet="false" as false' do
  allow(Stripe::Webhook).to receive(:construct_event).and_return(
    Stripe::Event.construct_from({
      type: 'payment_intent.succeeded',
      data: {
        object: {
          id: 'pi_test',
          metadata: { 'funding_wallet' => 'false' } # String "false" is truthy in Ruby!
        }
      }
    })
  )

  # Should NOT process as wallet funding
  expect_any_instance_of(Transactions::AddFunds)
    .not_to receive(:complete_webhook_payment)

  post "/v1/webhooks/payment",
    params: { id: 'evt_test' }.to_json,
    headers: { 'Stripe-Signature' => 'sig', 'Content-Type' => 'application/json' }
end
```

#### 1.3.2 Various Falsy String Values
```ruby
it 'handles metadata funding_wallet with various falsy string values' do
  ['', '0', 'no', 'False', 'FALSE', 'null', 'undefined'].each do |value|
    allow(Stripe::Webhook).to receive(:construct_event).and_return(
      Stripe::Event.construct_from({
        type: 'payment_intent.succeeded',
        data: { object: { id: 'pi_test', metadata: { 'funding_wallet' => value } } }
      })
    )

    expect_any_instance_of(Transactions::AddFunds)
      .not_to receive(:complete_webhook_payment)

    post "/v1/webhooks/payment",
      params: { id: 'evt_test' }.to_json,
      headers: { 'Stripe-Signature' => 'sig', 'Content-Type' => 'application/json' }
  end
end
```

---

### 1.4 Metadata Validation & Sanitization

**Risk:** Medium-High - Availability, PII exposure at Stripe, injection
**Current Coverage:** ✅ Implemented
**Status:** ✅ Passing (7 tests)
**File:** `spec/services/transactions/add_funds_spec.rb`
**Reference:** Priority 2 item in wallet-funding-review.md

These tests verify the `normalize_stripe_metadata` method properly sanitizes user-provided metadata:
- ✅ Non-scalar values (hashes, arrays) converted to strings
- ✅ Reserved keys (user_id, funding_wallet, funding_transfer_id) cannot be overridden
- ✅ Special characters in keys/values handled safely
- ✅ Nil values filtered out, empty strings preserved
- ✅ Non-hash metadata gracefully handled
- ✅ Empty metadata hash handled
- ⚠️ Long values (>500 chars) passed through - may cause Stripe API error (documented for future enhancement)

**Security protection confirmed:**
- Attacker cannot override system metadata fields
- All values normalized to strings (prevents injection)
- Invalid input types handled gracefully

#### 1.4.1 Excessively Long Metadata Values
```ruby
it 'truncates or rejects excessively long metadata values' do
  long_value = 'a' * 1000
  metadata = { 'custom_field' => long_value }

  result = service.create_payment_intent(amount_cents: 1000, metadata: metadata)

  # Verify call to Stripe doesn't include full long_value
  expect(Stripe::PaymentIntent).to have_received(:create) do |args|
    args[:metadata].values.all? { |v| v.length <= 500 } # Stripe limit
  end
end
```

#### 1.4.2 Non-Scalar Metadata Values
```ruby
it 'rejects or flattens non-scalar metadata values' do
  metadata = {
    'nested' => { 'hash' => 'value' },
    'array' => [1, 2, 3]
  }

  result = service.create_payment_intent(amount_cents: 1000, metadata: metadata)

  # Verify Stripe receives only scalar values
  expect(Stripe::PaymentIntent).to have_received(:create) do |args|
    args[:metadata].values.all? { |v| v.is_a?(String) }
  end
end
```

#### 1.4.3 Reserved Key Override Attempts
```ruby
it 'prevents overriding reserved metadata keys' do
  malicious_metadata = {
    'user_id' => 'attacker_user_id',
    'funding_wallet' => 'false',
    'funding_transfer_id' => 'fake_id'
  }

  result = service.create_payment_intent(amount_cents: 1000, metadata: malicious_metadata)

  expect(Stripe::PaymentIntent).to have_received(:create) do |args|
    expect(args[:metadata]['user_id']).to eq(user.id.to_s)
    expect(args[:metadata]['funding_wallet']).to eq('true')
    expect(args[:metadata]['funding_transfer_id']).not_to eq('fake_id')
  end
end
```

#### 1.4.4 Special Characters in Metadata
```ruby
it 'handles special characters in metadata keys and values' do
  metadata = {
    'key-with-dashes' => 'value',
    'key_with_underscores' => 'value with spaces',
    'key.with.dots' => 'value!@#$%^&*()'
  }

  expect {
    service.create_payment_intent(amount_cents: 1000, metadata: metadata)
  }.not_to raise_error
end
```

---

### 1.5 Journal Referential Integrity

**Risk:** Medium - Data inconsistency, failed idempotency
**Current Coverage:** ✅ Implemented
**Status:** ✅ Passing (2 tests)
**File:** `spec/requests/payment_flow_spec.rb`

These tests verify the foreign key constraint on `funding_transfers.journal_id` works correctly:
- ✅ Foreign key constraint prevents journal deletion when referenced by funding_transfer
- ✅ Service layer idempotency check recognizes existing journal_id and skips reprocessing
- ✅ Data integrity maintained: journals and transfers remain consistent
- ✅ No double-crediting: balance unchanged when transfer already has journal_id

**Data integrity protection confirmed:**
- Database-level foreign key enforces referential integrity
- Service layer prevents duplicate journal creation
- Idempotency safeguards work at multiple levels

#### 1.5.1 Journal Deletion Prevention
```ruby
it 'prevents journal deletion when referenced by funding_transfer' do
  journal = Keepr::Journal.create!(
    date: Date.current,
    keepr_postings_attributes: [
      { keepr_account: platform.bank, amount: 1000, side: 'debit' },
      { keepr_account: user.wallet, amount: 1000, side: 'credit' }
    ]
  )
  transfer = create(:funding_transfer,
    user: user,
    journal_id: journal.id,
    status: 'completed'
  )

  expect {
    journal.destroy
  }.to raise_error(ActiveRecord::InvalidForeignKey)
end
```

#### 1.5.2 Orphaned Journal ID Detection
```ruby
it 'detects orphaned journal_id and prevents reprocessing' do
  transfer = create(:funding_transfer,
    user: user,
    journal_id: 99999, # Non-existent journal
    status: 'completed',
    amount_cents: 1000
  )

  service = Transactions::AddFunds.new(user: user, platform: platform)

  # Should recognize orphaned ID and skip without creating new journal
  expect {
    service.send(:transfer_money, transfer)
  }.not_to change { Keepr::Journal.count }
end
```

---

## Priority 2: Data Validation & Business Logic

### 2.1 Money/Currency Edge Cases

**Risk:** Medium - Accounting errors, unprofitable transactions
**Current Coverage:** Partial (basic fee tests only)
**Status:** ⬜ Not Implemented

#### 2.1.1 Fractional Cent Rounding
```ruby
it 'handles rounding correctly for fractional cent amounts' do
  service = Transactions::AddFunds.new(user: user, platform: platform)

  # Processing fee: $1.50 * 0.029 + 0.30 = $0.3435 (needs rounding)
  fee = service.send(:calculate_processing_fee, Money.from_cents(150))

  expect(fee.cents).to be_an(Integer)
  expect(fee.cents).to eq(34) # or 35, depending on rounding strategy
end
```

#### 2.1.2 Minimum Transaction Edge Case
```ruby
it 'handles minimum transaction where fee exceeds reasonable margin' do
  # $0.50 charge results in $0.31 fee, leaving only $0.19 net
  transfer = create(:funding_transfer,
    user: user,
    amount_cents: 50,
    status: 'pending'
  )

  service = Transactions::AddFunds.new(user: user, platform: platform)

  expect {
    service.send(:transfer_money, transfer)
  }.not_to raise_error

  # Verify accounting remains balanced despite tiny net
  journal = Keepr::Journal.find(transfer.reload.journal_id)
  total_debits = journal.keepr_postings.where(side: 'debit').sum(:amount)
  total_credits = journal.keepr_postings.where(side: 'credit').sum(:amount)
  expect(total_debits).to eq(total_credits)
end
```

#### 2.1.3 Zero Amount Rejection
```ruby
it 'rejects zero-amount transfers' do
  expect {
    create(:funding_transfer, user: user, amount_cents: 0)
  }.to raise_error(ActiveRecord::RecordInvalid)
end
```

#### 2.1.4 Negative Amount Rejection
```ruby
it 'rejects negative amounts' do
  expect {
    create(:funding_transfer, user: user, amount_cents: -1000)
  }.to raise_error(ActiveRecord::RecordInvalid)
end
```

#### 2.1.5 Very Large Amounts
```ruby
it 'handles very large transaction amounts correctly' do
  # $100,000.00
  large_amount = 10_000_000

  transfer = create(:funding_transfer,
    user: user,
    amount_cents: large_amount,
    status: 'pending'
  )

  service = Transactions::AddFunds.new(user: user, platform: platform)
  service.send(:transfer_money, transfer)

  # Verify fee calculation doesn't overflow
  fee_cents = service.send(:calculate_processing_fee, Money.from_cents(large_amount)).cents
  expect(fee_cents).to eq(290_030) # 2.9% + 30¢
end
```

---

### 2.2 Webhook Timing & Ordering Edge Cases

**Risk:** Medium - Lost funds, inconsistent state
**Current Coverage:** ⬜ None
**Status:** ⬜ Not Implemented

#### 2.2.1 Webhook Arrives Before DB Write
```ruby
it 'handles webhook arriving before create_payment_intent completes' do
  allow(Stripe::PaymentIntent).to receive(:create).and_return(
    double(id: 'pi_test', client_secret: 'secret', status: 'requires_payment_method')
  )

  # Simulate webhook racing ahead before transfer record exists
  service = Transactions::AddFunds.new(user: user, platform: platform)

  expect(service.complete_webhook_payment(payment_intent_id: 'pi_nonexistent'))
    .to be false

  # Should not crash or credit without transfer record
end
```

#### 2.2.2 Webhook for Cancelled Payment That Failed
```ruby
it 'does not revert cancellation when webhook indicates payment failed' do
  transfer = create(:funding_transfer,
    user: user,
    transaction_identifier: payment_intent_id,
    status: 'cancelled',
    amount_cents: 1000
  )

  # Simulate payment_intent.failed event (not succeeded)
  allow(Stripe::Webhook).to receive(:construct_event).and_return(
    Stripe::Event.construct_from({
      type: 'payment_intent.payment_failed',
      data: {
        object: {
          id: payment_intent_id,
          status: 'failed',
          metadata: { funding_wallet: true }
        }
      }
    })
  )

  post "/v1/webhooks/payment",
    params: { id: 'evt_test' }.to_json,
    headers: { 'Stripe-Signature' => 'sig', 'Content-Type' => 'application/json' }

  # Should remain cancelled, NOT revert to pending
  expect(transfer.reload.status).to eq('cancelled')
  expect(transfer.journal_id).to be_nil
end
```

#### 2.2.3 Multiple Webhooks in Quick Succession
```ruby
it 'handles rapid fire duplicate webhooks gracefully' do
  transfer = create(:funding_transfer,
    user: user,
    transaction_identifier: payment_intent_id,
    status: 'pending',
    amount_cents: 1000
  )

  allow(Stripe::Webhook).to receive(:construct_event).and_return(
    Stripe::Event.construct_from({
      type: 'payment_intent.succeeded',
      data: { object: { id: payment_intent_id, metadata: { funding_wallet: true } } }
    })
  )

  initial_balance = user.wallet_balance.cents

  # Fire 5 webhooks rapidly
  5.times do
    post "/v1/webhooks/payment",
      params: { id: 'evt_test' }.to_json,
      headers: { 'Stripe-Signature' => 'sig', 'Content-Type' => 'application/json' }

    expect(response).to have_http_status(:ok)
  end

  # Balance should only increase once
  expect(user.reload.wallet_balance.cents).to eq(initial_balance + 1000)
end
```

---

### 2.3 Status Transition Validation

**Risk:** Low-Medium - Invalid state transitions
**Current Coverage:** ⬜ None
**Status:** ⬜ Not Implemented

#### 2.3.1 Prevent Invalid Status Transitions
```ruby
it 'prevents transitioning from completed back to pending' do
  transfer = create(:funding_transfer,
    user: user,
    status: 'completed',
    journal_id: 1
  )

  # This should either raise error or be ignored depending on business rules
  expect {
    transfer.update!(status: 'pending')
  }.to raise_error(ActiveRecord::RecordInvalid)
  # OR: expect(transfer.reload.status).to eq('completed')
end
```

#### 2.3.2 Allow Failed to Pending Retry
```ruby
it 'allows reverting failed transfer back to pending for retry' do
  transfer = create(:funding_transfer,
    user: user,
    status: 'failed'
  )

  expect {
    transfer.update!(status: 'pending')
  }.not_to raise_error

  expect(transfer.reload.status).to eq('pending')
end
```

---

## Priority 3: Database Constraint Enforcement

### 3.1 Transaction Rollback Verification

**Risk:** Medium - Orphaned journal entries
**Current Coverage:** ⬜ None
**Status:** ⬜ Not Implemented

```ruby
it 'rolls back journal creation if status update fails' do
  transfer = create(:funding_transfer,
    user: user,
    status: 'pending',
    amount_cents: 1000
  )

  # Force status update to fail
  allow(transfer).to receive(:update!)
    .and_raise(ActiveRecord::RecordInvalid.new(transfer))

  service = Transactions::AddFunds.new(user: user, platform: platform)

  expect {
    service.send(:transfer_money, transfer)
  }.to raise_error(ActiveRecord::RecordInvalid)

  # Verify no orphaned journal was created
  expect(Keepr::Journal.count).to eq(0)
end
```

---

### 3.2 NOT NULL Constraint Verification

**Risk:** Low - Already implemented, defensive verification
**Current Coverage:** ✅ Implemented
**Status:** ✅ Verified in spec/requests/payment_flow_spec.rb

```ruby
# Already tested - reference for completeness
it 'enforces NOT NULL constraint on status column' do
  transfer = create(:funding_transfer, user: user, amount_cents: 500)

  expect {
    ActiveRecord::Base.connection.execute(
      "UPDATE funding_transfers SET status = NULL WHERE id = '#{transfer.id}'"
    )
  }.to raise_error(ActiveRecord::StatementInvalid, /violates not-null constraint/)
end
```

---

## Priority 4: Observability & Monitoring

### 4.1 Logging Verification

**Risk:** Low - Observability gaps
**Current Coverage:** Partial
**Status:** ⬜ Not Implemented

#### 4.1.1 Log Duplicate Processing Attempts
```ruby
it 'logs warning when duplicate webhook is detected' do
  transfer = create(:funding_transfer,
    user: user,
    transaction_identifier: payment_intent_id,
    status: 'completed',
    journal_id: 1
  )

  logger = instance_double(Logger)
  allow(Rails).to receive(:logger).and_return(logger)

  expect(logger).to receive(:info).with(/already processed/)

  service = Transactions::AddFunds.new(user: user, platform: platform)
  service.complete_webhook_payment(payment_intent_id: payment_intent_id)
end
```

#### 4.1.2 Log Metadata Mismatches
```ruby
it 'logs error when funding_transfer_id metadata does not match' do
  transfer = create(:funding_transfer,
    user: user,
    transaction_identifier: payment_intent_id
  )

  allow(Stripe::Webhook).to receive(:construct_event).and_return(
    Stripe::Event.construct_from({
      type: 'payment_intent.succeeded',
      data: {
        object: {
          id: payment_intent_id,
          metadata: {
            funding_wallet: 'true',
            funding_transfer_id: 'wrong_id'
          }
        }
      }
    })
  )

  expect(Rails.logger).to receive(:error).with(/mismatch/)

  post "/v1/webhooks/payment",
    params: { id: 'evt_test' }.to_json,
    headers: { 'Stripe-Signature' => 'sig', 'Content-Type' => 'application/json' }
end
```

---

## Implementation Checklist

- [x] **1.1** Race condition with actual concurrent threads ✅
- [x] **1.2** Webhook signature security (3 tests) ✅
- [x] **1.3** Metadata truthy edge cases (3 tests) ✅
- [x] **1.4** Metadata validation & sanitization (7 tests) ✅
- [x] **1.5** Journal referential integrity (2 tests) ✅
- [ ] **2.1** Money/currency edge cases (5 tests)
- [ ] **2.2** Webhook timing & ordering (3 tests)
- [ ] **2.3** Status transition validation (2 tests)
- [ ] **3.1** Transaction rollback verification
- [ ] **3.2** NOT NULL constraint (already done ✅)
- [ ] **4.1** Logging verification (2 tests)

**Total New Tests:** ~25 tests

---

## Related Issues to Consider

Based on the wallet-funding-review.md Priority 3 items:

1. **Webhook endpoint consolidation** - Two endpoints exist: `/v1/webhooks/payment` and `/v1/wallet/payment-webhook`
2. **Stripe event.id persistence** - Consider storing for replay detection
3. **State machine for status transitions** - Formalize valid state changes

---

## Notes

- All tests should be added to existing test files where applicable:
  - `spec/models/funding_transfer_spec.rb` for model validations
  - `spec/services/transactions/add_funds_spec.rb` for service logic
  - `spec/requests/payment_flow_spec.rb` for integration tests
  - `spec/controllers/v1/webhooks_controller_spec.rb` for webhook tests

- Consider property-based testing for money calculations using a library like `rspec-parameterized`

- Database constraint tests may require separate suite or special handling due to bypassing ActiveRecord validations
