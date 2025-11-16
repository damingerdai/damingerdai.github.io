---
title: Performance Fix for Stripe.js Script Loading
date: 2025-11-16 10:24:58
tags: [react, stripe]
categories: [前端]
---


## Technical Summary

This summary details a refactoring effort aimed at preventing the premature loading of the Stripe payment script, a common cause of degraded initial page load performance in applications that integrate third-party libraries.

### The Objective

The primary goal was to ensure that the core Stripe JavaScript file (e.g., `https://m.stripe.com/6` or `https://js.stripe.com/v3/`) is only requested and downloaded when a user actually enters a payment-required context, facilitating true lazy loading.

### The Problem: Immediate Execution on Module Import

In the original code structure, the Stripe initialization was handled in a shared service file as follows:

```typescript
// src/shared/service/stripe.ts (Old Code)
import { loadStripe } from '@stripe/stripe-js';

// The issue lies here: loadStripe is executed immediately in the top-level scope.
export const stripePromise = loadStripe(
    import.meta.env.VITE_STRIPE_PRIVATE_API_KEY
).then(stripe => {
    // ...
    return stripe;
});
```

Because the expression `loadStripe(...)` was immediately executed and its resulting Promise was exported (`export const stripePromise = ...`), any component that imported this file—even if it was a component on a non-payment page—would trigger the synchronous execution of this module. This instantly called `loadStripe()`, initiating the network request for the Stripe script, regardless of the component's render state.

### The Solution: Wrapping Initialization in an Exported Function

The fix involved decoupling the `loadStripe` call from the module's initial execution flow by wrapping it inside an exported function, `getStripePromise()`.

#### 1\. Refactoring the Stripe Service File

The `stripe.ts` file was changed to export a function instead of an eagerly evaluated Promise:

```typescript
// src/shared/service/stripe.ts (New Code)
import { loadStripe } from '@stripe/stripe-js';

export const getStripePromise = () => {
    console.warn('Stripe load function called.');
    return loadStripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY).then(stripe => {
        console.warn('Stripe script downloaded and loaded.');
        return stripe;
    });
};

// The immediate export of the promise was removed:
// //export const stripePromise = getStripePromise(); 
```

#### 2\. Local Initialization in Consumer Components

The components that utilize the Stripe context (e.g., `paymentMethod/index.tsx`, `setupChecker/index.tsx`) were updated to call this function locally within their own module scope:

```typescript
// src/components/stripe/paymentMethod/index.tsx (New Code)
import { getStripePromise } from '@/shared/service/stripe';

// The promise is now initialized here, within the specific payment module.
const stripePromise = getStripePromise(); 

// This module is expected to be loaded via React.lazy() in a parent component.
```

### Conclusion

This refactoring successfully shifts the responsibility for initiating the Stripe script download from the application's core bundle to the specific, smaller JavaScript chunk of the payment module. By ensuring the `loadStripe()` function is only executed when the payment component is mounted and its module is finally loaded (typically via a `React.lazy` trigger), we eliminate unnecessary network requests on initial page loads, contributing to a better user experience and improved performance metrics.

