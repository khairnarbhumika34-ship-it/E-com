# 🛡️ Validation & Security Specification

This document details the complete validation rules, security practices, cryptographic operations, and access control patterns implemented across the **Mini E-Commerce Demo Project (MERN Stack)**.

---

## 📋 Table of Contents
1. [Core Principles](#1-core-principles)
2. [Input Validation Matrix](#2-input-validation-matrix)
   - [Authentication & User Accounts](#authentication--user-accounts)
   - [Categories](#categories)
   - [Products](#products)
   - [Checkout & Orders](#checkout--orders)
3. [Server-Authoritative Pricing Architecture](#3-server-authoritative-pricing-architecture)
4. [Atomic Stock Management & Concurrency Control](#4-atomic-stock-management--concurrency-control)
5. [Authentication & Cryptography](#5-authentication--cryptography)
   - [Password Hashing with bcrypt](#password-hashing-with-bcrypt)
   - [JWT Design & Lifecycle](#jwt-design--lifecycle)
6. [Access Control Middleware Architecture](#6-access-control-middleware-architecture)
   - [Customer Authentication Guard (`authMiddleware`)](#customer-authentication-guard-authmiddleware)
   - [Admin Privilege Guard (`adminMiddleware`)](#admin-privilege-guard-adminmiddleware)
7. [Frontend Defense & UX Safeguards](#7-frontend-defense--ux-safeguards)

---

## 1. Core Principles

The application adheres to five non-negotiable security rules:
1. **Never Trust the Client**: All data received from client HTTP requests must be rigorously validated and sanitized on the server before processing or storing in MongoDB.
2. **Server-Side Price Calculation**: Product prices sent by the client are strictly ignored during checkout; prices are fetched directly from MongoDB to calculate line items and order totals.
3. **Atomic Stock Decrements**: Inventory checks and decrements are performed to ensure stock cannot dip below zero under concurrent traffic.
4. **Principle of Least Privilege**: Protected endpoints enforce strict role verification. Administrative actions require valid JWT authentication with an explicit `admin` role claim.
5. **Defense-in-Depth Validation**: Dual-layer validation with instant feedback on the client side (HTML5 + React state validation) backed by authoritative schema and middleware validation on the backend.

---

## 2. Input Validation Matrix

### Authentication & User Accounts

| Field | Client Validation | Server (Mongoose & Middleware) Validation | Error Message |
| :--- | :--- | :--- | :--- |
| `name` | Required, trimmed, min 2 chars | `String`, required, `trim: true`, `minlength: 2`, `maxlength: 50` | *"Name must be between 2 and 50 characters."* |
| `email` | Required, HTML5 email regex | `String`, required, `unique: true`, `lowercase: true`, regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` | *"Please enter a valid email address."* / *"User with this email already exists."* |
| `password` | Required, type `password`, min 6 chars | `String`, required, `minlength: 6` | *"Password must be at least 6 characters long."* |
| `confirmPassword` | Required, must match `password` | Checked in controller before hashing: `password === confirmPassword` | *"Passwords do not match."* |

---

### Categories

| Field | Client Validation | Server Validation | Error Message |
| :--- | :--- | :--- | :--- |
| `name` | Required, trimmed, min 2 chars | `String`, required, `unique: true`, `trim: true`, `maxlength: 50` | *"Category name is required and must be unique."* |
| `description` | Optional, string, max 255 chars | `String`, optional, `trim: true`, `maxlength: 255` | *"Description cannot exceed 255 characters."* |

---

### Products

| Field | Client Validation | Server Validation | Error Message |
| :--- | :--- | :--- | :--- |
| `name` | Required, trimmed | `String`, required, `trim: true`, `maxlength: 100` | *"Product name is required (max 100 characters)."* |
| `description` | Required, trimmed | `String`, required, `trim: true` | *"Product description is required."* |
| `price` | Required, type `number`, `step="0.01"`, `min="0.01"` | `Number`, required, `min: [0.01, 'Price must be positive']` | *"Price must be greater than zero."* |
| `image` | Required, valid URL or relative path | `String`, required, `trim: true` | *"Product image URL or path is required."* |
| `category` | Required, valid dropdown selection | `ObjectId`, required, `ref: 'Category'`, verified to exist in DB | *"Please select a valid category."* |
| `stock` | Required, integer, `min="0"` | `Number`, required, integer, `min: [0, 'Stock cannot be negative']` | *"Stock cannot be negative."* |

---

### Checkout & Orders

| Field | Client Validation | Server Validation | Error Message |
| :--- | :--- | :--- | :--- |
| `items` | Cart cannot be empty (`length > 0`) | Array of objects, required, `minlength: 1` | *"Cart is empty. Please add products to checkout."* |
| `items[].product`| Valid product identifier | `ObjectId`, required, verified to exist in DB | *"Invalid product reference in order."* |
| `items[].quantity`| Integer, `min="1"`, `max=stock` | Integer, required, `min: 1`, `<= current product.stock` | *"Quantity must be at least 1 and cannot exceed available stock."* |
| `shippingAddress.name` | Required, trimmed, min 2 chars | `String`, required, `trim: true` | *"Recipient name is required."* |
| `shippingAddress.phone`| Required, numeric, min 10 digits | `String`, required, regex `/^\+?[0-9]{10,15}$/` | *"Please enter a valid phone number."* |
| `shippingAddress.address`| Required, min 5 chars | `String`, required, `trim: true` | *"Street address is required."* |
| `shippingAddress.city` | Required, min 2 chars | `String`, required, `trim: true` | *"City is required."* |
| `shippingAddress.pincode`| Required, 4-10 alphanumeric/digits | `String`, required, `trim: true` | *"Valid postal/pincode is required."* |
| `paymentMethod`| Static display: `"Cash on Delivery"` | Fixed enum `['Cash on Delivery']` | *"Only Cash on Delivery is currently supported."* |

---

## 3. Server-Authoritative Pricing Architecture

### Vulnerability Addressed
Malicious actors can easily manipulate HTTP requests via browser dev tools or cURL, altering the price payload (e.g., changing a `$500` phone to `$1.00`).

### Implementation Strategy
1. The frontend submits **only** the `productId` and requested `quantity` inside `items`:
   ```json
   {
     "items": [
       { "product": "650c1f1e2f1b2c001f8d9999", "quantity": 2 }
     ],
     "shippingAddress": { ... }
   }
   ```
2. The server controller queries MongoDB for each referenced `productId`:
   ```javascript
   const productDocs = await Product.find({
     _id: { $in: items.map(item => item.product) }
   });
   ```
3. The server computes line totals and overall total amount:
   ```javascript
   let calculatedTotal = 0;
   const orderItems = items.map(item => {
     const dbProduct = productDocs.find(p => p._id.toString() === item.product);
     if (!dbProduct) {
       throw new Error(`Product ${item.product} not found`);
     }
     if (dbProduct.stock < item.quantity) {
       throw new Error(`Insufficient stock for product: ${dbProduct.name}`);
     }

     const lineTotal = dbProduct.price * item.quantity;
     calculatedTotal += lineTotal;

     return {
       product: dbProduct._id,
       name: dbProduct.name,
       price: dbProduct.price, // Stored as immutable snapshot
       quantity: item.quantity
     };
   });
   ```
4. The calculated total amount is saved into the new `Order` document. Any price sent from the client is completely disregarded.

---

## 4. Atomic Stock Management & Concurrency Control

### Vulnerability Addressed
If two customers attempt to purchase the last remaining item (`stock = 1`) simultaneously, naive read-then-write updates can result in negative stock (`stock = -1`).

### Implementation Strategy
Mongoose atomic query conditional updates ensure that stock decrements only occur if the available stock is greater than or equal to the requested quantity:

```javascript
// In Order Creation Transaction or Loop
const updatedProduct = await Product.findOneAndUpdate(
  {
    _id: item.product,
    stock: { $gte: item.quantity } // Condition: Available stock must be >= quantity
  },
  {
    $inc: { stock: -item.quantity } // Atomic decrement
  },
  { new: true }
);

if (!updatedProduct) {
  // Stock condition failed: either out of stock or race condition occurred
  return res.status(400).json({
    success: false,
    message: `Insufficient stock for product ID: ${item.product}`
  });
}
```

If any item fails the stock condition, the operation aborts and rollbacks any previously decremented items.

---

## 5. Authentication & Cryptography

### Password Hashing with bcrypt

- **Library**: `bcryptjs`
- **Salt Work Factor**: `10`
- **Pre-Save Hook**:
  ```javascript
  userSchema.pre('save', async function (next) {
    if (!this.isModified('password')) return next();
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  });
  ```
- **Password Verification**:
  ```javascript
  userSchema.methods.comparePassword = async function (enteredPassword) {
    return await bcrypt.compare(enteredPassword, this.password);
  };
  ```

---

### JWT Design & Lifecycle

- **Signing Algorithm**: HMAC SHA-256 (`HS256`)
- **Payload Schema**:
  ```json
  {
    "id": "650c1f1e2f1b2c001f8d1234",
    "role": "customer" // or "admin"
  }
  ```
- **Expiration Window**: `7d` (configurable via `JWT_EXPIRES_IN`)
- **Token Delivery**: Attached in JSON response payload upon successful login or registration; stored in client `localStorage`.
- **Token Consumption**: Passed via standard HTTP Bearer scheme:
  ```http
  Authorization: Bearer <token>
  ```

---

## 6. Access Control Middleware Architecture

### Customer Authentication Guard (`authMiddleware`)

Extracts and verifies the JWT token from incoming request headers:

```javascript
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

export const protect = async (req, res, next) => {
  let token;

  if (
    req.headers.authorization &&
    req.headers.authorization.startsWith('Bearer')
  ) {
    try {
      token = req.headers.authorization.split(' ')[1];
      const decoded = jwt.verify(token, process.env.JWT_SECRET);

      // Attach user object (excluding hashed password) to request
      req.user = await User.findById(decoded.id).select('-password');
      if (!req.user) {
        return res.status(401).json({ success: false, message: 'User no longer exists.' });
      }

      return next();
    } catch (error) {
      return res.status(401).json({ success: false, message: 'Invalid or expired token.' });
    }
  }

  if (!token) {
    return res.status(401).json({ success: false, message: 'Not authorized, no token provided.' });
  }
};
```

---

### Admin Privilege Guard (`adminMiddleware`)

Verifies that the authenticated user possesses administrator rights:

```javascript
export const adminOnly = (req, res, next) => {
  if (req.user && req.user.role === 'admin') {
    return next();
  }
  return res.status(403).json({
    success: false,
    message: 'Access denied. Administrative privileges required.'
  });
};
```

#### Route Protection Example:
```javascript
// Public route
router.get('/products', getProducts);

// Customer protected route
router.post('/orders', protect, createOrder);

// Admin protected route
router.post('/products', protect, adminOnly, createProduct);
router.patch('/admin/orders/:id/status', protect, adminOnly, updateOrderStatus);
```

---

## 7. Frontend Defense & UX Safeguards

1. **State-Level Guards**:
   - `ProtectedRoute`: Redirects unauthenticated users to `/login` if attempting to reach `/cart`, `/checkout`, or `/my-orders`.
   - `AdminRoute`: Redirects non-admin users to `/` if attempting to reach `/admin/*`.
2. **Stock-Aware UI**:
   - Increment button disabled when `quantity >= product.stock`.
   - "Out of Stock" badge rendered when `product.stock === 0`.
   - Add to Cart button disabled when `stock === 0`.
3. **Form Validation UX**:
   - Live regex checks for email format.
   - Password confirmation equality matching before button enabling.
   - Clear and friendly inline error messages.
4. **Delete Confirmation Modals**:
   - Prevents accidental deletion of categories and products with custom interactive confirmation dialogs.
