# Layered Architecture (n-tier)

## Definition
**Layered Architecture** is a software design pattern that organizes an application into horizontal layers, each with a specific responsibility. Each layer provides services to the layer above and consumes services from the layer below. This separation of concerns creates modular, maintainable, and testable systems.

Also known as **n-tier architecture**, it's one of the most traditional and widely used architectural patterns.

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │         PRESENTATION LAYER          │
                    │    (UI, Controllers, API Endpoints) │
                    └────────────────┬────────────────────┘
                                    │
                    ┌────────────────▼────────────────────┐
                    │         BUSINESS LAYER              │
                    │    (Services, Use Cases, Logic)     │
                    └────────────────┬────────────────────┘
                                    │
                    ┌────────────────▼────────────────────┐
                    │        PERSISTENCE LAYER            │
                    │    (Repositories, DAOs)             │
                    └────────────────┬────────────────────┘
                                    │
                    ┌────────────────▼────────────────────┐
                    │          DATABASE LAYER             │
                    │    (DB, File System, External)      │
                    └─────────────────────────────────────┘
```

---

## Layer Responsibilities

| Layer | Responsibility | Examples |
|-------|----------------|----------|
| **Presentation** | User interface, input handling | React components, Express routes, Controllers |
| **Business** | Business logic, validation, workflows | Services, Use Cases, Domain models |
| **Persistence** | Data access, mapping | Repositories, DAOs, ORM |
| **Database** | Data storage | PostgreSQL, MongoDB, Redis |

---

## Good vs Bad Example

### Bad Example (Mixing Concerns)
```javascript
// BAD: All logic in one place
app.post('/api/orders', async (req, res) => {
  // 1. Validation mixed with everything
  if (!req.body.userId || !req.body.items) {
    return res.status(400).json({ error: 'Missing fields' });
  }
  
  // 2. Business logic in controller
  const total = req.body.items.reduce((sum, item) => {
    return sum + (item.price * item.quantity);
  }, 0);
  
  if (total > 10000) {
    return res.status(400).json({ error: 'Order too large' });
  }
  
  // 3. Database access in controller
  const user = await db.query('SELECT * FROM users WHERE id = $1', [req.body.userId]);
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  
  // 4. Direct DB insert
  const order = await db.query(
    'INSERT INTO orders (user_id, total, status) VALUES ($1, $2, $3) RETURNING *',
    [req.body.userId, total, 'PENDING']
  );
  
  // 5. External calls mixed in
  await axios.post('https://payment.com/charge', {
    amount: total,
    card: req.body.payment
  });
  
  // 6. Email sending in controller
  await email.send(order.id, user.email);
  
  res.json(order);
});

// Problems:
// - Hard to test (database, external calls)
// - No separation of concerns
// - Code duplication across endpoints
// - Difficult to maintain
// - Business logic scattered
```

### Good Example (Layered Architecture)
```javascript
// ============ PRESENTATION LAYER ============
// controllers/orderController.js
class OrderController {
  constructor(orderService) {
    this.orderService = orderService;
  }
  
  async createOrder(req, res) {
    try {
      // 1. Input validation (presentation concern)
      const { error } = validateOrderSchema(req.body);
      if (error) {
        return res.status(400).json({ 
          error: 'Validation failed', 
          details: error.details 
        });
      }
      
      // 2. Call business layer
      const order = await this.orderService.createOrder(req.body);
      
      // 3. Format response (presentation concern)
      res.status(201).json({
        success: true,
        data: order,
        links: {
          self: `/api/orders/${order.id}`,
          payment: `/api/orders/${order.id}/payment`
        }
      });
    } catch (error) {
      // 4. Error handling (presentation concern)
      this.handleError(error, res);
    }
  }
  
