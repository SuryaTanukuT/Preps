# MVC (Model-View-Controller) Pattern

## Definition
**MVC** is a architectural pattern that separates an application into three interconnected components:
- **Model**: Manages data and business logic
- **View**: Handles presentation and user interface
- **Controller**: Processes user input and coordinates between Model and View

This separation enables **separation of concerns**, making applications easier to maintain, test, and scale.

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │           USER ACTION               │
                    └────────────────┬────────────────────┘
                                    │
                    ┌────────────────▼────────────────────┐
                    │            CONTROLLER               │
                    │    (Processes input, coordinates)   │
                    └────────┬───────────────┬────────────┘
                            │               │
              ┌─────────────▼─────┐   ┌─────▼─────────────┐
              │       MODEL       │   │       VIEW        │
              │ (Data, Business   │   │  (Presentation)   │
              │  Logic, Database) │   │                   │
              └─────────────┬─────┘   └─────┬─────────────┘
                            │               │
                    ┌───────▼───────────────▼───────┐
                    │         RENDERS VIEW           │
                    └─────────────────────────────────┘
```

**Flow:**
1. User interacts with **View** (clicks button)
2. **Controller** receives request
3. **Controller** updates **Model**
4. **Model** notifies **View** of changes
5. **View** updates presentation

---

## Good vs Bad Example

### Bad Example (No MVC - Mixed Concerns)
```javascript
// BAD: Everything mixed together
app.get('/users/:id', async (req, res) => {
  // Database query (Model) mixed with Controller
  const user = await db.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
  
  // Business logic (Model) in Controller
  if (user.role === 'admin') {
    user.permissions = ['read', 'write', 'delete'];
  }
  
  // HTML generation (View) in Controller
  res.send(`
    <html>
      <body>
        <h1>${user.name}</h1>
        <p>Email: ${user.email}</p>
      </body>
    </html>
  `);
});

// Problems:
// - Can't reuse database queries
// - Can't change HTML without touching controller
// - Can't test business logic separately
// - Violates separation of concerns
```

### Good Example (MVC)
```javascript
// ============ MODEL ============
// models/User.js
class User {
  constructor(data) {
    this.id = data.id;
    this.name = data.name;
    this.email = data.email;
    this.role = data.role;
    this.createdAt = data.createdAt;
  }
  
  // Business logic
  isAdmin() {
    return this.role === 'admin';
  }
  
  getPermissions() {
    const basePermissions = ['read'];
    
    if (this.isAdmin()) {
      return [...basePermissions, 'write', 'delete', 'manage_users'];
    }
    
    if (this.role === 'editor') {
      return [...basePermissions, 'write', 'edit'];
    }
    
    return basePermissions;
  }
  
  getDisplayName() {
    return this.name || this.email.split('@')[0];
  }
  
  toJSON() {
    return {
      id: this.id,
      name: this.name,
      email: this.email,
      role: this.role,
      displayName: this.getDisplayName(),
      permissions: this.getPermissions()
    };
  }
}

// models/UserRepository.js
class UserRepository {
  constructor(db) {
    this.db = db;
  }
  
  async findById(id) {
    const result = await this.db.query(
      'SELECT * FROM users WHERE id = $1',
      [id]
    );
    
    if (!result.rows[0]) return null;
    
    return new User(result.rows[0]);
  }
  
  async findByEmail(email) {
    const result = await this.db.query(
      'SELECT * FROM users WHERE email = $1',
      [email]
    );
    
    return result.rows.map(row => new User(row));
  }
  
  async create(userData) {
    const result = await this.db.query(
      `INSERT INTO users (name, email, role, created_at) 
       VALUES ($1, $2, $3, $4) RETURNING *`,
      [userData.name, userData.email, userData.role || 'user', new Date()]
    );
    
    return new User(result.rows[0]);
  }
  
