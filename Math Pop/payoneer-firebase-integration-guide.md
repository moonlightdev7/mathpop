# Payoneer Checkout + Firebase: Math Pop Integration Guide

This is a step-by-step guide for integrating Payoneer Checkout into mathpop.io using Firebase Cloud Functions as the backend.

---

## How It Works (Big Picture)

Math Pop is a static site on GitHub Pages — it has no server. But Payoneer Checkout requires a server to securely create payment sessions (you can't put your API credentials in client-side JavaScript — anyone could steal them).

Firebase Cloud Functions solve this. They're small server-side functions that run on Google's servers. Your front-end JavaScript calls them, they talk to Payoneer's API, and they handle the secure parts.

Here's the flow for a customer buying the Customization Pass ($2.99):

```
User clicks "Buy Customization Pass"
        │
        ▼
Your JS calls a Firebase Cloud Function
        │
        ▼
Cloud Function calls Payoneer API (POST /lists)
with your secret credentials
        │
        ▼
Payoneer returns a checkout session (with a redirect URL)
        │
        ▼
User is redirected to Payoneer's hosted payment page
(or sees an embedded payment form on your site)
        │
        ▼
User enters card details, pays
        │
        ▼
Payoneer sends a webhook (notification) to another
Cloud Function confirming payment succeeded
        │
        ▼
Cloud Function updates Firestore:
  users/{uid}/customizationPass = true
        │
        ▼
Math Pop reads that flag → unlocks neon themes,
gradients, color wheel, gradient maker
```

---

## Prerequisites (Do These First)

### 1. Get Payoneer Checkout Activated

You already signed up for Payoneer and are awaiting identity verification. Once verified:

1. Log into your Payoneer account
2. Find the **"Checkout"** section in the sidebar
3. Click **"Activate"**
4. Go to **Checkout → Integration → API access**
5. You'll get two credentials:
   - **Merchant Code** (like a username)
   - **Payment Token** (like a password)
6. Also grab your **Sandbox** credentials for testing (same page)

The sandbox base URL for testing is: `https://api.sandbox.oscato.com/api/`
The live base URL is: `https://api.live.oscato.com/api/`

### 2. Upgrade Firebase (If Needed)

Cloud Functions require the **Blaze (pay-as-you-go)** plan. Don't worry — for the volume Math Pop will have, you'll stay within the free tier for a long time. You only pay if you exceed 2 million function invocations/month.

If you're already using Firebase Auth + Firestore, you may already be on Blaze.

### 3. Install Firebase CLI

```bash
npm install -g firebase-tools
firebase login
```

---

## Step 1: Set Up the Firebase Functions Project

In your `multiply` folder (the Math Pop project root), initialize Firebase Functions:

```bash
cd ~/multiply
firebase init functions
```

Choose:
- **Language:** JavaScript (keeps it simple, matches your existing code)
- **ESLint:** Yes (optional but recommended)
- **Install dependencies:** Yes

This creates a `functions/` folder inside your project:

```
multiply/
├── index.html              ← your main Math Pop file
├── functions/
│   ├── index.js            ← your Cloud Functions go here
│   ├── package.json
│   └── node_modules/
├── firebase.json
└── ... (your other folders)
```

---

## Step 2: Store Your Payoneer Credentials Securely

Never hardcode API keys. Use Firebase environment config:

```bash
# Set your Payoneer credentials as secrets
firebase functions:secrets:set PAYONEER_MERCHANT_CODE
# (it will prompt you to enter the value)

firebase functions:secrets:set PAYONEER_PAYMENT_TOKEN
# (it will prompt you to enter the value)

# For sandbox testing, you can also set:
firebase functions:secrets:set PAYONEER_SANDBOX_MERCHANT_CODE
firebase functions:secrets:set PAYONEER_SANDBOX_PAYMENT_TOKEN
```

---

## Step 3: Write the Cloud Functions

Open `functions/index.js` and replace its contents with:

```javascript
const functions = require("firebase-functions");
const admin = require("firebase-admin");
const { defineSecret } = require("firebase-functions/params");

admin.initializeApp();
const db = admin.firestore();

// ─── Secrets ───
const merchantCode = defineSecret("PAYONEER_MERCHANT_CODE");
const paymentToken = defineSecret("PAYONEER_PAYMENT_TOKEN");

// ─── Config ───
// Change to "https://api.live.oscato.com/api/" when going live
const PAYONEER_API_BASE = "https://api.sandbox.oscato.com/api/";

// Your products
const PRODUCTS = {
  customization_pass: {
    name: "Customization Pass",
    amount: 2.99,
    currency: "USD",
    description: "Unlock neon themes, gradients, color wheel & gradient maker",
  },
  premium_monthly: {
    name: "Premium Pass (Monthly)",
    amount: 4.99,
    currency: "USD",
    description: "Crown badge, titles, Fractions, Decimals, Percentages, Geometry, Basic Algebra",
  },
  premium_yearly: {
    name: "Premium Pass (Yearly)",
    amount: 49.99,
    currency: "USD",
    description: "Premium Pass — annual billing (save $10/year)",
  },
};


// ═══════════════════════════════════════════════════════
// FUNCTION 1: Create a Payoneer Checkout Session
// ═══════════════════════════════════════════════════════

exports.createCheckoutSession = functions
  .runWith({ secrets: [merchantCode, paymentToken] })
  .https.onCall(async (data, context) => {

    // 1. Verify the user is logged in
    if (!context.auth) {
      throw new functions.https.HttpsError(
        "unauthenticated",
        "You must be logged in to make a purchase."
      );
    }

    // 2. Validate the product
    const productId = data.productId;
    const product = PRODUCTS[productId];
    if (!product) {
      throw new functions.https.HttpsError(
        "invalid-argument",
        "Invalid product ID."
      );
    }

    // 3. Generate a unique transaction ID
    const transactionId = `mathpop_${context.auth.uid}_${productId}_${Date.now()}`;

    // 4. Build the Payoneer LIST request body
    //    This creates a checkout session on Payoneer's servers
    const listRequestBody = {
      transactionId: transactionId,
      country: "US",           // Merchant country (use US for USD pricing)
      integration: "HOSTED",   // Use Payoneer's hosted payment page
      payment: {
        amount: product.amount,
        currency: product.currency,
        reference: productId,
        description: product.description,
      },
      customer: {
        email: context.auth.token.email || "",
        registration: {
          id: context.auth.uid,
        },
      },
      callback: {
        // Where Payoneer sends the user AFTER payment
        returnUrl: `https://mathpop.io?payment=success&product=${productId}`,
        cancelUrl: `https://mathpop.io?payment=cancelled`,
        // Where Payoneer sends server-to-server notifications
        notificationUrl: `https://us-central1-YOUR_PROJECT_ID.cloudfunctions.net/payoneerWebhook`,
      },
      style: {
        // Optional: customize the hosted payment page appearance
        language: "en",
      },
    };

    // 5. Call Payoneer's API to create the session
    const authHeader = Buffer.from(
      `${merchantCode.value()}:${paymentToken.value()}`
    ).toString("base64");

    try {
      const response = await fetch(`${PAYONEER_API_BASE}lists`, {
        method: "POST",
        headers: {
          "Content-Type": "application/vnd.optile.payment.enterprise-v1-extensible+json",
          "Accept": "application/vnd.optile.payment.enterprise-v1-extensible+json",
          "Authorization": `Basic ${authHeader}`,
        },
        body: JSON.stringify(listRequestBody),
      });

      const result = await response.json();

      if (!response.ok) {
        console.error("Payoneer API error:", result);
        throw new functions.https.HttpsError(
          "internal",
          "Failed to create checkout session."
        );
      }

      // 6. Save the pending transaction to Firestore
      await db.collection("transactions").doc(transactionId).set({
        userId: context.auth.uid,
        productId: productId,
        amount: product.amount,
        currency: product.currency,
        status: "pending",
        payoneerListId: result.identification?.longId || null,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
      });

      // 7. Return the redirect URL to the client
      //    The client will redirect the user to this URL
      const redirectLink = result.redirect?.url || null;

      return {
        redirectUrl: redirectLink,
        transactionId: transactionId,
      };

    } catch (error) {
      console.error("Error creating checkout session:", error);
      throw new functions.https.HttpsError(
        "internal",
        "Something went wrong. Please try again."
      );
    }
  });