  handleError(error, res) {
    if (error.code === 'INSUFFICIENT_FUNDS') {
      return res.status(400).json({ error: 'Insufficient funds' });
    }
    if (error.code === 'PRODUCT_NOT_FOUND') {
      return res.status(404).json({ error: 'Product not found' });
    }
    
    console.error('Unexpected error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
}

// ============ BUSINESS LAYER ============
// services/orderService.js
class OrderService {
  constructor(
    orderRepository,
    productRepository,
    userRepository,
    paymentService,
    inventoryService
  ) {
    this.orderRepository = orderRepository;
    this.productRepository = productRepository;
    this.userRepository = userRepository;
    this.paymentService = paymentService;
    this.inventoryService = inventoryService;
  }
  
  async createOrder(orderData) {
    // 1. Business validation
    await this.validateOrder(orderData);
    
    // 2. Calculate totals (business logic)
    const items = await this.enrichItemsWithPrices(orderData.items);
    const subtotal = this.calculateSubtotal(items);
    const tax = this.calculateTax(subtotal, orderData.shippingAddress);
    const shipping = await this.calculateShipping(items, orderData.shippingAddress);
    const total = subtotal + tax + shipping;
    
    // 3. Apply business rules
    if (total > 10000) {
      await this.flagForReview(orderData.userId, total);
    }
    
    // 4. Apply discounts
    const discounts = await this.calculateDiscounts(orderData.userId, items);
    const finalTotal = total - discounts;
    
    // 5. Create order entity
    const order = new Order({
      userId: orderData.userId,
      items,
      subtotal,
      tax,
      shipping,
      discounts,
      total: finalTotal,
      status: 'PENDING',
      createdAt: new Date()
    });
    
    // 6. Save to repository
    const savedOrder = await this.orderRepository.save(order);
    
    // 7. Trigger downstream processes
    await this.inventoryService.reserve(savedOrder);
    await this.paymentService.process(savedOrder);
    
    return savedOrder;
  }
  
  async validateOrder(orderData) {
    // Check user exists
    const user = await this.userRepository.findById(orderData.userId);
    if (!user) {
      throw new BusinessError('USER_NOT_FOUND', 'User does not exist');
    }
    
    // Check products exist
    for (const item of orderData.items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) {
        throw new BusinessError('PRODUCT_NOT_FOUND', `Product ${item.productId} not found`);
      }
      
      // Check inventory
      if (product.stock < item.quantity) {
        throw new BusinessError('INSUFFICIENT_STOCK', `Insufficient stock for ${product.name}`);
      }
    }
    
    // Check payment method
    if (!user.paymentMethods.includes(orderData.paymentMethodId)) {
      throw new BusinessError('INVALID_PAYMENT', 'Payment method not valid');
    }
  }
  
  enrichItemsWithPrices(items) {
    return Promise.all(items.map(async item => {
      const product = await this.productRepository.findById(item.productId);
      return {
        ...item,
        price: product.price,
        name: product.name
      };
    }));
  }
  
  calculateSubtotal(items) {
    return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  }
  
  calculateTax(subtotal, address) {
    // Complex tax logic based on location
    const taxRate = this.getTaxRate(address.state, address.zip);
    return subtotal * taxRate;
  }
  
  async calculateShipping(items, address) {
    // Shipping logic based on weight, distance, etc.
    return this.shippingService.calculateRate(items, address);
  }
  
  async calculateDiscounts(userId, items) {
    // Apply user loyalty discounts
    // Apply product promotions
    // Apply seasonal discounts
    return 0;
  }
  
  async flagForReview(userId, amount) {
    // Create compliance record
    await this.complianceService.flagLargeOrder(userId, amount);
  }
}

// ============ DOMAIN MODELS ============
// models/Order.js
class Order {
  constructor(data) {
    this.id = data.id || uuid();
    this.userId = data.userId;
    this.items = data.items;
    this.subtotal = data.subtotal;
    this.tax = data.tax;
    this.shipping = data.shipping;
    this.discounts = data.discounts;
    this.total = data.total;
    this.status = data.status;
    this.createdAt = data.createdAt;
    this.updatedAt = data.updatedAt;
  }
  