  async update(id, userData) {
    const result = await this.db.query(
      `UPDATE users 
       SET name = COALESCE($1, name),
           email = COALESCE($2, email),
           role = COALESCE($3, role)
       WHERE id = $4
       RETURNING *`,
      [userData.name, userData.email, userData.role, id]
    );
    
    return new User(result.rows[0]);
  }
}

// ============ VIEW ============
// views/user/profile.js (template)
const userProfileView = (user) => {
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <title>${user.getDisplayName()} - Profile</title>
        <link rel="stylesheet" href="/css/profile.css">
      </head>
      <body>
        <div class="profile">
          <h1>${user.getDisplayName()}</h1>
          <p class="email">${user.email}</p>
          <p class="role">Role: ${user.role}</p>
          
          <div class="permissions">
            <h3>Permissions:</h3>
            <ul>
              ${user.getPermissions().map(p => `<li>${p}</li>`).join('')}
            </ul>
          </div>
          
          ${user.isAdmin() ? `
            <div class="admin-panel">
              <h3>Admin Controls</h3>
              <button onclick="deleteUser('${user.id}')">Delete User</button>
            </div>
          ` : ''}
        </div>
        
        <script src="/js/profile.js"></script>
      </body>
    </html>
  `;
};

// views/user/list.js
const userListView = (users) => {
  return `
    <!DOCTYPE html>
    <html>
      <head>
        <title>User List</title>
        <link rel="stylesheet" href="/css/users.css">
      </head>
      <body>
        <h1>Users</h1>
        <table class="user-table">
          <thead>
            <tr>
              <th>Name</th>
              <th>Email</th>
              <th>Role</th>
              <th>Actions</th>
            </tr>
          </thead>
          <tbody>
            ${users.map(user => `
              <tr>
                <td>${user.getDisplayName()}</td>
                <td>${user.email}</td>
                <td><span class="role-badge role-${user.role}">${user.role}</span></td>
                <td>
                  <a href="/users/${user.id}">View</a>
                  ${user.isAdmin() ? '' : '<button onclick="editUser(\'' + user.id + '\')">Edit</button>'}
                </td>
              </tr>
            `).join('')}
          </tbody>
        </table>
      </body>
    </html>
  `;
};

// JSON view for API responses
const userJSONView = (user) => {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
    permissions: user.getPermissions(),
    createdAt: user.createdAt
  };
};

// ============ CONTROLLER ============
// controllers/userController.js
class UserController {
  constructor(userRepository) {
    this.userRepository = userRepository;
  }
  
  // GET /users/:id
  async show(req, res) {
    try {
      const user = await this.userRepository.findById(req.params.id);
      
      if (!user) {
        return res.status(404).render('error', { 
          message: 'User not found' 
        });
      }
      
      // Check content type for response format
      if (req.accepts('html')) {
        res.send(userProfileView(user));
      } else {
        res.json(userJSONView(user));
      }
    } catch (error) {
      res.status(500).render('error', { 
        message: 'Internal server error' 
      });
    }
  }
  
  // GET /users
  async list(req, res) {
    try {
      const users = await this.userRepository.findAll();
      
      if (req.accepts('html')) {
        res.send(userListView(users));
      } else {
        res.json(users.map(u => userJSONView(u)));
      }
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch users' });
    }
  }
  
  // POST /users
  async create(req, res) {
    try {
      // Validate input
      const { error } = this.validateUser(req.body);
      if (error) {
        return res.status(400).json({ 
          error: 'Validation failed', 
          details: error.details 
        });
      }
      
      // Check if user exists
      const existing = await this.userRepository.findByEmail(req.body.email);
      if (existing.length > 0) {
        return res.status(409).json({ 
          error: 'Email already exists' 
        });
      }
      
      // Create user
      const user = await this.userRepository.create(req.body);
      
      res.status(201).json(userJSONView(user));
    } catch (error) {
      res.status(500).json({ error: 'Failed to create user' });
    }
  }
  
  // PUT /users/:id
  async update(req, res) {
    try {
      const user = await this.userRepository.update(req.params.id, req.body);
      
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      
      res.json(userJSONView(user));
    } catch (error) {
      res.status(500).json({ error: 'Failed to update user' });
    }
  }
  
  validateUser(userData) {
    const schema = {
      name: userData.name?.trim(),
      email: userData.email?.toLowerCase(),
      role: userData.role
    };
    
    // Validation logic
    const errors = [];
    if (!schema.email?.includes('@')) {
      errors.push('Invalid email');
    }
    
    return {
      error: errors.length > 0 ? { details: errors } : null,
      value: schema
    };
  }
}

// ============ ROUTES ============
// routes/userRoutes.js
const express = require('express');
const router = express.Router();
const UserController = require('../controllers/userController');
const UserRepository = require('../models/UserRepository');
const db = require('../database');

const userRepository = new UserRepository(db);
const userController = new UserController(userRepository);

router.get('/users', (req, res) => userController.list(req, res));
router.get('/users/:id', (req, res) => userController.show(req, res));
router.post('/users', (req, res) => userController.create(req, res));
router.put('/users/:id', (req, res) => userController.update(req, res));
router.delete('/users/:id', (req, res) => userController.delete(req, res));

module.exports = router;

// app.js - Main application
const express = require('express');
const app = express();
app.use(express.json());
app.use('/api', require('./routes/userRoutes'));
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Account Management

```javascript
// ============ MODEL ============
// models/Account.js
class Account {
  constructor(data) {
    this.id = data.id;
    this.accountNumber = data.accountNumber;
    this.customerId = data.customerId;
    this.type = data.type;
    this.currency = data.currency;
    this.balance = parseFloat(data.balance) || 0;
    this.status = data.status || 'ACTIVE';
    this.createdAt = data.createdAt;
    this.updatedAt = data.updatedAt;
  }
  
  canWithdraw(amount) {
    if (this.status !== 'ACTIVE') return false;
    
    const minimumBalance = this.getMinimumBalance();
    return this.balance - amount >= minimumBalance;
  }
  
  withdraw(amount) {
    if (!this.canWithdraw(amount)) {
      throw new Error('Insufficient funds');
    }
    
    this.balance -= amount;
    this.updatedAt = new Date();
    
    return {
      type: 'WITHDRAWAL',
      amount,
      newBalance: this.balance
    };
  }
  
  deposit(amount) {
    this.balance += amount;
    this.updatedAt = new Date();
    
    return {
      type: 'DEPOSIT',
      amount,
      newBalance: this.balance
    };
  }
  
  getMinimumBalance() {
    const minimums = {
      'CHECKING': 0,
      'SAVINGS': 500,
      'BUSINESS': 1000
    };
    return minimums[this.type] || 0;
  }
  
  toJSON() {
    return {
      id: this.id,
      accountNumber: this.accountNumber,
      type: this.type,
      currency: this.currency,
      balance: this.balance,
      status: this.status,
      availableBalance: this.balance - this.getMinimumBalance()
    };
  }
}

// models/AccountRepository.js
class AccountRepository {
  constructor(db) {
    this.db = db;
  }
  
  async findById(id) {
    const result = await this.db.query(
      'SELECT * FROM accounts WHERE id = $1',
      [id]
    );
    return result.rows[0] ? new Account(result.rows[0]) : null;
  }
  
  async findByCustomer(customerId) {
    const result = await this.db.query(
      'SELECT * FROM accounts WHERE customer_id = $1',
      [customerId]
    );
    return result.rows.map(row => new Account(row));
  }
  
  async updateBalance(id, amount) {
    const result = await this.db.query(
      `UPDATE accounts 
       SET balance = balance + $1, updated_at = NOW() 
       WHERE id = $2 
       RETURNING *`,
      [amount, id]
    );
    return new Account(result.rows[0]);
  }
}

// ============ VIEW ============
// views/account/dashboard.js
const accountDashboardView = (accounts, customer) => {
  const totalBalance = accounts.reduce((sum, acc) => sum + acc.balance, 0);
  
  return `
    <div class="dashboard">
      <h1>Welcome, ${customer.name}</h1>
      <div class="summary">
        <h2>Total Balance: $${totalBalance.toFixed(2)}</h2>
      </div>
      
      <div class="accounts">
        ${accounts.map(acc => `
          <div class="account-card account-${acc.type.toLowerCase()}">
            <h3>${acc.type} Account</h3>
            <p class="account-number">****${acc.accountNumber.slice(-4)}</p>
            <p class="balance">$${acc.balance.toFixed(2)}</p>
            <p class="available">Available: $${acc.availableBalance.toFixed(2)}</p>
            <button onclick="viewTransactions('${acc.id}')">View Transactions</button>
          </div>
        `).join('')}
      </div>
    </div>
  `;
};

// JSON view for API
const accountJSONView = (account) => ({
  id: account.id,
  accountNumber: `****${account.accountNumber.slice(-4)}`,
  type: account.type,
  currency: account.currency,
  balance: account.balance,
  availableBalance: account.availableBalance,
  status: account.status
});

// ============ CONTROLLER ============
// controllers/accountController.js
class AccountController {
  constructor(accountRepo, customerRepo, transactionRepo) {
    this.accountRepo = accountRepo;
    this.customerRepo = customerRepo;
    this.transactionRepo = transactionRepo;
  }
  
  async dashboard(req, res) {
    try {
      const customer = await this.customerRepo.findById(req.user.customerId);
      const accounts = await this.accountRepo.findByCustomer(customer.id);
      
      res.send(accountDashboardView(accounts, customer));
    } catch (error) {
      res.status(500).render('error', { message: 'Failed to load dashboard' });
    }
  }
  
  async getAccount(req, res) {
    try {
      const account = await this.accountRepo.findById(req.params.id);
      
      if (!account) {
        return res.status(404).json({ error: 'Account not found' });
      }
      
      // Verify ownership
      if (account.customerId !== req.user.customerId) {
        return res.status(403).json({ error: 'Unauthorized' });
      }
      
      res.json(accountJSONView(account));
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch account' });
    }
  }
  
  async transfer(req, res) {
    const { fromAccountId, toAccountId, amount } = req.body;
    
    // Start transaction
    const dbTransaction = await this.accountRepo.beginTransaction();
    
    try {
      const fromAccount = await this.accountRepo.findById(fromAccountId);
      const toAccount = await this.accountRepo.findById(toAccountId);
      
      if (!fromAccount || !toAccount) {
        throw new Error('Account not found');
      }
      
      // Withdraw from source
      fromAccount.withdraw(amount);
      await this.accountRepo.updateBalance(fromAccountId, -amount, dbTransaction);
      
      // Deposit to destination
      await this.accountRepo.updateBalance(toAccountId, amount, dbTransaction);
      
      // Record transaction
      await this.transactionRepo.create({
        fromAccountId,
        toAccountId,
        amount,
        type: 'TRANSFER'
      }, dbTransaction);
      
      await dbTransaction.commit();
      
      res.json({ 
        success: true, 
        message: 'Transfer completed',
        newBalance: fromAccount.balance - amount
      });
    } catch (error) {
      await dbTransaction.rollback();
      res.status(400).json({ error: error.message });
    }
  }
}
```

### 2. E-commerce - Product Catalog

```javascript
// ============ MODEL ============
// models/Product.js
class Product {
  constructor(data) {
    this.id = data.id;
    this.sku = data.sku;
    this.name = data.name;
    this.description = data.description;
    this.price = parseFloat(data.price);
    this.cost = parseFloat(data.cost);
    this.category = data.category;
    this.brand = data.brand;
    this.attributes = data.attributes || {};
    this.inStock = data.inStock || 0;
    this.images = data.images || [];
    this.status = data.status || 'ACTIVE';
    this.createdAt = data.createdAt;
    this.updatedAt = data.updatedAt;
  }
  
  applyDiscount(percentage) {
    this.price = this.price * (1 - percentage / 100);
    this.updatedAt = new Date();
  }
  
  getProfitMargin() {
    return ((this.price - this.cost) / this.price) * 100;
  }
  
  isAvailable(quantity = 1) {
    return this.status === 'ACTIVE' && this.inStock >= quantity;
  }
  
  toJSON() {
    return {
      id: this.id,
      name: this.name,
      price: this.price,
      formattedPrice: new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD'
      }).format(this.price),
      inStock: this.inStock > 0,
      stockLevel: this.inStock,
      images: this.images,
      attributes: this.attributes
    };
  }
}

