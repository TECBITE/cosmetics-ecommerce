# BACKEND FILES PART 3

## backend/controllers/orderController.js

```js
const asyncHandler = require('express-async-handler');
const db = require('../models');
const Order = db.Order;

// @desc    Create new order
// @route   POST /api/orders
// @access  Private
const createOrder = asyncHandler(async (req, res) => {
  const {
    orderItems,
    shippingAddress,
    paymentMethod,
    totalPrice,
    customerName,
    customerEmail,
    customerPhone
  } = req.body;

  if (orderItems && orderItems.length === 0) {
    res.status(400);
    throw new Error('No order items');
  }

  const orderId = `ORD${Date.now()}`;

  const order = await Order.create({
    orderId,
    userId: req.user ? req.user.id : null,
    items: orderItems,
    shippingAddress,
    totalAmount: totalPrice,
    customerName: customerName || req.user?.name,
    customerEmail: customerEmail || req.user?.email,
    customerPhone: customerPhone || req.user?.phone
  });

  res.status(201).json(order);
});

// @desc    Get order by ID
// @route   GET /api/orders/:id
// @access  Private
const getOrderById = asyncHandler(async (req, res) => {
  const order = await Order.findOne({
    where: { orderId: req.params.id },
    include: [{ model: db.User, attributes: ['name', 'email'] }]
  });

  if (order) {
    res.json(order);
  } else {
    res.status(404);
    throw new Error('Order not found');
  }
});

// @desc    Get order status by ID
// @route   GET /api/orders/status/:orderId
// @access  Public
const getOrderStatus = asyncHandler(async (req, res) => {
  const order = await Order.findOne({ 
    where: { orderId: req.params.orderId },
    attributes: ['orderId', 'status', 'paymentStatus', 'totalAmount', 'createdAt']
  });

  if (order) {
    res.json({
      orderId: order.orderId,
      status: order.status,
      paymentStatus: order.paymentStatus,
      totalAmount: order.totalAmount,
      createdAt: order.createdAt
    });
  } else {
    res.status(404).json({ message: 'Order not found' });
  }
});

// @desc    Update order status
// @route   PUT /api/orders/:id/status
// @access  Private/Admin
const updateOrderStatus = asyncHandler(async (req, res) => {
  const { status } = req.body;
  
  const order = await Order.findOne({ where: { orderId: req.params.id } });

  if (order) {
    order.status = status;
    const updatedOrder = await order.save();
    res.json(updatedOrder);
  } else {
    res.status(404);
    throw new Error('Order not found');
  }
});

// @desc    Get logged in user orders
// @route   GET /api/orders/myorders
// @access  Private
const getMyOrders = asyncHandler(async (req, res) => {
  const orders = await Order.findAll({
    where: { userId: req.user.id },
    order: [['createdAt', 'DESC']]
  });
  
  res.json(orders);
});

// @desc    Get all orders
// @route   GET /api/orders
// @access  Private/Admin
const getOrders = asyncHandler(async (req, res) => {
  const orders = await Order.findAll({
    include: [{ model: db.User, attributes: ['name', 'email'] }],
    order: [['createdAt', 'DESC']]
  });
  
  res.json(orders);
});

module.exports = {
  createOrder,
  getOrderById,
  getOrderStatus,
  updateOrderStatus,
  getMyOrders,
  getOrders
};
```

## backend/controllers/paymentController.js