// ═══════════════════════════════════════════════════════
// FUNCTION 2: Handle Payoneer Webhook (Payment Notification)
// ═══════════════════════════════════════════════════════

exports.payoneerWebhook = functions.https.onRequest(async (req, res) => {

  // Payoneer sends POST requests with payment status updates
  if (req.method !== "POST") {
    res.status(405).send("Method not allowed");
    return;
  }

  try {
    const notification = req.body;

    console.log("Payoneer webhook received:", JSON.stringify(notification));

    // Extract key fields from the notification
    const transactionId = notification.transactionId;
    const statusCode = notification.statusCode;
    // Possible statusCodes: "charged", "declined", "pending", etc.

    if (!transactionId) {
      console.error("No transactionId in webhook");
      res.status(400).send("Missing transactionId");
      return;
    }

    // Look up the transaction in Firestore
    const txRef = db.collection("transactions").doc(transactionId);
    const txDoc = await txRef.get();

    if (!txDoc.exists) {
      console.error("Transaction not found:", transactionId);
      res.status(404).send("Transaction not found");
      return;
    }

    const txData = txDoc.data();

    // ─── PAYMENT SUCCESSFUL ───
    if (statusCode === "charged") {

      // Update the transaction record
      await txRef.update({
        status: "completed",
        completedAt: admin.firestore.FieldValue.serverTimestamp(),
        payoneerStatus: statusCode,
      });

      // Grant the user their purchase!
      const userRef = db.collection("users").doc(txData.userId);

      if (txData.productId === "customization_pass") {
        // One-time purchase: flip the flag
        await userRef.update({
          customizationPass: true,
          customizationPassPurchasedAt: admin.firestore.FieldValue.serverTimestamp(),
        });

      } else if (txData.productId === "premium_monthly" || txData.productId === "premium_yearly") {
        // Subscription: set active + expiry date
        const now = new Date();
        const expiryDate = txData.productId === "premium_monthly"
          ? new Date(now.setMonth(now.getMonth() + 1))
          : new Date(now.setFullYear(now.getFullYear() + 1));

        await userRef.update({
          premiumPass: true,
          premiumExpiry: admin.firestore.Timestamp.fromDate(expiryDate),
          premiumPlan: txData.productId === "premium_monthly" ? "monthly" : "yearly",
          premiumPurchasedAt: admin.firestore.FieldValue.serverTimestamp(),
        });
      }

      console.log(`Payment successful for user ${txData.userId}, product ${txData.productId}`);

    // ─── PAYMENT FAILED / DECLINED ───
    } else if (statusCode === "declined" || statusCode === "failed") {
      await txRef.update({
        status: "failed",
        payoneerStatus: statusCode,
        failedAt: admin.firestore.FieldValue.serverTimestamp(),
      });
      console.log(`Payment failed for transaction ${transactionId}`);

    // ─── OTHER STATUS (pending, etc.) ───
    } else {
      await txRef.update({
        payoneerStatus: statusCode,
      });
      console.log(`Payment status update: ${statusCode} for ${transactionId}`);
    }

    // Always respond 200 to Payoneer so they know we got it
    res.status(200).send("OK");

  } catch (error) {
    console.error("Webhook processing error:", error);
    // Still return 200 to avoid Payoneer retrying endlessly
    res.status(200).send("OK");
  }
});


