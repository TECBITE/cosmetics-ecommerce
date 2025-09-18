# BACKEND FILES PART 4

## backend/app.js

```js
require('./utils/loadSecrets');
require('dotenv').config();

const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const path = require('path');

const db = require('./models');
const { errorHandler } = require('./middleware/errorMiddleware');

// Route imports
const authRoutes = require('./routes/auth');
const productRoutes = require('./routes/product');
const orderRoutes = require('./routes/order');
const paymentRoutes = require('./routes/payment');
const labelRoutes = require('./routes/label');

const app = express();

// Security middleware
app.use(helmet());
app.use(cors());

// Body parsing middleware
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Logging middleware
if (process.env.NODE_ENV === 'development') {
  app.use(morgan('dev'));
}

// Static files
app.use('/labels', express.static(path.join(__dirname, 'public', 'labels')));

// Routes
app.use('/api/auth', authRoutes);
app.use('/api/products', productRoutes);
app.use('/api/orders', orderRoutes);
app.use('/api/payment', paymentRoutes);
app.use('/api/label', labelRoutes);

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ 
    status: 'OK', 
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// Error handling middleware
app.use(errorHandler);

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({ message: 'Route not found' });
});

const PORT = process.env.PORT || 5000;

// Database sync and server start
const startServer = async () => {
  try {
    await db.sequelize.authenticate();
    console.log('Database connected successfully');
    
    if (process.env.NODE_ENV !== 'production') {
      await db.sequelize.sync();
      console.log('Database synchronized');
    }
    
    app.listen(PORT, () => {
      console.log(`Server running in ${process.env.NODE_ENV} mode on port ${PORT}`);
    });
    
  } catch (error) {
    console.error('Unable to connect to database:', error);
    process.exit(1);
  }
};

startServer();

module.exports = app;
```

## backend/Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Create labels directory
RUN mkdir -p public/labels

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:5000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

# Start the application
CMD ["node", "app.js"]
```

## backend/.env.example

```env
# Database Configuration
DB_HOST=localhost
DB_USER=postgres
DB_PASS=password
DB_NAME=cosmetics_ecommerce

# JWT Secret
JWT_SECRET=your_super_secret_jwt_key_here

# Cashfree Payment Gateway
CASHFREE_CLIENT_ID=your_cashfree_client_id
CASHFREE_CLIENT_SECRET=your_cashfree_client_secret

# Application URL
BASE_URL=http://localhost:5000

# Node Environment
NODE_ENV=development

# Server Port
PORT=5000
```

## backend/.gitignore

```
# Dependencies
node_modules/
npm-debug.log*

# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Production build
dist/
build/

# Logs
logs
*.log

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory used by tools like istanbul
coverage/

# nyc test coverage
.nyc_output

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Temporary files
tmp/
temp/

# Authentication files
auth_info.json