// models/Category.js
class Category {
  constructor(data) {
    this.id = data.id;
    this.name = data.name;
    this.slug = data.slug;
    this.parentId = data.parentId;
    this.description = data.description;
    this.image = data.image;
    this.productCount = data.productCount || 0;
  }
  
  getPath() {
    return `/category/${this.slug}`;
  }
}

// ============ VIEW ============
// views/product/detail.js
const productDetailView = (product, relatedProducts) => {
  return `
    <div class="product-detail">
      <div class="product-gallery">
        ${product.images.map(img => `
          <img src="${img}" alt="${product.name}" class="product-image">
        `).join('')}
      </div>
      
      <div class="product-info">
        <h1>${product.name}</h1>
        <p class="price">${product.formattedPrice}</p>
        
        <div class="availability ${product.inStock ? 'in-stock' : 'out-of-stock'}">
          ${product.inStock ? '✓ In Stock' : '✗ Out of Stock'}
        </div>
        
        <div class="description">
          ${product.description}
        </div>
        
        <button class="add-to-cart" 
                ${!product.inStock ? 'disabled' : ''}
                onclick="addToCart('${product.id}')">
          Add to Cart
        </button>
      </div>
      
      <div class="related-products">
        <h2>Related Products</h2>
        <div class="product-grid">
          ${relatedProducts.map(p => `
            <div class="product-card">
              <img src="${p.images[0]}" alt="${p.name}">
              <h3>${p.name}</h3>
              <p class="price">${p.formattedPrice}</p>
              <button onclick="viewProduct('${p.id}')">View</button>
            </div>
          `).join('')}
        </div>
      </div>
    </div>
  `;
};