// ═══════════════════════════════════════════════════════
// FUNCTION 3: Check Subscription Status (Optional Helper)
// ═══════════════════════════════════════════════════════

exports.checkSubscriptionStatus = functions
  .https.onCall(async (data, context) => {

    if (!context.auth) {
      throw new functions.https.HttpsError("unauthenticated", "Must be logged in.");
    }

    const userDoc = await db.collection("users").doc(context.auth.uid).get();
    if (!userDoc.exists) {
      return { customizationPass: false, premiumPass: false };
    }

    const userData = userDoc.data();

    // Check if premium has expired
    let premiumActive = false;
    if (userData.premiumPass && userData.premiumExpiry) {
      premiumActive = userData.premiumExpiry.toDate() > new Date();
      // If expired, update the flag
      if (!premiumActive) {
        await db.collection("users").doc(context.auth.uid).update({
          premiumPass: false,
        });
      }
    }

    return {
      customizationPass: userData.customizationPass || false,
      premiumPass: premiumActive,
      premiumPlan: premiumActive ? userData.premiumPlan : null,
      premiumExpiry: premiumActive ? userData.premiumExpiry.toDate().toISOString() : null,
    };
  });
```

### Install dependencies in the functions folder:

```bash
cd functions
npm install firebase-admin firebase-functions
```

---

## Step 4: Deploy the Cloud Functions

```bash
cd ~/multiply
firebase deploy --only functions
```

After deploying, Firebase will give you URLs like:

```
✓ functions[createCheckoutSession]: Callable
✓ functions[payoneerWebhook]: https://us-central1-YOUR_PROJECT.cloudfunctions.net/payoneerWebhook
✓ functions[checkSubscriptionStatus]: Callable
```

**Copy the webhook URL** and go back to Step 3's code — replace `YOUR_PROJECT_ID` in the `notificationUrl` with your actual Firebase project ID. Then redeploy.

---

## Step 5: Add the Client-Side Code to Math Pop

In your `index.html`, add this code to handle purchases. This goes wherever your gamepass buttons live:

```html
<!-- Add this ONCE, near your other Firebase script tags -->
<script src="https://www.gstatic.com/firebasejs/10.x.x/firebase-functions.js"></script>
```

Then in your JavaScript:

```javascript
// ─── Payment Integration ───