  canBeCancelled() {
    return ['PENDING', 'PAYMENT_AUTHORIZED'].includes(this.status);
  }
  
  isShippable() {
    return this.status === 'PAYMENT_CAPTURED' && 
           this.items.some(i => i.requiresShipping);
  }
  
  calculateTotal() {
    return this.subtotal + this.tax + this.shipping - this.discounts;
  }
}

// ============ PERSISTENCE LAYER ============
// repositories/orderRepository.js
class OrderRepository {
  constructor(database) {
    this.db = database;
    this.cache = new RedisCache();
  }
  
  async save(order) {
    const client = await this.db.beginTransaction();
    
    try {
      // Save order
      const result = await client.query(`
        INSERT INTO orders (
          id, user_id, items, subtotal, tax, shipping, 
          discounts, total, status, created_at, updated_at
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
        RETURNING *
      `, [
        order.id,
        order.userId,
        JSON.stringify(order.items),
        order.subtotal,
        order.tax,
        order.shipping,
        order.discounts,
        order.total,
        order.status,
        order.createdAt,
        order.updatedAt || order.createdAt
      ]);
      
      // Update cache
      await this.cache.set(
        `order:${order.id}`,
        JSON.stringify(order),
        'EX',
        3600
      );
      
      await client.commit();
      
      return this.toDomain(result.rows[0]);
    } catch (error) {
      await client.rollback();
      throw new PersistenceError('Failed to save order', error);
    }
  }
  