// views/product/search.js
const productSearchView = (products, query, pagination) => {
  return `
    <div class="search-results">
      <h1>Search Results for "${query}"</h1>
      <p>${pagination.total} products found</p>
      
      <div class="filters">
        <select onchange="filterByPrice()">
          <option value="">Price</option>
          <option value="0-50">Under $50</option>
          <option value="50-100">$50 - $100</option>
          <option value="100-200">$100 - $200</option>
          <option value="200+">$200+</option>
        </select>
      </div>
      
      <div class="product-grid">
        ${products.map(p => `
          <div class="product-card">
            <img src="${p.images[0]}" alt="${p.name}">
            <h3>${p.name}</h3>
            <p class="price">${p.formattedPrice}</p>
            <button onclick="viewProduct('${p.id}')">View</button>
          </div>
        `).join('')}
      </div>
      
      <div class="pagination">
        ${Array.from({ length: pagination.pages }, (_, i) => i + 1).map(page => `
          <a href="/products/search?q=${query}&page=${page}" 
             class="${page === pagination.current ? 'active' : ''}">
            ${page}
          </a>
        `).join('')}
      </div>
    </div>
  `;
};

// ============ CONTROLLER ============
// controllers/productController.js
class ProductController {
  constructor(productRepo, categoryRepo, reviewService) {
    this.productRepo = productRepo;
    this.categoryRepo = categoryRepo;
    this.reviewService = reviewService;
  }
  