// Initialize Cloud Functions
const functions = firebase.functions();

// If testing locally, uncomment this:
// functions.useEmulator("localhost", 5001);


async function buyProduct(productId) {
  // productId is one of: "customization_pass", "premium_monthly", "premium_yearly"

  const user = firebase.auth().currentUser;
  if (!user) {
    alert("Please sign in to make a purchase!");
    return;
  }

  // Show loading state
  const button = document.querySelector(`[data-product="${productId}"]`);
  if (button) {
    button.disabled = true;
    button.textContent = "Processing...";
  }

  try {
    // Call your Cloud Function
    const createCheckout = functions.httpsCallable("createCheckoutSession");
    const result = await createCheckout({ productId: productId });

    if (result.data.redirectUrl) {
      // Redirect to Payoneer's hosted payment page
      window.location.href = result.data.redirectUrl;
    } else {
      alert("Something went wrong. Please try again.");
    }

  } catch (error) {
    console.error("Purchase error:", error);
    alert(error.message || "Purchase failed. Please try again.");
  } finally {
    if (button) {
      button.disabled = false;
      button.textContent = "Buy Now";
    }
  }
}


// ─── Handle Return from Payment ───

function handlePaymentReturn() {
  const params = new URLSearchParams(window.location.search);
  const paymentStatus = params.get("payment");
  const product = params.get("product");

  if (paymentStatus === "success") {
    // Show a success message
    // The actual unlock happens via the webhook → Firestore update
    // But we can check immediately:
    setTimeout(async () => {
      await refreshSubscriptionStatus();
    }, 2000); // Give webhook a moment to process

    showNotification("Payment received! Your purchase is being activated...");
    // Clean the URL
    window.history.replaceState({}, "", window.location.pathname);

  } else if (paymentStatus === "cancelled") {
    showNotification("Payment was cancelled.");
    window.history.replaceState({}, "", window.location.pathname);
  }
}

// Call on page load
handlePaymentReturn();


// ─── Check What the User Has Purchased ───

async function refreshSubscriptionStatus() {
  try {
    const checkStatus = functions.httpsCallable("checkSubscriptionStatus");
    const result = await checkStatus();
    const status = result.data;

    // Update your UI based on what they own
    if (status.customizationPass) {
      unlockCustomizationFeatures();  // Your existing function
    }
    if (status.premiumPass) {
      unlockPremiumFeatures();        // Your existing function
    }

  } catch (error) {
    console.error("Status check error:", error);
  }
}
```

### Wire up your gamepass buttons:

```html
<!-- In your gamepass-screen -->
<button data-product="customization_pass" onclick="buyProduct('customization_pass')">
  Buy Customization Pass — $2.99
