# 🌐 REST API Specification

This document details the complete API contract, HTTP methods, headers, query parameters, request bodies, and responses for the **Mini E-Commerce Demo Project**.

---

## 1. General Standards

- **Base URL**: `http://localhost:5000/api`
- **Content-Type**: `application/json`
- **Authentication**: JWT Bearer token passed in the `Authorization` header:
  ```http
  Authorization: Bearer <jwt_token>
  ```
- **Error Response Standard**:
  ```json
  {
    "success": false,
    "message": "Error description message"
  }
  ```

---

## 2. Authentication APIs (`/api/auth`)

### 2.1 Register Customer
Registers a new customer account.

- **Route**: `POST /api/auth/register`
- **Access**: Public
- **Request Body**:
  ```json
  {
    "name": "Jane Doe",
    "email": "jane@example.com",
    "password": "secretpassword",
    "confirmPassword": "secretpassword"
  }
  ```
- **Validation**:
  - `name`: Required, 2-50 characters.
  - `email`: Required, valid email format, must be unique.
  - `password`: Required, minimum 6 characters.
  - `confirmPassword`: Must match `password`.
- **Response `201 Created`**:
  ```json
  {
    "success": true,
    "message": "Customer registered successfully",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "650c1f1e2f1b2c001f8d1234",
      "name": "Jane Doe",
      "email": "jane@example.com",
      "role": "customer"
    }
  }
  ```
- **Error Response `400 Bad Request`**:
  ```json
  {
    "success": false,
    "message": "User with this email already exists"
  }
  ```

---

### 2.2 Login (Customer & Admin)
Authenticates a user and returns a signed JWT token containing their user ID and role.

- **Route**: `POST /api/auth/login`
- **Access**: Public
- **Request Body**:
  ```json
  {
    "email": "jane@example.com",
    "password": "secretpassword"
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "message": "Login successful",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "650c1f1e2f1b2c001f8d1234",
      "name": "Jane Doe",
      "email": "jane@example.com",
      "role": "customer" // or "admin"
    }
  }
  ```
- **Error Response `401 Unauthorized`**:
  ```json
  {
    "success": false,
    "message": "Invalid email or password"
  }
  ```

---

## 3. Categories APIs (`/api/categories`)

### 3.1 Get All Categories
Retrieves the list of all product categories.

- **Route**: `GET /api/categories`
- **Access**: Public
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "count": 3,
    "categories": [
      {
        "_id": "650c20aa2f1b2c001f8d2001",
        "name": "Electronics",
        "description": "Smartphones, laptops, and smart gadgets",
        "createdAt": "2026-10-01T10:00:00.000Z"
      },
      {
        "_id": "650c20aa2f1b2c001f8d2002",
        "name": "Fashion",
        "description": "Apparel, clothing, and daily wear",
        "createdAt": "2026-10-01T10:05:00.000Z"
      }
    ]
  }
  ```

---

### 3.2 Create Category
Creates a new product category.

- **Route**: `POST /api/categories`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "name": "Shoes",
    "description": "Footwear, athletic sneakers, and formal shoes"
  }
  ```
- **Response `201 Created`**:
  ```json
  {
    "success": true,
    "message": "Category created successfully",
    "category": {
      "_id": "650c20aa2f1b2c001f8d2003",
      "name": "Shoes",
      "description": "Footwear, athletic sneakers, and formal shoes",
      "createdAt": "2026-10-02T13:00:00.000Z"
    }
  }
  ```

---

### 3.3 Update Category
Modifies an existing category.

- **Route**: `PUT /api/categories/:id`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "name": "Footwear & Shoes",
    "description": "Updated category description"
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "message": "Category updated successfully",
    "category": {
      "_id": "650c20aa2f1b2c001f8d2003",
      "name": "Footwear & Shoes",
      "description": "Updated category description"
    }
  }
  ```

---

### 3.4 Delete Category
Removes an existing category.

- **Route**: `DELETE /api/categories/:id`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "message": "Category deleted successfully"
  }
  ```

---

## 4. Products APIs (`/api/products`)

### 4.1 Get All Products (Filter & Search)
Fetches products with optional category filtering and search queries.

- **Route**: `GET /api/products`
- **Access**: Public
- **Query Parameters**:
  - `category` *(optional)*: Category ID or slug/name (e.g., `?category=electronics`)
  - `search` *(optional)*: Text keyword matching title or description (e.g., `?search=phone`)
- **Example Request**:
  `GET /api/products?category=electronics&search=phone`
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "count": 1,
    "products": [
      {
        "_id": "650c30bb2f1b2c001f8d3001",
        "name": "Smartphone Pro Max",
        "description": "High-end smartphone with OLED display and triple camera setup.",
        "price": 899.99,
        "image": "https://images.unsplash.com/photo-1511707171634-5f897ff02aa9?w=500",
        "category": {
          "_id": "650c20aa2f1b2c001f8d2001",
          "name": "Electronics"
        },
        "stock": 25,
        "createdAt": "2026-10-01T12:00:00.000Z"
      }
    ]
  }
  ```

---

### 4.2 Get Single Product
Fetches detailed information for a single product.

- **Route**: `GET /api/products/:id`
- **Access**: Public
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "product": {
      "_id": "650c30bb2f1b2c001f8d3001",
      "name": "Smartphone Pro Max",
      "description": "High-end smartphone with OLED display and triple camera setup.",
      "price": 899.99,
      "image": "https://images.unsplash.com/photo-1511707171634-5f897ff02aa9?w=500",
      "category": {
        "_id": "650c20aa2f1b2c001f8d2001",
        "name": "Electronics"
      },
      "stock": 25
    }
  }
  ```

---

### 4.3 Create Product
Creates a new product document.