  async show(req, res) {
    try {
      const product = await this.productRepo.findById(req.params.id);
      
      if (!product) {
        return res.status(404).render('error', { 
          message: 'Product not found' 
        });
      }
      
      const related = await this.productRepo.findRelated(
        product.category,
        product.id,
        4
      );
      
      const reviews = await this.reviewService.getForProduct(product.id);
      
      // Enhance product with reviews
      product.rating = reviews.average;
      product.reviewCount = reviews.total;
      
      if (req.accepts('html')) {
        res.send(productDetailView(product, related));
      } else {
        res.json({
          product: product.toJSON(),
          related: related.map(p => p.toJSON()),
          reviews
        });
      }
    } catch (error) {
      res.status(500).json({ error: 'Failed to load product' });
    }
  }
  
  async search(req, res) {
    try {
      const { q, category, minPrice, maxPrice, page = 1 } = req.query;
      
      const results = await this.productRepo.search({
        query: q,
        category,
        minPrice,
        maxPrice,
        page,
        limit: 20
      });
      
      if (req.accepts('html')) {
        res.send(productSearchView(
          results.products,
          q,
          results.pagination
        ));
      } else {
        res.json({
          products: results.products.map(p => p.toJSON()),
          pagination: results.pagination,
          facets: results.facets
        });
      }
    } catch (error) {
      res.status(500).json({ error: 'Search failed' });
    }
  }
  
