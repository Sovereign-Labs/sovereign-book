# Revenue Share for Premium Components

When using Sovereign SDK's premium components (such as the ultra-low latency soft-confirming sequencer), applications that generate revenue must comply with the Sovereign Permissionless Commercial License. This guide walks you through the implementation.

## License Requirements

The license requires two things:
1. **Revenue Sharing**: Share a portion of revenue from transactions processed by the preferred sequencer
2. **Notification**: Contact Sovereign Labs before production deployment with rollup access details

The `sov-revenue-share` module handles the revenue sharing automatically, ensuring compliance while you focus on building your application.

## How Revenue Sharing Works

The module acts as an escrow for Sovereign Labs' revenue share:
- **Default Rate**: 10% (1,000 basis points) - can only be decreased, never increased
- **Activation**: Disabled by default; Sovereign Labs activates it when ready
- **Conditional**: Only applies to transactions from the preferred sequencer
- **Flexible**: Supports any token compatible with `sov-bank`

## Implementation

The revenue share module is included in the starter template. You just need to integrate it where your application generates revenue.

### Step 1: Add the Module Reference

Add the revenue share module to your application:

```rust
#[derive(Clone, ModuleInfo, ModuleRestApi)]
pub struct YourApp<S: Spec> {
    #[id]
    pub id: ModuleId,
    
    #[module]
    pub revenue_share: sov_revenue_share::RevenueShare<S>,
    
    // ... other modules
}
```

### Step 2: Share Revenue on Fee Collection

When collecting fees, check if the transaction came from the preferred sequencer and share accordingly:

```rust
pub fn charge_fee(
    &mut self,
    payer: &S::Address,
    total_fee: Amount,
    token_id: TokenId,
    context: &Context<S>,
    state: &mut impl TxState<S>,
) -> anyhow::Result<()> {
    // Only share revenue for preferred sequencer transactions
    if self.revenue_share.is_preferred_sequencer(context, state) {
        self.revenue_share.compute_and_pay_revenue_share(
            payer, 
            token_id, 
            total_fee, 
            state
        )?;
    }
    
    // Continue with your fee logic...
    Ok(())
}
```

This pattern ensures you only share revenue when required. For custom implementations, use `get_revenue_share_percentage_bps()` and `pay_revenue_share()` directly.

## Production Deployment Requirements

**Before deploying to production**, you must notify Sovereign Labs at **info@sovlabs.io** with:

1. **Documentation**: Link to or copy of your rollup interaction docs
2. **API Endpoint**: Where Sovereign can submit revenue share admin transactions
   - Must cover reasonable gas costs (via direct transfer, paymaster, or agreed method)
3. **Schema** (if not available via API): Your rollup's universal wallet schema JSON

This notification ensures Sovereign Labs can manage the revenue share module as intended by the license.

## Questions?

- Technical issues: Join our [Slack community](https://join.slack.com/t/sovereigndevelopers/shared_invite/zt-39aolimfp-XsFK6dL6LhOFHhtXsD_kCA)
- Licensing questions: Contact info@sovlabs.io