```js
const asyncHandler = require('express-async-handler');
const fetch = require('node-fetch');
const db = require('../models');
const Order = db.Order;

// @desc    Create payment order
// @route   POST /api/payment/create-order
// @access  Private
const createPaymentOrder = asyncHandler(async (req, res) => {
  const { amount, orderId, customerName, customerEmail, customerPhone } = req.body;

  // Cashfree configuration
  const cashfreeApiUrl = 'https://sandbox.cashfree.com/pg/orders';
  const clientId = process.env.CASHFREE_CLIENT_ID;
  const clientSecret = process.env.CASHFREE_CLIENT_SECRET;

  const data = {
    order_id: orderId,
    order_amount: amount,
    order_currency: 'INR',
    customer_details: {
      customer_name: customerName,
      customer_email: customerEmail,
      customer_phone: customerPhone,
    },
    order_meta: {
      return_url: `${process.env.BASE_URL}/payment/success?order_id=${orderId}`,
      notify_url: `${process.env.BASE_URL}/api/payment/callback`,
    }
  };

  try {
    const response = await fetch(cashfreeApiUrl, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-client-id': clientId,
        'x-client-secret': clientSecret,
        'x-api-version': '2022-09-01'
      },
      body: JSON.stringify(data)
    });

    const result = await response.json();

    if (response.ok) {
      res.json({
        paymentSessionId: result.payment_session_id,
        orderId: result.order_id,
        status: result.order_status
      });
    } else {
      res.status(400).json({ message: 'Payment order creation failed', error: result });
    }
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

// @desc    Payment callback webhook
// @route   POST /api/payment/callback
// @access  Public
const handlePaymentCallback = asyncHandler(async (req, res) => {
  const paymentData = req.body;
  
  // TODO: Verify webhook signature for security
  
  const { order_id, order_status, payment_status, transaction_id } = paymentData;

  try {
    const order = await Order.findOne({ where: { orderId: order_id } });
    
    if (order) {
      order.paymentStatus = payment_status;
      order.paymentReference = transaction_id;
      
      if (payment_status === 'SUCCESS') {
        order.status = 'Confirmed';
      } else if (payment_status === 'FAILED') {
        order.status = 'Payment Failed';
      }
      
      await order.save();
      
      console.log(`Payment update for order ${order_id}: ${payment_status}`);
    }
    
    res.status(200).send('OK');
  } catch (error) {
    console.error('Payment callback error:', error);
    res.status(500).send('Error processing callback');
  }
});

// @desc    Verify payment
// @route   GET /api/payment/verify/:orderId
// @access  Private
const verifyPayment = asyncHandler(async (req, res) => {
  const { orderId } = req.params;
  
  const clientId = process.env.CASHFREE_CLIENT_ID;
  const clientSecret = process.env.CASHFREE_CLIENT_SECRET;
  
  try {
    const response = await fetch(`https://sandbox.cashfree.com/pg/orders/${orderId}`, {
      method: 'GET',
      headers: {
        'x-client-id': clientId,
        'x-client-secret': clientSecret,
        'x-api-version': '2022-09-01'
      }
    });
    
    const result = await response.json();
    
    if (response.ok) {
      res.json(result);
    } else {
      res.status(400).json({ message: 'Payment verification failed', error: result });
    }
  } catch (error) {
    res.status(500).json({ message: 'Server error', error: error.message });
  }
});

module.exports = {
  createPaymentOrder,
  handlePaymentCallback,
  verifyPayment
};
```

## backend/controllers/labelController.js

```js
const asyncHandler = require('express-async-handler');
const JSZip = require('jszip');
const fs = require('fs');
const path = require('path');
const db = require('../models');
const Product = db.Product;

// Simple barcode generator (you can use a proper library like jsbarcode)
const generateBarcodeLabel = async (product) => {
  // This is a simplified version - in production, use a proper barcode library
  const labelContent = `
    Product: ${product.name}
    Price: ₹${product.price}
    SKU: ${product.sku || product.id}
    Stock: ${product.stock}
  `;
  
  // In a real implementation, generate actual barcode image
  return Buffer.from(labelContent, 'utf8');
};