# Label files
public/labels/*.zip
public/labels/*.txt
public/labels/*.png
```

---

# WHATSAPP BOT FILES

## whatsapp-bot/package.json

```json
{
  "name": "whatsapp-bot",
  "version": "1.0.0",
  "description": "WhatsApp bot for customer support",
  "main": "whatsappClient.js",
  "scripts": {
    "start": "node whatsappClient.js",
    "dev": "nodemon whatsappClient.js"
  },
  "dependencies": {
    "@adiwajshing/baileys": "^4.4.0",
    "pino": "^8.14.1",
    "@hapi/boom": "^10.0.1",
    "node-fetch": "^2.6.11",
    "dotenv": "^16.0.3"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
```

## whatsapp-bot/whatsappClient.js

```js
require('dotenv').config();
const { default: makeWASocket, useSingleFileAuthState, fetchLatestBaileysVersion, DisconnectReason } = require('@adiwajshing/baileys');
const P = require('pino');
const { Boom } = require('@hapi/boom');
const fetch = require('node-fetch');
const { state, saveState } = useSingleFileAuthState('./auth_info.json');

// Configuration
const BACKEND_URL = process.env.BACKEND_URL || 'http://localhost:5000';

async function startWhatsApp() {
  const { version } = await fetchLatestBaileysVersion();
  
  const sock = makeWASocket({
    version,
    printQRInTerminal: true,
    auth: state,
    logger: P({ level: 'silent' }),
    browser: ['Cosmetics Bot', 'Chrome', '1.0.0']
  });

  sock.ev.on('creds.update', saveState);

  sock.ev.on('connection.update', (update) => {
    const { connection, lastDisconnect, qr } = update;
    
    if (qr) {
      console.log('🔗 Scan this QR code with WhatsApp Business app');
    }
    
    if (connection === 'close') {
      const shouldReconnect = (lastDisconnect?.error as Boom)?.output?.statusCode !== DisconnectReason.loggedOut;
      console.log('⚠️ Connection closed due to:', lastDisconnect?.error, ', reconnecting =', shouldReconnect);
      
      if (shouldReconnect) {
        setTimeout(startWhatsApp, 5000);
      }
    } else if (connection === 'open') {
      console.log('✅ WhatsApp connected successfully');
    }
  });

  // Message handler
  sock.ev.on('messages.upsert', async (m) => {
    const msg = m.messages[0];
    
    // Ignore system messages and messages from self
    if (!msg.message || msg.key.fromMe) return;

    const sender = msg.key.remoteJid;
    let text = '';

    // Extract text from different message types
    if (msg.message.conversation) {
      text = msg.message.conversation.toLowerCase();
    } else if (msg.message.extendedTextMessage?.text) {
      text = msg.message.extendedTextMessage.text.toLowerCase();
    }

    console.log(`📩 Message from ${sender}: ${text}`);

    let response = 'Sorry, I did not understand that. Type "help" for available commands.';

    try {
      // Handle different commands
      if (text.includes('order status') || text.includes('track order')) {
        response = '📦 Please reply with your order ID to check the status.\n\nExample: ORD1695123456789';
        
      } else if (/^ord\d+$/i.test(text)) {
        // Order ID pattern matching
        const orderId = text.toUpperCase();
        response = await getOrderStatus(orderId);
        
      } else if (text === 'help' || text === 'menu') {
        response = `🛍️ *Welcome to Cosmetics Shop!*\n\nAvailable commands:\n📦 *"order status"* - Check your order\n🆔 *Order ID* - Get specific order details\n📞 *"contact"* - Get support info\n🛒 *"products"* - View our products\n❓ *"help"* - Show this menu`;
        
      } else if (text.includes('contact') || text.includes('support')) {
        response = `📞 *Customer Support*\n\n📱 Phone: +91-XXXXXXXXXX\n📧 Email: support@cosmeticsshop.com\n🕒 Hours: 9 AM - 8 PM (Mon-Sat)\n\nWe're here to help! 😊`;
        
      } else if (text.includes('products') || text.includes('catalog')) {
        response = `🛒 *Our Product Categories*\n\n💄 Makeup & Cosmetics\n🧴 Skincare Products\n🌸 Perfumes & Fragrances\n🧼 Body Care\n\nVisit our website: ${process.env.WEBSITE_URL || 'www.cosmeticsshop.com'}\n\nFor specific products, feel free to ask!`;
        
      } else if (text.includes('price') || text.includes('cost')) {
        response = '💰 For current pricing and offers, please visit our website or ask about specific products.\n\nType "products" to see our categories!';
        
      } else if (text.includes('delivery') || text.includes('shipping')) {
        response = '🚚 *Delivery Information*\n\n📍 Free delivery above ₹500\n⏱️ 2-3 business days\n🏃‍♂️ Same day delivery in select cities\n📦 Cash on delivery available\n\nFor order tracking, share your order ID!';
        
      } else {
        // Default response with suggestions
        response = `I didn't understand "${text}". 🤔\n\nTry these commands:\n• "help" - See all options\n• "order status" - Track orders\n• "contact" - Get support\n• Share your order ID for tracking`;
      }

    } catch (error) {
      console.error('❌ Error processing message:', error);
      response = 'Sorry, I encountered an error. Please try again or contact support.';
    }

    // Send response
    try {
      await sock.sendMessage(sender, { text: response });
      console.log(`✅ Response sent to ${sender}`);
    } catch (error) {
      console.error('❌ Error sending message:', error);
    }
  });
}

// Function to get order status from backend API
async function getOrderStatus(orderId) {
  try {
    const response = await fetch(`${BACKEND_URL}/api/orders/status/${orderId}`);
    
    if (response.ok) {
      const data = await response.json();
      return `📋 *Order Status*\n\n🆔 Order ID: ${data.orderId}\n📊 Status: ${data.status}\n💳 Payment: ${data.paymentStatus}\n💰 Amount: ₹${data.totalAmount}\n📅 Date: ${new Date(data.createdAt).toLocaleDateString('en-IN')}\n\nThank you for shopping with us! 🛍️`;
    } else if (response.status === 404) {
      return `❌ Order not found!\n\nPlease check your order ID and try again.\nOrder IDs start with "ORD" followed by numbers.\n\nNeed help? Type "contact"`;
    } else {
      throw new Error('API request failed');
    }
  } catch (error) {
    console.error('❌ Error fetching order status:', error);
    return '⚠️ Unable to fetch order status right now. Please try again later or contact support.\n\nType "contact" for support details.';
  }
}

// Start the WhatsApp client
console.log('🚀 Starting WhatsApp Bot...');
startWhatsApp().catch(error => {
  console.error('❌ Failed to start WhatsApp bot:', error);
  process.exit(1);
});

// Handle process termination
process.on('SIGINT', () => {
  console.log('👋 WhatsApp Bot shutting down...');
  process.exit(0);
});

process.on('SIGTERM', () => {
  console.log('👋 WhatsApp Bot terminated...');
  process.exit(0);
});
```

## whatsapp-bot/Dockerfile

```dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Expose port (if needed for health checks)
EXPOSE 3001

# Health check
HEALTHCHECK --interval=60s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "console.log('WhatsApp bot health check'); process.exit(0)"

# Start the application
CMD ["node", "whatsappClient.js"]
```

## whatsapp-bot/.env.example

```env
# Backend API URL
BACKEND_URL=http://backend:5000

# Website URL (for sharing with customers)
WEBSITE_URL=https://yourdomain.com

# Node Environment
NODE_ENV=production
```