  async getByCategory(req, res) {
    try {
      const category = await this.categoryRepo.findBySlug(req.params.slug);
      
      if (!category) {
        return res.status(404).json({ error: 'Category not found' });
      }
      
      const products = await this.productRepo.findByCategory(
        category.id,
        req.query
      );
      
      res.json({
        category,
        products: products.map(p => p.toJSON()),
        total: products.length
      });
    } catch (error) {
      res.status(500).json({ error: 'Failed to load category' });
    }
  }
}
```

---

## MVC vs Other Patterns

| Pattern | Model | View | Controller | Use Case |
|---------|-------|------|------------|----------|
| **MVC** | Data & Logic | Presentation | Input handling | Web apps, desktop apps |
| **MVP** | Data & Logic | Passive View | Presenter (logic) | Complex UIs |
| **MVVM** | Data & Logic | Data-bound | ViewModel | Modern frameworks |
| **MVA** | Data & Logic | View | Mediator | Distributed systems |

---

## Best Practices

### 1. Thin Controllers, Fat Models
```javascript
// GOOD: Controller is thin
class UserController {
  async update(req, res) {
    const user = await User.findById(req.params.id);
    user.updateProfile(req.body);
    await user.save();
    res.json(user.toJSON());
  }
}

// Model has logic
class User {
  updateProfile(data) {
    this.validateEmail(data.email);
    this.checkPasswordStrength(data.password);
    this.name = data.name;
    this.email = data.email;
  }
}

// BAD: Controller is fat
class UserController {
  async update(req, res) {
    // Validation in controller
    if (!req.body.email.includes('@')) {
      return res.status(400).json({ error: 'Invalid email' });
    }
    
    // Logic in controller
    const user = await User.findById(req.params.id);
    user.name = req.body.name;
    user.email = req.body.email;
    
    await user.save();
    res.json(user);
  }
}
```

### 2. Single Responsibility
```javascript
// models/User.js - Only user data and logic
class User {
  validate() { /* ... */ }
  getFullName() { /* ... */ }
}

// views/UserView.js - Only presentation
class UserView {
  renderProfile(user) { /* ... */ }
  renderList(users) { /* ... */ }
}

// controllers/UserController.js - Only coordination
class UserController {
  async showProfile(req, res) {
    const user = await User.findById(req.params.id);
    res.send(UserView.renderProfile(user));
  }
}
```

### 3. Dependency Injection
```javascript
class UserController {
  constructor(userRepo, emailService, logger) {
    this.userRepo = userRepo;      // Injected
    this.emailService = emailService;
    this.logger = logger;
  }
  
  async create(req, res) {
    const user = await this.userRepo.create(req.body);
    await this.emailService.sendWelcome(user.email);
    this.logger.info(`User created: ${user.id}`);
    res.json(user);
  }
}
```

### 4. Request/Response Objects
```javascript
// DTOs for request/response
class CreateUserRequest {
  constructor(body) {
    this.email = body.email;
    this.password = body.password;
    this.name = body.name;
  }
  