  async findById(id) {
    // Try cache first
    const cached = await this.cache.get(`order:${id}`);
    if (cached) {
      return this.toDomain(JSON.parse(cached));
    }
    
    // Fallback to database
    const result = await this.db.query(
      'SELECT * FROM orders WHERE id = $1',
      [id]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return this.toDomain(result.rows[0]);
  }
  
  async findByUser(userId, limit = 10, offset = 0) {
    const result = await this.db.query(`
      SELECT * FROM orders 
      WHERE user_id = $1 
      ORDER BY created_at DESC 
      LIMIT $2 OFFSET $3
    `, [userId, limit, offset]);
    
    return result.rows.map(row => this.toDomain(row));
  }
  
  toDomain(row) {
    return new Order({
      id: row.id,
      userId: row.user_id,
      items: JSON.parse(row.items),
      subtotal: parseFloat(row.subtotal),
      tax: parseFloat(row.tax),
      shipping: parseFloat(row.shipping),
      discounts: parseFloat(row.discounts),
      total: parseFloat(row.total),
      status: row.status,
      createdAt: row.created_at,
      updatedAt: row.updated_at
    });
  }
}
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Loan Application System

```javascript
// ============ PRESENTATION LAYER ============
// controllers/loanController.js
class LoanController {
  constructor(loanService) {
    this.loanService = loanService;
  }
  
  async applyForLoan(req, res) {
    // Validate input
    const schema = Joi.object({
      userId: Joi.string().required(),
      amount: Joi.number().min(1000).max(1000000).required(),
      term: Joi.number().min(12).max(360).required(),
      purpose: Joi.string().valid('home', 'car', 'personal', 'business').required(),
      employmentDetails: Joi.object({
        employer: Joi.string().required(),
        income: Joi.number().required(),
        yearsEmployed: Joi.number().required()
      }).required()
    });
    
    const { error } = schema.validate(req.body);
    if (error) {
      return res.status(400).json({
        error: 'VALIDATION_ERROR',
        details: error.details
      });
    }
    
    try {
      const application = await this.loanService.submitApplication(req.body);
      
      res.status(201).json({
        applicationId: application.id,
        status: application.status,
        estimatedDecision: application.estimatedDecision,
        links: {
          self: `/api/loans/${application.id}`,
          documents: `/api/loans/${application.id}/documents`
        }
      });
    } catch (error) {
      this.handleError(error, res);
    }
  }
  
  async getLoanStatus(req, res) {
    const { id } = req.params;
    
    try {
      const status = await this.loanService.getApplicationStatus(id);
      
      res.json({
        applicationId: id,
        ...status,
        _links: {
          self: `/api/loans/${id}`,
          documents: `/api/loans/${id}/documents`
        }
      });
    } catch (error) {
      this.handleError(error, res);
    }
  }
}

// ============ BUSINESS LAYER ============
// services/loanService.js
class LoanService {
  constructor(
    userRepository,
    loanRepository,
    creditScoreService,
    complianceService,
    underwritingService,
    documentService
  ) {
    this.userRepository = userRepository;
    this.loanRepository = loanRepository;
    this.creditScoreService = creditScoreService;
    this.complianceService = complianceService;
    this.underwritingService = underwritingService;
    this.documentService = documentService;
  }
  
  async submitApplication(applicationData) {
    // 1. Check eligibility
    await this.checkEligibility(applicationData);
    
    // 2. Get credit score
    const creditScore = await this.creditScoreService.getScore(
      applicationData.userId
    );
    
    // 3. Check compliance (KYC, AML, etc.)
    await this.complianceService.check(applicationData.userId);
    
    // 4. Create loan application
    const application = new LoanApplication({
      userId: applicationData.userId,
      amount: applicationData.amount,
      term: applicationData.term,
      purpose: applicationData.purpose,
      employmentDetails: applicationData.employmentDetails,
      creditScore,
      status: 'PENDING_REVIEW',
      appliedAt: new Date()
    });
    
    // 5. Calculate risk score
    application.riskScore = this.calculateRiskScore(application);
    
    // 6. Auto-approve if low risk
    if (application.riskScore < 30) {
      application.status = 'AUTO_APPROVED';
      application.decision = 'APPROVED';
      application.approvedAt = new Date();
    } else {
      // Send to underwriting
      application.status = 'UNDERWRITING';
      await this.underwritingService.queueForReview(application);
    }
    
    // 7. Save application
    const saved = await this.loanRepository.save(application);
    
    // 8. Generate document requests
    await this.documentService.generateRequiredDocuments(saved);
    
    return saved;
  }
  
  async getApplicationStatus(applicationId) {
    const application = await this.loanRepository.findById(applicationId);
    
    if (!application) {
      throw new BusinessError('APPLICATION_NOT_FOUND', 'Loan application not found');
    }
    
    return {
      status: application.status,
      decision: application.decision,
      approvedAmount: application.approvedAmount,
      interestRate: application.interestRate,
      decisionDate: application.decisionDate,
      requiredDocuments: application.requiredDocuments,
      nextSteps: this.getNextSteps(application)
    };
  }
  
  calculateRiskScore(application) {
    let score = 0;
    
    // Credit score factor
    if (application.creditScore < 600) score += 40;
    else if (application.creditScore < 700) score += 20;
    else score += 10;
    
    // Income factor
    const monthlyIncome = application.employmentDetails.income / 12;
    const payment = this.calculateMonthlyPayment(
      application.amount,
      application.term
    );
    
    if (payment / monthlyIncome > 0.5) score += 30;
    else if (payment / monthlyIncome > 0.3) score += 15;
    
    // Employment stability
    if (application.employmentDetails.yearsEmployed < 1) score += 20;
    else if (application.employmentDetails.yearsEmployed < 3) score += 10;
    
    return Math.min(score, 100);
  }
  
  calculateMonthlyPayment(amount, termMonths, rate = 0.05) {
    const monthlyRate = rate / 12;
    const payment = amount * monthlyRate * Math.pow(1 + monthlyRate, termMonths) /
                   (Math.pow(1 + monthlyRate, termMonths) - 1);
    return payment;
  }
  
  getNextSteps(application) {
    const steps = {
      PENDING_REVIEW: ['Upload required documents', 'Check email for updates'],
      UNDERWRITING: ['Await underwriting decision', 'Additional info may be requested'],
      APPROVED: ['Review terms', 'Sign documents', 'Schedule closing'],
      REJECTED: ['Review rejection reasons', 'Consider alternative options']
    };
    
    return steps[application.status] || ['Contact support'];
  }
  
  async checkEligibility(applicationData) {
    // Minimum amount
    if (applicationData.amount < 1000) {
      throw new BusinessError('INVALID_AMOUNT', 'Minimum loan amount is $1,000');
    }
    
    // Maximum amount based on purpose
    const maxAmounts = {
      home: 1000000,
      car: 100000,
      personal: 50000,
      business: 500000
    };
    
    if (applicationData.amount > maxAmounts[applicationData.purpose]) {
      throw new BusinessError(
        'EXCEEDS_LIMIT',
        `Maximum for ${applicationData.purpose} loans is $${maxAmounts[applicationData.purpose]}`
      );
    }
  }
}

// ============ DOMAIN MODELS ============
// models/LoanApplication.js
class LoanApplication {
  constructor(data) {
    this.id = data.id || uuid();
    this.userId = data.userId;
    this.amount = data.amount;
    this.term = data.term;
    this.purpose = data.purpose;
    this.employmentDetails = data.employmentDetails;
    this.creditScore = data.creditScore;
    this.riskScore = data.riskScore;
    this.status = data.status || 'DRAFT';
    this.decision = data.decision;
    this.approvedAmount = data.approvedAmount;
    this.interestRate = data.interestRate;
    this.decisionDate = data.decisionDate;
    this.appliedAt = data.appliedAt || new Date();
    this.documents = data.documents || [];
    this.notes = data.notes || [];
  }
  
  addDocument(document) {
    this.documents.push({
      ...document,
      uploadedAt: new Date()
    });
  }
  
  approve(amount, rate, reviewerId) {
    if (this.status !== 'UNDERWRITING') {
      throw new BusinessError('INVALID_STATE', 'Application not in underwriting');
    }
    
    this.status = 'APPROVED';
    this.decision = 'APPROVED';
    this.approvedAmount = amount;
    this.interestRate = rate;
    this.decisionDate = new Date();
    this.notes.push({
      type: 'APPROVAL',
      userId: reviewerId,
      timestamp: new Date(),
      note: `Approved for $${amount} at ${rate}%`
    });
  }
  
  reject(reason, reviewerId) {
    if (this.status !== 'UNDERWRITING') {
      throw new BusinessError('INVALID_STATE', 'Application not in underwriting');
    }
    
    this.status = 'REJECTED';
    this.decision = 'REJECTED';
    this.decisionDate = new Date();
    this.notes.push({
      type: 'REJECTION',
      userId: reviewerId,
      timestamp: new Date(),
      note: `Rejected: ${reason}`
    });
  }
  
  isComplete() {
    return this.documents.length >= this.requiredDocuments.length;
  }
}

// ============ PERSISTENCE LAYER ============
// repositories/loanRepository.js
class LoanRepository {
  constructor(db) {
    this.db = db;
  }
  
  async save(application) {
    const result = await this.db.query(`
      INSERT INTO loan_applications (
        id, user_id, amount, term, purpose, employment_details,
        credit_score, risk_score, status, decision, approved_amount,
        interest_rate, decision_date, applied_at, documents, notes
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16)
      ON CONFLICT (id) DO UPDATE SET
        status = EXCLUDED.status,
        decision = EXCLUDED.decision,
        approved_amount = EXCLUDED.approved_amount,
        interest_rate = EXCLUDED.interest_rate,
        decision_date = EXCLUDED.decision_date,
        documents = EXCLUDED.documents,
        notes = EXCLUDED.notes
      RETURNING *
    `, [
      application.id,
      application.userId,
      application.amount,
      application.term,
      application.purpose,
      JSON.stringify(application.employmentDetails),
      application.creditScore,
      application.riskScore,
      application.status,
      application.decision,
      application.approvedAmount,
      application.interestRate,
      application.decisionDate,
      application.appliedAt,
      JSON.stringify(application.documents),
      JSON.stringify(application.notes)
    ]);
    
    return this.toDomain(result.rows[0]);
  }
  
  async findById(id) {
    const result = await this.db.query(
      'SELECT * FROM loan_applications WHERE id = $1',
      [id]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return this.toDomain(result.rows[0]);
  }
  
  async findByUser(userId, status = null) {
    let query = 'SELECT * FROM loan_applications WHERE user_id = $1';
    const params = [userId];
    
    if (status) {
      query += ' AND status = $2';
      params.push(status);
    }
    
    query += ' ORDER BY applied_at DESC';
    
    const result = await this.db.query(query, params);
    return result.rows.map(row => this.toDomain(row));
  }
  
  toDomain(row) {
    return new LoanApplication({
      id: row.id,
      userId: row.user_id,
      amount: parseFloat(row.amount),
      term: row.term,
      purpose: row.purpose,
      employmentDetails: JSON.parse(row.employment_details),
      creditScore: row.credit_score,
      riskScore: row.risk_score,
      status: row.status,
      decision: row.decision,
      approvedAmount: row.approved_amount ? parseFloat(row.approved_amount) : null,
      interestRate: row.interest_rate ? parseFloat(row.interest_rate) : null,
      decisionDate: row.decision_date,
      appliedAt: row.applied_at,
      documents: JSON.parse(row.documents || '[]'),
      notes: JSON.parse(row.notes || '[]')
    });
  }
}
```

### 2. E-commerce - Order Processing System

```javascript
// ============ PRESENTATION LAYER ============
// controllers/orderController.js
class OrderController {
  constructor(orderService) {
    this.orderService = orderService;
  }
  
  async createOrder(req, res) {
    // Validate request
    const { error } = validateOrderSchema(req.body);
    if (error) {
      return res.status(400).json({
        error: 'VALIDATION_ERROR',
        details: error.details
      });
    }
    
    try {
      const order = await this.orderService.createOrder(req.body);
      
      res.status(201).json({
        orderId: order.id,
        status: order.status,
        total: order.total,
        estimatedDelivery: order.estimatedDelivery,
        _links: {
          self: `/api/orders/${order.id}`,
          payment: `/api/orders/${order.id}/payment`,
          tracking: order.trackingId ? `/api/tracking/${order.trackingId}` : null
        }
      });
    } catch (error) {
      this.handleError(error, res);
    }
  }
  
  async getOrder(req, res) {
    const { id } = req.params;
    
    try {
      const order = await this.orderService.getOrder(id);
      
      // Format for presentation
      res.json({
        ...order,
        _links: {
          self: `/api/orders/${id}`,
          cancel: `/api/orders/${id}/cancel`,
          return: `/api/orders/${id}/return`
        }
      });
    } catch (error) {
      this.handleError(error, res);
    }
  }
  
  handleError(error, res) {
    const errorMap = {
      'ORDER_NOT_FOUND': 404,
      'INVALID_PAYMENT': 400,
      'INSUFFICIENT_STOCK': 409,
      'UNAUTHORIZED': 401
    };
    
    const status = errorMap[error.code] || 500;
    const message = error.message || 'Internal server error';
    
    res.status(status).json({
      error: error.code || 'INTERNAL_ERROR',
      message,
      timestamp: new Date().toISOString(),
      requestId: req.id
    });
  }
}

// ============ BUSINESS LAYER ============
// services/orderService.js
class OrderService {
  constructor(
    orderRepository,
    productRepository,
    userRepository,
    inventoryService,
    paymentService,
    shippingService,
    emailService,
    eventBus
  ) {
    this.orderRepository = orderRepository;
    this.productRepository = productRepository;
    this.userRepository = userRepository;
    this.inventoryService = inventoryService;
    this.paymentService = paymentService;
    this.shippingService = shippingService;
    this.emailService = emailService;
    this.eventBus = eventBus;
  }
  
  async createOrder(orderData) {
    // Start transaction
    const transaction = await this.orderRepository.beginTransaction();
    
    try {
      // 1. Validate user
      const user = await this.userRepository.findById(orderData.userId);
      if (!user) {
        throw new BusinessError('USER_NOT_FOUND', 'User not found');
      }
      
      // 2. Validate and enrich items
      const items = await this.validateAndEnrichItems(orderData.items);
      
      // 3. Check inventory
      await this.inventoryService.checkAvailability(items);
      
      // 4. Calculate pricing
      const pricing = await this.calculatePricing(items, user);
      
      // 5. Calculate shipping
      const shipping = await this.shippingService.calculateRates(
        items,
        orderData.shippingAddress
      );
      
      // 6. Apply promotions
      const promotions = await this.applyPromotions(user, items, pricing);
      
      // 7. Create order entity
      const order = new Order({
        userId: user.id,
        items,
        subtotal: pricing.subtotal,
        tax: pricing.tax,
        shipping: shipping.cost,
        discounts: promotions.totalDiscount,
        total: pricing.subtotal + pricing.tax + shipping.cost - promotions.totalDiscount,
        shippingAddress: orderData.shippingAddress,
        billingAddress: orderData.billingAddress,
        paymentMethod: orderData.paymentMethod,
        status: 'PENDING_PAYMENT',
        createdAt: new Date()
      });
      
      // 8. Save order
      const savedOrder = await this.orderRepository.save(order, { transaction });
      
      // 9. Reserve inventory
      await this.inventoryService.reserve(savedOrder.id, items);
      
      // 10. Process payment
      const payment = await this.paymentService.authorize({
        orderId: savedOrder.id,
        amount: savedOrder.total,
        method: orderData.paymentMethod
      });
      
      // 11. Update order with payment info
      savedOrder.paymentId = payment.id;
      savedOrder.status = 'PAYMENT_AUTHORIZED';
      await this.orderRepository.update(savedOrder, { transaction });
      
      // 12. Commit transaction
      await transaction.commit();
      
      // 13. Send events (after commit)
      await this.eventBus.publish('order.created', {
        orderId: savedOrder.id,
        userId: user.id,
        amount: savedOrder.total,
        items: items.length
      });
      
      // 14. Send confirmation email (async)
      this.emailService.sendOrderConfirmation(savedOrder).catch(e => {
        console.error('Failed to send email:', e);
      });
      
      return savedOrder;
      
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
  
  async validateAndEnrichItems(items) {
    const enriched = [];
    
    for (const item of items) {
      const product = await this.productRepository.findById(item.productId);
      
      if (!product) {
        throw new BusinessError('PRODUCT_NOT_FOUND', `Product ${item.productId} not found`);
      }
      
      if (!product.isActive) {
        throw new BusinessError('PRODUCT_INACTIVE', `Product ${product.name} is not available`);
      }
      
      enriched.push({
        productId: product.id,
        sku: product.sku,
        name: product.name,
        quantity: item.quantity,
        unitPrice: product.price,
        total: product.price * item.quantity,
        requiresShipping: product.requiresShipping,
        weight: product.weight,
        category: product.category
      });
    }
    
    return enriched;
  }
  
  async calculatePricing(items, user) {
    const subtotal = items.reduce((sum, item) => sum + item.total, 0);
    
    // Calculate tax based on user location
    const taxRate = await this.getTaxRate(user.address.state);
    const tax = subtotal * taxRate;
    
    return { subtotal, tax };
  }
  
  async applyPromotions(user, items, pricing) {
    // Check for user loyalty discounts
    const loyaltyDiscount = await this.getLoyaltyDiscount(user);
    
    // Check for product promotions
    const productPromotions = await this.getProductPromotions(items);
    
    // Check for cart-level promotions
    const cartPromotion = await this.getCartPromotion(pricing.subtotal);
    
    const totalDiscount = loyaltyDiscount + productPromotions + cartPromotion;
    
    return {
      loyalty: loyaltyDiscount,
      products: productPromotions,
      cart: cartPromotion,
      totalDiscount
    };
  }
}

// ============ DOMAIN MODELS ============
// models/Order.js
class Order {
  constructor(data) {
    this.id = data.id || uuid();
    this.userId = data.userId;
    this.items = data.items;
    this.subtotal = data.subtotal;
    this.tax = data.tax;
    this.shipping = data.shipping;
    this.discounts = data.discounts;
    this.total = data.total;
    this.shippingAddress = data.shippingAddress;
    this.billingAddress = data.billingAddress;
    this.paymentMethod = data.paymentMethod;
    this.paymentId = data.paymentId;
    this.status = data.status;
    this.trackingId = data.trackingId;
    this.createdAt = data.createdAt;
    this.updatedAt = data.updatedAt;
    this.cancelledAt = data.cancelledAt;
  }
  
  canCancel() {
    const cancellableStatuses = ['PENDING_PAYMENT', 'PAYMENT_AUTHORIZED', 'PAYMENT_FAILED'];
    return cancellableStatuses.includes(this.status);
  }
  
  canReturn() {
    if (this.status !== 'DELIVERED') return false;
    
    const deliveryDate = new Date(this.updatedAt);
    const now = new Date();
    const daysSinceDelivery = (now - deliveryDate) / (1000 * 60 * 60 * 24);
    
    return daysSinceDelivery <= 30; // 30-day return policy
  }
  
  cancel() {
    if (!this.canCancel()) {
      throw new BusinessError('CANNOT_CANCEL', 'Order cannot be cancelled at this stage');
    }
    
    this.status = 'CANCELLED';
    this.cancelledAt = new Date();
  }
  
  addTracking(trackingId, carrier) {
    this.trackingId = trackingId;
    this.carrier = carrier;
    this.status = 'SHIPPED';
    this.updatedAt = new Date();
  }
  
  markDelivered() {
    this.status = 'DELIVERED';
    this.deliveredAt = new Date();
  }
}

// ============ PERSISTENCE LAYER ============
// repositories/orderRepository.js
class OrderRepository {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }
  
  async beginTransaction() {
    const client = await this.db.connect();
    await client.query('BEGIN');
    return client;
  }
  
  async save(order, options = {}) {
    const client = options.transaction || this.db;
    
    const result = await client.query(`
      INSERT INTO orders (
        id, user_id, items, subtotal, tax, shipping,
        discounts, total, shipping_address, billing_address,
        payment_method, payment_id, status, tracking_id,
        created_at, updated_at
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16)
      RETURNING *
    `, [
      order.id,
      order.userId,
      JSON.stringify(order.items),
      order.subtotal,
      order.tax,
      order.shipping,
      order.discounts,
      order.total,
      JSON.stringify(order.shippingAddress),
      JSON.stringify(order.billingAddress),
      order.paymentMethod,
      order.paymentId,
      order.status,
      order.trackingId,
      order.createdAt,
      order.updatedAt || order.createdAt
    ]);
    
    // Invalidate cache
    await this.cache.del(`order:${order.id}`);
    
    return this.toDomain(result.rows[0]);
  }
  
  async findById(id) {
    // Try cache first
    const cached = await this.cache.get(`order:${id}`);
    if (cached) {
      return this.toDomain(JSON.parse(cached));
    }
    
    const result = await this.db.query(
      'SELECT * FROM orders WHERE id = $1',
      [id]
    );
    
    if (result.rows.length === 0) {
      return null;
    }
    
    const order = this.toDomain(result.rows[0]);
    
    // Cache for future requests
    await this.cache.set(
      `order:${id}`,
      JSON.stringify(order),
      'EX',
      300 // 5 minutes
    );
    
    return order;