- **Route**: `POST /api/products`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "name": "Wireless Noise Cancelling Headphones",
    "description": "Premium over-ear headphones with 30-hour battery life.",
    "price": 199.99,
    "image": "https://images.unsplash.com/photo-1505740420928-5e560c06d30e?w=500",
    "category": "650c20aa2f1b2c001f8d2001",
    "stock": 40
  }
  ```
- **Response `201 Created`**:
  ```json
  {
    "success": true,
    "message": "Product created successfully",
    "product": {
      "_id": "650c30bb2f1b2c001f8d3002",
      "name": "Wireless Noise Cancelling Headphones",
      "price": 199.99,
      "stock": 40
    }
  }
  ```

---

### 4.4 Update Product
Updates an existing product's details, pricing, or stock level.

- **Route**: `PUT /api/products/:id`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "name": "Wireless Noise Cancelling Headphones (v2)",
    "price": 179.99,
    "stock": 50
  }
  ```
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "message": "Product updated successfully",
    "product": {
      "_id": "650c30bb2f1b2c001f8d3002",
      "name": "Wireless Noise Cancelling Headphones (v2)",
      "price": 179.99,
      "stock": 50
    }
  }
  ```

---

### 4.5 Delete Product
Removes a product from catalog.

- **Route**: `DELETE /api/products/:id`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "message": "Product deleted successfully"
  }
  ```

---

## 5. Orders APIs (`/api/orders` & `/api/admin/orders`)

### 5.1 Place Order
Validates inventory, re-calculates prices from DB, decreases stock, and saves order.

- **Route**: `POST /api/orders`
- **Access**: Protected (Customer)
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "products": [
      {
        "productId": "650c30bb2f1b2c001f8d3001",
        "quantity": 2
      }
    ],
    "shippingAddress": {
      "name": "Jane Doe",
      "phone": "+1 555-0199",
      "address": "456 Market St, Apt 7B",
      "city": "Metropolis",
      "pincode": "94016"
    }
  }
  ```
- **Response `201 Created`**:
  ```json
  {
    "success": true,
    "message": "Order placed successfully",
    "order": {
      "_id": "650c40cc2f1b2c001f8d4001",
      "user": "650c1f1e2f1b2c001f8d1234",
      "products": [
        {
          "product": "650c30bb2f1b2c001f8d3001",
          "name": "Smartphone Pro Max",
          "price": 899.99,
          "quantity": 2
        }
      ],
      "totalAmount": 1799.98,
      "shippingAddress": {
        "name": "Jane Doe",
        "phone": "+1 555-0199",
        "address": "456 Market St, Apt 7B",
        "city": "Metropolis",
        "pincode": "94016"
      },
      "paymentMethod": "Cash on Delivery",
      "status": "Pending",
      "createdAt": "2026-10-02T13:10:00.000Z"
    }
  }
  ```
- **Error Response `400 Bad Request` (Stock Exceeded)**:
  ```json
  {
    "success": false,
    "message": "Insufficient stock for product 'Smartphone Pro Max'. Only 1 available."
  }
  ```

---

### 5.2 Get Customer's Orders
Fetches all orders placed by the authenticated customer.

- **Route**: `GET /api/orders/my-orders`
- **Access**: Protected (Customer)
- **Headers**: `Authorization: Bearer <token>`
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "count": 1,
    "orders": [
      {
        "_id": "650c40cc2f1b2c001f8d4001",
        "products": [
          {
            "product": "650c30bb2f1b2c001f8d3001",
            "name": "Smartphone Pro Max",
            "price": 899.99,
            "quantity": 2
          }
        ],
        "totalAmount": 1799.98,
        "paymentMethod": "Cash on Delivery",
        "status": "Pending",
        "createdAt": "2026-10-02T13:10:00.000Z"
      }
    ]
  }
  ```

---

### 5.3 Get All Orders (Admin)
Retrieves all customer orders with customer user info and shipping details.

- **Route**: `GET /api/admin/orders`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "count": 1,
    "orders": [
      {
        "_id": "650c40cc2f1b2c001f8d4001",
        "user": {
          "_id": "650c1f1e2f1b2c001f8d1234",
          "name": "Jane Doe",
          "email": "jane@example.com"
        },
        "products": [
          {
            "product": "650c30bb2f1b2c001f8d3001",
            "name": "Smartphone Pro Max",
            "price": 899.99,
            "quantity": 2
          }
        ],
        "totalAmount": 1799.98,
        "shippingAddress": {
          "name": "Jane Doe",
          "phone": "+1 555-0199",
          "address": "456 Market St, Apt 7B",
          "city": "Metropolis",
          "pincode": "94016"
        },
        "paymentMethod": "Cash on Delivery",
        "status": "Pending",
        "createdAt": "2026-10-02T13:10:00.000Z"
      }
    ]
  }
  ```

---

### 5.4 Update Order Status (Admin)
Updates the fulfillment lifecycle stage of an order.

- **Route**: `PATCH /api/admin/orders/:id/status`
- **Access**: Protected (Admin Only)
- **Headers**: `Authorization: Bearer <token>`
- **Request Body**:
  ```json
  {
    "status": "Shipped"
  }
  ```
- **Allowed Status Values**:
  `"Pending"`, `"Confirmed"`, `"Shipped"`, `"Delivered"`, `"Cancelled"`
- **Response `200 OK`**:
  ```json
  {
    "success": true,
    "message": "Order status updated to Shipped",
    "order": {
      "_id": "650c40cc2f1b2c001f8d4001",
      "status": "Shipped",
      "updatedAt": "2026-10-02T13:30:00.000Z"
    }
  }
  ```
- **Error Response `400 Bad Request`**:
  ```json
  {
    "success": false,
    "message": "Invalid status value provided"
  }
  ```