  validate() {
    const errors = [];
    if (!this.email?.includes('@')) errors.push('Invalid email');
    if (!this.password || this.password.length < 8) {
      errors.push('Password too short');
    }
    return errors;
  }
}

class UserResponse {
  constructor(user) {
    this.id = user.id;
    this.email = user.email;
    this.name = user.name;
    this.createdAt = user.createdAt;
  }
}

// Controller uses DTOs
class UserController {
  async create(req, res) {
    const request = new CreateUserRequest(req.body);
    
    const errors = request.validate();
    if (errors.length > 0) {
      return res.status(400).json({ errors });
    }
    
    const user = await this.userRepo.create(request);
    res.json(new UserResponse(user));
  }
}
```

### 5. Error Handling
```javascript
class BaseController {
  handleError(error, res) {
    if (error.name === 'ValidationError') {
      return res.status(400).json({ 
        error: 'Validation failed',
        details: error.details 
      });
    }
    
    if (error.name === 'NotFoundError') {
      return res.status(404).json({ 
        error: 'Resource not found' 
      });
    }
    
    if (error.name === 'UnauthorizedError') {
      return res.status(401).json({ 
        error: 'Unauthorized' 
      });
    }
    
    // Log unexpected errors
    console.error('Unexpected error:', error);
    res.status(500).json({ 
      error: 'Internal server error' 
    });
  }
}

class UserController extends BaseController {
  async get(req, res) {
    try {
      const user = await this.userRepo.findById(req.params.id);
      if (!user) {
        throw new NotFoundError('User not found');
      }
      res.json(user);
    } catch (error) {
      this.handleError(error, res);
    }
  }
}
```

---

## Project Structure

```
src/
├── models/               # Data and business logic
│   ├── User.js
│   ├── Product.js
│   └── Order.js
│
├── views/                # Presentation templates
│   ├── user/
│   │   ├── profile.js
│   │   └── list.js
│   └── product/
│       ├── detail.js
│       └── search.js
│
├── controllers/          # Request handlers
│   ├── userController.js
│   ├── productController.js
│   └── orderController.js
│
├── services/             # Business services
│   ├── emailService.js
│   ├── paymentService.js
│   └── authService.js
│
├── repositories/         # Data access
│   ├── userRepository.js
│   └── productRepository.js
│
├── middleware/           # Express middleware
│   ├── auth.js
│   └── validation.js
│
├── routes/               # Route definitions
│   ├── userRoutes.js
│   └── productRoutes.js
│
└── app.js                # Main application
```

---

## When to Use MVC

### ✅ **USE WHEN:**
- Building web applications
- Need clear separation of concerns
- Multiple presentation formats (HTML, JSON)
- Team familiar with pattern
- Traditional server-rendered apps

### ❌ **DON'T USE WHEN:**
- Simple scripts or utilities
- Real-time applications (WebSockets)
- Event-driven architectures
- Microservices (use different patterns per service)

---

## Benefits vs Tradeoffs

| Benefit | Tradeoff |
|---------|----------|
| ✅ Clear separation | ❌ More files/classes |
| ✅ Reusable components | ❌ Steeper learning curve |
| ✅ Testable in isolation | ❌ Can be overkill for simple apps |
| ✅ Parallel development | ❌ Navigation between files |
| ✅ Multiple views per model | ❌ Requires discipline |

---

## Summary

**MVC Core Principles:**
- **Model**: Data, business logic, validation
- **View**: Presentation, templates, UI
- **Controller**: Input handling, coordination

**Best Practices:**
- ✓ Thin controllers, fat models
- ✓ Single responsibility
- ✓ Dependency injection
- ✓ Request/response objects
- ✓ Centralized error handling

**MVC is the foundation of most web frameworks—master it before moving to more complex patterns.**