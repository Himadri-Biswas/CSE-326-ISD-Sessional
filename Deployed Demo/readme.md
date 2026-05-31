<div align="center">

# Amazon — Full-Stack E-Commerce Platform

### A production-grade online marketplace with dual customer & seller interfaces, Stripe payments, and role-based access control

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Visit%20Site-orange?style=for-the-badge&logo=vercel)](https://amazon-frontend-rosy.vercel.app/)
[![Backend Repo](https://img.shields.io/badge/Backend-GitHub-181717?style=for-the-badge&logo=github)](https://github.com/Himadri-Biswas/amazon-backend)
[![Frontend Repo](https://img.shields.io/badge/Frontend-GitHub-181717?style=for-the-badge&logo=github)](https://github.com/Himadri-Biswas/amazon-frontend)

![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-E0234E?style=flat-square&logo=nestjs&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat-square&logo=react&logoColor=black)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat-square&logo=mongodb&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Stripe](https://img.shields.io/badge/Stripe-635BFF?style=flat-square&logo=stripe&logoColor=white)
![Clerk](https://img.shields.io/badge/Clerk-6C47FF?style=flat-square&logo=clerk&logoColor=white)
![Vercel](https://img.shields.io/badge/Vercel-000000?style=flat-square&logo=vercel&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)

</div>

---

## Table of Contents

- [Overview](#overview)
- [Live Demo](#live-demo)
- [Feature Walkthrough](#feature-walkthrough)
  - [Customer Features](#customer-features)
  - [Seller Features](#seller-features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Backend Deep Dive](#backend-deep-dive)
  - [Project Structure](#backend-structure)
  - [Authentication & Authorization](#authentication--authorization)
  - [Database Design](#database-design)
  - [Payment Integration](#payment-integration)
  - [API Reference](#api-reference)
- [Frontend Deep Dive](#frontend-deep-dive)
  - [Project Structure](#frontend-structure)
  - [State Management](#state-management)
  - [API Layer](#api-layer)
  - [UI & Design System](#ui--design-system)
- [CI/CD Pipeline](#cicd-pipeline)
- [Environment Variables](#environment-variables)
- [Getting Started](#getting-started)

---

## Overview

Amazon is a full-stack e-commerce web application built with a **NestJS** backend and a **React + Vite** frontend. The platform supports two distinct user roles — **customers** who browse, purchase, and review products, and **sellers** who list products, manage inventory, and fulfill orders.

Under the hood, the system integrates **Clerk** for enterprise-grade authentication with JWT-based session verification, **Stripe** for secure online payment processing, **MongoDB Atlas** as the cloud database, and is deployed serverlessly on **Vercel** with a **GitHub Actions** CI/CD pipeline for automated testing on every push.

The backend is architected following NestJS's opinionated module pattern — each domain (Products, Orders, Cart, Reviews, etc.) is an isolated feature module with its own Controller, Service, Schema, and DTOs. This ensures clean separation of concerns, easy testability, and scalability as the application grows.

---

## Live Demo

> **Try the full application live:**
>
> [https://amazon-frontend-rosy.vercel.app/](https://amazon-frontend-rosy.vercel.app/)
>
> You can sign up as a customer and browse products, add items to cart, save wishlists, and place orders. To explore the seller dashboard, a seller role must be assigned via Clerk's admin panel.

---

## Feature Walkthrough

### Customer Features

#### 1. Authentication & Profile
- Sign up / sign in via **Clerk** (supports email/password and social OAuth)
- User profile page with quick-links to orders, wishlist, and addresses
- Automatic backend user sync on first login — Clerk's `userId` is stored as the primary key in MongoDB, linking the authentication layer to the data layer without storing passwords

#### 2. Product Discovery
- **Home page** — Hero carousel, category grid, best-sellers section, and popular products with purchase-momentum ranking (`boughtInLastMonth` metric)
- **All Products page** — Full catalog with server-side filtering by category, price range, star rating, and best-seller status; sortable by price, rating, review count, and popularity
- **Search** — Powered by a **MongoDB compound text index** on `title` and `category_name` with case-insensitive regex fallback; search state is stored in URL params so results are shareable and bookmark-friendly
- **Product Detail page** — Full product info, image, pricing with original vs. offer price discount percentage, star rating, related products by category, and complete review section with rating distribution histogram

#### 3. Cart & Wishlist
- Cart persisted to the database — synced across sessions and devices
- Cart stored as a sparse map `{ productId: quantity }` directly on the user document for O(1) lookups; quantity of `0` auto-removes the item
- Optimistic UI updates — the cart count in the navbar updates instantly while the API call runs in the background, reverting on error for a snappy feel
- Wishlist with toggle support; a dedicated `GET /wishlist/:productId/check` endpoint allows the product card to instantly reflect a product's wishlist state without loading the entire wishlist

#### 4. Checkout & Orders
- **Multi-step checkout** — address selection (or add new), payment method selection (Online / Cash on Delivery), and order review with live price summary
- Orders are created with a **2% tax** applied server-side and stored immutably in the `amount` field at time of purchase — price changes after ordering never affect historical records
- **Stripe online payment** — checkout session created server-side with line items per product + a separate tax line item; on success, Stripe redirects back with a `session_id` which the frontend verifies against the backend before updating the order's `paymentStatus` to `Paid`
- **Cash on Delivery** — order placed immediately with `paymentStatus: Pending`; user can initiate online payment later from the order detail page
- Order cancellation allowed only in `Pending` or `Confirmed` states — the backend enforces this transition guard

#### 5. Order Tracking
- Full order lifecycle: `Pending → Confirmed → Processing → Shipped → Delivered`
- Every status transition is appended to a `statusHistory` array with a timestamp and optional note — gives customers a visible timeline of their order's journey
- Estimated delivery date auto-calculated 5 days from the `Shipped` timestamp

#### 6. Reviews
- Only customers with a `Delivered` order containing the specific product can leave a review — **verified purchase enforcement** is done server-side by querying the `orders` collection
- One review per user per product enforced by a **unique compound MongoDB index** on `{ userId, productId }`
- 1–5 star rating with title and comment; sortable by newest, highest-rated, or lowest-rated
- Rating distribution (1-star through 5-star counts) calculated via a MongoDB aggregation pipeline on the `reviews` collection

#### 7. Address Management
- Full CRUD for shipping addresses with a default address concept
- When a new default is set, the backend atomically updates all other addresses for that user to `isDefault: false` in a single query before setting the new default — preventing race conditions
- Deleting the default address automatically promotes the next available address as default

---

### Seller Features

#### 1. Role-Based Access
- Seller role is stored in **Clerk's `publicMetadata`** (`publicMetadata.role === 'seller'`) — this is set in the Clerk admin dashboard and is cryptographically signed into the JWT, so it cannot be spoofed by a client
- A custom **`SellerGuard`** on the backend extends `ClerkAuthGuard` and additionally validates the role claim — any attempt to hit seller endpoints without the role returns `403 Forbidden`
- The seller dashboard is surfaced in the navbar only when `user.publicMetadata.role === 'seller'`

#### 2. Seller Dashboard
- At-a-glance statistics: total revenue, total orders, active product count, and pending orders requiring attention
- Recent orders table showing the last 5 transactions with quick-links to order detail

#### 3. Product Management
- Create, update, and delete product listings
- Seller-created products get an auto-generated ASIN (`SELLER-{timestamp}`) and an auto-incremented numeric product ID via a `findOne().sort({ id: -1 })` query at creation time
- Fields: title, category, list price, offer price, image URL, stock quantity, best-seller flag
- Image URL preview is rendered live in the Add Product form — sellers can verify the image before saving

#### 4. Inventory & Stock
- Inline stock editing directly from the product grid — no need to navigate to a full edit form for a quick stock update
- **Low stock badge** surfaces automatically when `stock ≤ 5` so sellers can replenish before a stockout

#### 5. Order Fulfillment
- View all orders that include their products
- Update order status through the pipeline (`Confirmed → Processing → Shipped → Delivered`)
- Add tracking numbers and notes at each status transition, which are appended to the order's `statusHistory` and visible to the customer

---

## Tech Stack

### Backend

| Layer | Technology | Version |
|---|---|---|
| Framework | NestJS | 10.4.0 |
| Language | TypeScript | 5.5.0 |
| Database | MongoDB + Mongoose | 8.5.0 |
| Authentication | Clerk (`@clerk/backend`) | 1.20.0 |
| Payments | Stripe | 14.25.0 |
| Validation | class-validator / class-transformer | 0.14.1 / 0.5.1 |
| API Docs | Swagger / swagger-ui-express | 7.4.0 / 5.0.1 |
| Config | @nestjs/config | 3.2.0 |
| Testing | Jest + ts-jest | 29.7.0 / 29.1.2 |
| Deployment | Vercel (serverless) | — |
| CI/CD | GitHub Actions | — |

### Frontend

| Layer | Technology | Version |
|---|---|---|
| Framework | React | 18.3.1 |
| Build Tool | Vite | 5.4.0 |
| Routing | React Router DOM | 6.28.0 |
| Authentication | Clerk (`@clerk/clerk-react`) | 5.15.0 |
| Payments | `@stripe/stripe-js` | 2.4.0 |
| HTTP Client | Axios | 1.7.0 |
| Styling | Tailwind CSS | 3.4.14 |
| Icons | lucide-react | 0.460.0 |
| Notifications | react-hot-toast | 2.4.1 |
| Testing | Vitest | 1.6.0 |
| Deployment | Vercel | — |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         BROWSER / CLIENT                        │
│                                                                 │
│   React + Vite SPA  ──►  Clerk (auth)  ──►  Stripe (payments)  │
│          │                                                      │
└──────────┼──────────────────────────────────────────────────────┘
           │  HTTPS + Bearer JWT
           ▼
┌──────────────────────────────────────────────────────────────────┐
│                       BACKEND (NestJS)                           │
│                                                                  │
│   ┌─────────────┐   ┌─────────────────────────────────────────┐ │
│   │ ClerkAuth   │   │          Feature Modules                │ │
│   │   Guard     │──►│  Auth │ Products │ Cart │ Orders │ ...  │ │
│   │ SellerGuard │   └─────────────────────────────────────────┘ │
│   └─────────────┘                    │                          │
│                                      ▼                          │
│                            Mongoose ODM                         │
└──────────────────────────────────────┼──────────────────────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │  MongoDB Atlas   │
                              │  (Cloud DB)      │
                              └─────────────────┘
```

Both the frontend and backend are deployed as **serverless functions on Vercel**. The backend's `vercel.json` routes all incoming requests to a single `api/index.ts` entry point:

```json
{
  "builds": [{ "src": "api/index.ts", "use": "@vercel/node" }],
  "routes": [{ "src": "/(.*)", "dest": "api/index.ts" }]
}
```

The `api/index.ts` bootstraps a NestJS application and caches the initialized app instance across invocations — this is the standard **singleton pattern** for serverless NestJS, dramatically reducing cold-start latency on subsequent requests within the same Vercel function instance.

---

## Backend Deep Dive

> **Repository:** [github.com/Himadri-Biswas/amazon-backend](https://github.com/Himadri-Biswas/amazon-backend)

### Backend Structure

```
src/
├── common/
│   ├── decorators/
│   │   └── current-user.decorator.ts   # @CurrentUser() param decorator
│   └── guards/
│       ├── clerk-auth.guard.ts          # JWT verification via Clerk SDK
│       └── seller.guard.ts             # Extends ClerkAuthGuard + role check
└── modules/
    ├── auth/           # User sync endpoint (Clerk → MongoDB)
    ├── users/          # Profile read/update
    ├── products/       # Catalog CRUD + seller listing
    ├── cart/           # Cart operations
    ├── orders/         # Order lifecycle + seller fulfillment
    ├── addresses/      # Shipping address management
    ├── reviews/        # Product reviews with verified-purchase check
    ├── wishlist/       # Wishlist toggle/list
    └── payments/       # Stripe checkout session + verification
```

Each module follows NestJS convention:
```
products/
├── products.controller.ts   # Route handlers, decorators
├── products.service.ts      # Business logic, MongoDB queries
├── products.schema.ts       # Mongoose schema + text index definition
└── dto/
    ├── create-product.dto.ts
    └── query-products.dto.ts
```

### Authentication & Authorization

Authentication is built on top of **Clerk's backend SDK**. Every protected request must include an `Authorization: Bearer <token>` header. The `ClerkAuthGuard` implements NestJS's `CanActivate` interface:

```typescript
// src/common/guards/clerk-auth.guard.ts
@Injectable()
export class ClerkAuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedException('No token provided');

    const { sub, ...sessionClaims } = await clerkClient.verifyToken(token);
    request.userId = sub;
    request.sessionClaims = sessionClaims;
    return true;
  }
}
```

The **`SellerGuard`** extends this, additionally checking the user's `publicMetadata`:

```typescript
// src/common/guards/seller.guard.ts
@Injectable()
export class SellerGuard extends ClerkAuthGuard {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    await super.canActivate(context);
    const request = context.switchToHttp().getRequest();
    const user = await clerkClient.users.getUser(request.userId);
    if (user.publicMetadata?.role !== 'seller') {
      throw new ForbiddenException('Seller access required');
    }
    return true;
  }
}
```

A custom **`@CurrentUser()` parameter decorator** cleanly extracts the `userId` from the request context inside any controller method:

```typescript
// src/common/decorators/current-user.decorator.ts
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): string => {
    const request = ctx.switchToHttp().getRequest();
    return request.userId;
  },
);
```

**Global `ValidationPipe`** is registered at the app level with strict settings:
```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,           // Strip unknown fields from request bodies
  forbidNonWhitelisted: true, // Throw 400 on unknown fields
  transform: true,           // Auto-coerce primitives to DTO types
}));
```

This means every incoming request body is automatically validated against its DTO class using `class-validator` decorators before the controller method is invoked — no manual validation needed in services.

### Database Design

The database uses **MongoDB Atlas** with Mongoose ODM. Here is the core schema design, highlighting the notable decisions:

**`users` collection**

The user's `_id` is set to their **Clerk user ID string** (not a MongoDB ObjectId). This creates a direct, join-free link between the Clerk auth layer and every MongoDB document that references a user:

```typescript
@Schema({ timestamps: true })
export class User {
  @Prop({ type: String, required: true }) // Clerk userId as primary key
  _id: string;

  @Prop({ type: Object, default: {} })
  cartItems: Record<string, number>;      // { productId: quantity } map

  @Prop({ type: [String], default: [] })
  wishlist: string[];
}
```

**`products` collection** — Text Index for Search

MongoDB's compound text index is defined directly in the schema to power the search endpoint:

```typescript
@Schema({ timestamps: true })
@index({ title: 'text', category_name: 'text' }) // Compound text index
export class Product {
  @Prop() asin: string;
  @Prop() title: string;
  @Prop() price: number;
  @Prop() listPrice: number;
  @Prop() stars: number;
  @Prop() isBestSeller: boolean;
  @Prop() boughtInLastMonth: number; // Purchase momentum metric
  @Prop({ default: 0 }) stock: number;
}
```

**`orders` collection** — Immutable Status History

Each status transition is appended (never mutated) to a `statusHistory` array, providing a full audit trail:

```typescript
@Schema({ timestamps: true })
export class Order {
  @Prop({ required: true }) userId: string;
  @Prop({ type: [OrderItemSchema] }) items: OrderItem[];
  @Prop({ required: true }) amount: number;       // Total with 2% tax, locked at creation
  @Prop({ type: String, enum: OrderStatus, default: 'Pending' }) status: string;
  @Prop({ type: String, enum: PaymentStatus, default: 'Pending' }) paymentStatus: string;
  @Prop() paymentId: string;                      // Stripe payment_intent ID
  @Prop() paidAt: Date;
  @Prop() trackingNumber: string;
  @Prop() estimatedDelivery: Date;                // Auto-calculated on 'Shipped'
  @Prop({ type: [StatusHistoryEntrySchema] })
  statusHistory: StatusHistoryEntry[];            // Append-only audit log
}
```

Index defined for efficient user order queries:
```typescript
OrderSchema.index({ userId: 1, createdAt: -1 });
```

**`reviews` collection** — One Review Per Product Per User

A unique compound index at the database level enforces the one-review constraint, regardless of how many concurrent requests might slip through:

```typescript
ReviewSchema.index({ userId: 1, productId: 1 }, { unique: true });
```

The **verified purchase check** is a real MongoDB query before review creation:
```typescript
const verifiedOrder = await this.orderModel.findOne({
  userId,
  'items.product': productId,
  status: 'Delivered',
});
if (!verifiedOrder) throw new ForbiddenException('Only verified purchasers can review');
```

### Payment Integration

The Stripe integration is built around Stripe's **Checkout Sessions** API. Here is the complete server-side payment flow:

**Step 1 — Create Checkout Session** (`POST /api/payments/checkout/:orderId`)

```typescript
async createCheckoutSession(orderId: string, userId: string) {
  const order = await this.orderModel.findById(orderId).populate('items.product');

  if (order.userId !== userId) throw new ForbiddenException();
  if (order.paymentMethod !== 'Online') throw new BadRequestException('COD order');

  const lineItems = order.items.map(item => ({
    price_data: {
      currency: process.env.CURRENCY || 'usd',
      product_data: { name: item.product.title, images: [item.product.imgUrl] },
      unit_amount: Math.round(item.price * 100), // Stripe expects cents
    },
    quantity: item.quantity,
  }));

  // 2% tax as a separate line item
  const taxAmount = Math.round(order.amount * 0.02 * 100);
  lineItems.push({
    price_data: {
      currency: 'usd',
      product_data: { name: 'Tax (2%)' },
      unit_amount: taxAmount,
    },
    quantity: 1,
  });

  const session = await this.stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: lineItems,
    mode: 'payment',
    success_url: `${frontend}/orders/${orderId}?payment=success&session_id={CHECKOUT_SESSION_ID}`,
    cancel_url:  `${frontend}/orders/${orderId}?payment=cancelled`,
    metadata: { orderId, userId },
  });

  return { sessionId: session.id, url: session.url };
}
```

**Step 2 — Verify Payment** (`POST /api/payments/verify`)

After Stripe redirects back with `?payment=success&session_id=xxx`, the frontend calls the verify endpoint:

```typescript
async verifyPayment(sessionId: string, orderId: string) {
  const session = await this.stripe.checkout.sessions.retrieve(sessionId);

  if (session.payment_status !== 'paid') {
    throw new BadRequestException('Payment not completed');
  }

  const updatedOrder = await this.orderModel.findByIdAndUpdate(
    orderId,
    {
      paymentStatus: 'Paid',
      paymentId: session.payment_intent,
      paidAt: new Date(),
      status: 'Confirmed',
      $push: {
        statusHistory: {
          status: 'Confirmed',
          timestamp: new Date(),
          note: 'Payment confirmed via Stripe',
        },
      },
    },
    { new: true },
  );

  return { success: true, order: updatedOrder };
}
```

The `$push` to `statusHistory` and the status update happen in a **single atomic `findByIdAndUpdate`** — no risk of partial updates.

### API Reference

| Module | Method | Endpoint | Auth |
|---|---|---|---|
| Auth | POST | `/api/auth/sync` | — |
| Users | GET | `/api/users/profile` | Customer |
| Users | PUT | `/api/users/profile` | Customer |
| Products | GET | `/api/products` | — |
| Products | GET | `/api/products/search` | — |
| Products | GET | `/api/products/categories` | — |
| Products | GET | `/api/products/best-sellers` | — |
| Products | GET | `/api/products/:id` | — |
| Products | GET | `/api/products/:id/related` | — |
| Products | POST | `/api/products` | **Seller** |
| Products | PUT | `/api/products/:id` | **Seller** |
| Products | DELETE | `/api/products/:id` | **Seller** |
| Cart | GET | `/api/cart` | Customer |
| Cart | POST | `/api/cart/update` | Customer |
| Cart | DELETE | `/api/cart/clear` | Customer |
| Orders | POST | `/api/orders` | Customer |
| Orders | GET | `/api/orders` | Customer |
| Orders | GET | `/api/orders/:id` | Customer |
| Orders | POST | `/api/orders/:id/cancel` | Customer |
| Orders | GET | `/api/orders/all` | **Seller** |
| Orders | PATCH | `/api/orders/:id/status` | **Seller** |
| Orders | GET | `/api/orders/stats` | **Seller** |
| Addresses | GET/POST | `/api/addresses` | Customer |
| Addresses | GET/PUT/DELETE | `/api/addresses/:id` | Customer |
| Reviews | POST | `/api/reviews` | Customer |
| Reviews | GET | `/api/reviews/product/:productId` | — |
| Reviews | PUT/DELETE | `/api/reviews/:id` | Customer |
| Wishlist | GET | `/api/wishlist` | Customer |
| Wishlist | POST/DELETE | `/api/wishlist/:productId` | Customer |
| Payments | POST | `/api/payments/checkout/:orderId` | Customer |
| Payments | POST | `/api/payments/verify` | Customer |

Full Swagger/OpenAPI documentation is auto-generated and served at `/api/docs` in development.

---

## Frontend Deep Dive

> **Repository:** [github.com/Himadri-Biswas/amazon-frontend](https://github.com/Himadri-Biswas/amazon-frontend)
> 
> **Live Application:** [amazon-frontend-rosy.vercel.app](https://amazon-frontend-rosy.vercel.app/)

### Frontend Structure

```
src/
├── api/
│   └── client.js          # Axios instance + auth interceptor + API modules
├── components/
│   ├── common/
│   │   ├── Navbar.jsx      # Search (Ctrl+K shortcut), cart badge, user menu
│   │   ├── ProductCard.jsx # Reusable card with discount badge, wishlist button
│   │   ├── Pagination.jsx
│   │   ├── StarRating.jsx  # Static display
│   │   └── Loading.jsx
│   └── layout/
│       ├── Layout.jsx       # Main layout (Navbar + Footer wrapper)
│       └── SellerLayout.jsx # Sidebar layout for seller dashboard
├── context/
│   └── AppContext.jsx       # Global cart + wishlist state with Clerk integration
├── pages/
│   ├── Home.jsx
│   ├── AllProducts.jsx      # URL-synced filters and pagination
│   ├── ProductDetail.jsx    # Reviews, related products, add to cart
│   ├── Cart.jsx
│   ├── Checkout.jsx         # Multi-step: address → payment → review
│   ├── MyOrders.jsx
│   ├── OrderDetail.jsx      # Status timeline + Stripe payment trigger
│   ├── Wishlist.jsx
│   ├── Search.jsx
│   ├── AddAddress.jsx
│   ├── Profile.jsx
│   └── seller/
│       ├── Dashboard.jsx
│       ├── SellerProducts.jsx   # Inline stock editing, low-stock badge
│       ├── AddProduct.jsx       # Form with live image preview + tips sidebar
│       ├── SellerOrders.jsx
│       └── SellerOrderDetail.jsx
└── utils/
    ├── helpers.js           # formatCurrency, getStatusColor, calculateDiscount
    └── helpers.test.js      # Vitest unit tests
```

### State Management

Global state is managed with **React Context** via `AppContext.jsx`. The context integrates directly with Clerk's hooks to stay in sync with authentication state:

```javascript
// src/context/AppContext.jsx
export const AppProvider = ({ children }) => {
  const { getToken } = useAuth();
  const { user } = useUser();
  const [cart, setCart] = useState({ items: [], total: 0, itemCount: 0 });
  const [wishlist, setWishlist] = useState([]);

  // Wire Clerk's token provider into the Axios instance once on mount
  useEffect(() => {
    setAuthTokenProvider(getToken);
  }, [getToken]);

  // Re-fetch cart and wishlist whenever the signed-in user changes
  useEffect(() => {
    if (user) {
      fetchCart();
      fetchWishlist();
      userAPI.syncUser({ id: user.id, name: user.fullName, email: user.primaryEmailAddress?.emailAddress });
    }
  }, [user]);

  const updateCart = async (productId, quantity) => {
    // Optimistic update: change local state immediately
    setCart(prev => optimisticallyUpdate(prev, productId, quantity));
    try {
      await cartAPI.update(productId, quantity);
    } catch {
      fetchCart(); // Revert on failure by re-fetching ground truth
      toast.error('Could not update cart');
    }
  };

  return (
    <AppContext.Provider value={{ cart, wishlist, updateCart, toggleWishlist, isInWishlist }}>
      {children}
    </AppContext.Provider>
  );
};
```

Each page manages its own loading states, filters, and pagination with local `useState` — only cart and wishlist (shared across the app) live in context. This avoids unnecessary re-renders.

**URL-synced filters** in `AllProducts.jsx` and `Search.jsx` mean every filter selection updates the URL via `useSearchParams`, making filtered views bookmarkable and preserving state on browser back navigation:

```javascript
const [searchParams, setSearchParams] = useSearchParams();
const category = searchParams.get('category') || '';
const sortBy = searchParams.get('sortBy') || 'boughtInLastMonth';

const handleCategoryChange = (val) => {
  setSearchParams(prev => { prev.set('category', val); prev.set('page', '1'); return prev; });
};
```

### API Layer

`src/api/client.js` exports a single **Axios instance** with two interceptors:

```javascript
const api = axios.create({ baseURL: import.meta.env.VITE_API_URL });

// Request interceptor — attach Clerk JWT to every request
api.interceptors.request.use(async (config) => {
  if (authTokenProvider) {
    const token = await authTokenProvider();
    if (token) config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor — normalize error format
api.interceptors.response.use(
  response => response,
  error => Promise.reject(error.response?.data || error.message)
);
```

All API calls are organized into typed modules (`productsAPI`, `cartAPI`, `ordersAPI`, etc.) exported from the same file — a single import gives any component access to the full API surface.

**Parallel data fetching** with `Promise.all` is used on data-heavy pages:

```javascript
// ProductDetail.jsx — load product, reviews, and related items in parallel
const [productRes, reviewsRes, relatedRes] = await Promise.all([
  productsAPI.getById(id),
  reviewsAPI.getForProduct(id),
  productsAPI.getRelated(id, 6),
]);
```

### UI & Design System

The UI is built entirely with **Tailwind CSS** using a custom Amazon-inspired color palette defined in `tailwind.config.js`:

```javascript
theme: {
  extend: {
    colors: {
      amazon: '#131921',          // Dark navy header
      'amazon-accent': '#febd69', // Yellow highlight
      primary: '#f97316',         // Orange CTA
    }
  }
}
```

Notable UX patterns implemented:

- **Keyboard shortcut** — pressing `Ctrl+K` or `/` anywhere on the page focuses the Navbar search input (Shneiderman's rule of user control and efficiency)
- **Toast notifications** via `react-hot-toast` for every async operation (cart updates, wishlist toggles, order actions) — immediate, non-blocking feedback
- **Progress stepper** in the Add Product form — a live completion indicator counts how many required fields are filled, guiding the seller through the form
- **Low stock badge** — surfaces on the seller product grid when `stock ≤ 5`, a simple threshold check that surfaces important inventory signals without overwhelming the interface
- **Order status timeline** — the `OrderDetail` page renders `statusHistory` as a vertical stepper with timestamps, giving customers a clear visual of their order's journey
- **Inline stock editing** — sellers can update product stock directly from the product grid without navigating to a full edit form, reducing friction for routine inventory tasks

---

## CI/CD Pipeline

Every push to `main`, `master`, or feature branches triggers a **GitHub Actions** workflow:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, master, mehedi, test]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20]
      fail-fast: false

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

`fail-fast: false` ensures all matrix entries complete even if one fails — useful for catching environment-specific issues without aborting the entire run early.

Production deployments are handled automatically by **Vercel's GitHub integration** — every merge to `main` triggers a production deploy, and every pull request gets its own preview deployment URL.

---

## Environment Variables

### Backend (`.env`)

```env
MONGODB_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/amazon
CLERK_SECRET_KEY=sk_live_...
CLERK_PUBLISHABLE_KEY=pk_live_...
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
FRONTEND_URL=https://amazon-frontend-rosy.vercel.app
PORT=3001
CURRENCY=usd
```

### Frontend (`.env`)

```env
VITE_CLERK_PUBLISHABLE_KEY=pk_live_...
VITE_API_URL=https://amazon-backend.vercel.app
```

---

## Getting Started

### Prerequisites
- Node.js 20+
- MongoDB Atlas account
- Clerk account
- Stripe account

### Backend

```bash
git clone https://github.com/Himadri-Biswas/amazon-backend
cd amazon-backend
npm install
cp .env.example .env     # Fill in your credentials
npm run start:dev        # Starts on http://localhost:3001
```

Swagger docs available at `http://localhost:3001/api/docs`

### Frontend

```bash
git clone https://github.com/Himadri-Biswas/amazon-frontend
cd amazon-frontend
npm install
# Create .env with VITE_CLERK_PUBLISHABLE_KEY and VITE_API_URL
npm run dev              # Starts on http://localhost:5173
```

### Running Tests

```bash
# Backend
npm test

# Frontend
npm run test
```

---

<div align="center">

Built with React, NestJS, MongoDB, Stripe & Clerk

[Live Demo](https://amazon-frontend-rosy.vercel.app/) · [Backend Repo](https://github.com/Himadri-Biswas/amazon-backend) · [Frontend Repo](https://github.com/Himadri-Biswas/amazon-frontend)

</div>