</button>

<button data-product="premium_monthly" onclick="buyProduct('premium_monthly')">
  Subscribe Monthly — $4.99/mo
</button>

<button data-product="premium_yearly" onclick="buyProduct('premium_yearly')">
  Subscribe Yearly — $49.99/yr (Save $10!)
</button>
```

---

## Step 6: Test Everything in Sandbox

Before going live, test with Payoneer's sandbox:

1. Make sure `PAYONEER_API_BASE` in your Cloud Function points to `https://api.sandbox.oscato.com/api/`
2. Use your sandbox credentials
3. Payoneer provides test card numbers for sandbox:
   - **Visa:** 4111 1111 1111 1111
   - **Mastercard:** 5500 0000 0000 0004
   - Expiry: any future date
   - CVV: any 3 digits
4. Walk through the full flow: click buy → redirect → enter test card → confirm → check Firestore

### Test the webhook locally with Firebase Emulators:

```bash
cd ~/multiply
firebase emulators:start --only functions
```

This runs your functions locally so you can test without deploying. Use a tool like [ngrok](https://ngrok.com) to expose your local webhook to the internet so Payoneer's sandbox can reach it.

---

## Step 7: Go Live

When everything works in sandbox:

1. Update `PAYONEER_API_BASE` to `https://api.live.oscato.com/api/`
2. Set your live credentials:
   ```bash
   firebase functions:secrets:set PAYONEER_MERCHANT_CODE    # live value
   firebase functions:secrets:set PAYONEER_PAYMENT_TOKEN     # live value
   ```
3. Update the `notificationUrl` to your production Cloud Function URL
4. Redeploy: `firebase deploy --only functions`
5. Change your gamepass buttons from "Coming Soon" to active
6. Make a real $2.99 test purchase with your own card to confirm end-to-end

---

## Firestore Data Structure

Here's what gets stored:

```
/transactions/{transactionId}
  ├── userId: "abc123"
  ├── productId: "customization_pass"
  ├── amount: 2.99
  ├── currency: "USD"
  ├── status: "pending" → "completed" or "failed"
  ├── payoneerListId: "..."
  ├── payoneerStatus: "charged"
  ├── createdAt: Timestamp
  └── completedAt: Timestamp

/users/{uid}
  ├── customizationPass: true/false
  ├── customizationPassPurchasedAt: Timestamp
  ├── premiumPass: true/false
  ├── premiumPlan: "monthly" or "yearly"
  ├── premiumExpiry: Timestamp
  └── premiumPurchasedAt: Timestamp
```

---

## Important Notes

### Subscriptions / Recurring Payments
Payoneer Checkout supports one-time payments out of the box. For true recurring subscriptions (auto-charge every month), you have two options:
- **Payoneer's recurring API** — if available on your account, lets you store a payment token and charge again later
- **Manual renewal** — prompt users to re-purchase when their premium expires. Simpler to implement, and fine for a small site

For now, I'd recommend starting with manual renewal. You can check `premiumExpiry` on each login and show a "Renew" button when it's about to expire.

### Security Checklist
- Never expose your Merchant Code or Payment Token in client-side code
- Always validate the product ID server-side (the Cloud Function does this)
- Always verify webhooks come from Payoneer (check the request signature)
- Use Firestore security rules so users can't modify their own `customizationPass` or `premiumPass` fields directly

### Costs
- **Firebase Cloud Functions:** Free for first 2M invocations/month
- **Firestore:** Free for first 50K reads/day and 20K writes/day
- **Payoneer Checkout:** Up to 3.99% per transaction, $1 minimum fee, 0% FX on USD
- **New merchant promo:** First $60K in payments fee-free

### The Money Path
Customer pays with card → money lands in your Payoneer balance → you spend it with your Payoneer Mastercard at ATMs/stores in the Bahamas, or use it online anywhere Mastercard is accepted.