// @desc    Generate batch labels
// @route   POST /api/label/print-batch
// @access  Private/Admin
const generateBatchLabels = asyncHandler(async (req, res) => {
  const { productIds } = req.body;
  
  if (!productIds || productIds.length === 0) {
    res.status(400);
    throw new Error('No products selected');
  }

  const products = await Product.findAll({
    where: { id: productIds, isActive: true }
  });

  if (products.length === 0) {
    res.status(404);
    throw new Error('No valid products found');
  }

  const zip = new JSZip();

  for (const product of products) {
    const labelBuffer = await generateBarcodeLabel(product);
    zip.file(`${product.name.replace(/[^a-z0-9]/gi, '_')}_${product.id}.txt`, labelBuffer);
  }

  const zipBuffer = await zip.generateAsync({ type: 'nodebuffer' });

  // Ensure labels directory exists
  const labelsDir = path.join(__dirname, '..', 'public', 'labels');
  if (!fs.existsSync(labelsDir)) {
    fs.mkdirSync(labelsDir, { recursive: true });
  }

  const zipFileName = `labels_${Date.now()}.zip`;
  const zipPath = path.join(labelsDir, zipFileName);
  
  fs.writeFileSync(zipPath, zipBuffer);

  const downloadUrl = `/labels/${zipFileName}`;
  
  res.json({ 
    labelZipUrl: downloadUrl,
    message: `Generated labels for ${products.length} products`
  });
});

// @desc    Generate single product label
// @route   POST /api/label/print-single/:id
// @access  Private/Admin
const generateSingleLabel = asyncHandler(async (req, res) => {
  const product = await Product.findByPk(req.params.id);

  if (!product || !product.isActive) {
    res.status(404);
    throw new Error('Product not found');
  }

  const labelBuffer = await generateBarcodeLabel(product);
  
  // Ensure labels directory exists
  const labelsDir = path.join(__dirname, '..', 'public', 'labels');
  if (!fs.existsSync(labelsDir)) {
    fs.mkdirSync(labelsDir, { recursive: true });
  }

  const fileName = `${product.name.replace(/[^a-z0-9]/gi, '_')}_${product.id}_${Date.now()}.txt`;
  const filePath = path.join(labelsDir, fileName);
  
  fs.writeFileSync(filePath, labelBuffer);

  const downloadUrl = `/labels/${fileName}`;
  
  res.json({ 
    labelUrl: downloadUrl,
    message: `Generated label for ${product.name}`
  });
});

module.exports = {
  generateBatchLabels,
  generateSingleLabel
};
```

## backend/routes/auth.js

```js
const express = require('express');
const { registerUser, loginUser, getUserProfile } = require('../controllers/authController');
const { protect } = require('../middleware/authMiddleware');

const router = express.Router();

router.post('/register', registerUser);
router.post('/login', loginUser);
router.get('/profile', protect, getUserProfile);

module.exports = router;
```

## backend/routes/product.js

```js
const express = require('express');
const {
  getProducts,
  getProductById,
  createProduct,
  updateProduct,
  deleteProduct
} = require('../controllers/productController');
const { protect, admin } = require('../middleware/authMiddleware');

const router = express.Router();

router.route('/').get(getProducts).post(protect, admin, createProduct);
router.route('/:id')
  .get(getProductById)
  .put(protect, admin, updateProduct)
  .delete(protect, admin, deleteProduct);

module.exports = router;
```

## backend/routes/order.js

```js
const express = require('express');
const {
  createOrder,
  getOrderById,
  getOrderStatus,
  updateOrderStatus,
  getMyOrders,
  getOrders
} = require('../controllers/orderController');
const { protect, admin } = require('../middleware/authMiddleware');

const router = express.Router();

router.route('/').post(createOrder).get(protect, admin, getOrders);
router.get('/myorders', protect, getMyOrders);
router.get('/status/:orderId', getOrderStatus);
router.get('/:id', getOrderById);
router.put('/:id/status', protect, admin, updateOrderStatus);

module.exports = router;
```

## backend/routes/payment.js

```js
const express = require('express');
const {
  createPaymentOrder,
  handlePaymentCallback,
  verifyPayment
} = require('../controllers/paymentController');

const router = express.Router();

router.post('/create-order', createPaymentOrder);
router.post('/callback', handlePaymentCallback);
router.get('/verify/:orderId', verifyPayment);

module.exports = router;
```

## backend/routes/label.js

```js
const express = require('express');
const {
  generateBatchLabels,
  generateSingleLabel
} = require('../controllers/labelController');
const { protect, admin } = require('../middleware/authMiddleware');

const router = express.Router();

router.post('/print-batch', protect, admin, generateBatchLabels);
router.post('/print-single/:id', protect, admin, generateSingleLabel);

module.exports = router;
```