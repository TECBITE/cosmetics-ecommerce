# CI/CD AND SCRIPTS

## .github/workflows/deploy.yml

```yaml
name: Deploy Cosmetics E-commerce Platform

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: '**/package-lock.json'
    
    - name: Install Backend Dependencies
      working-directory: ./backend
      run: npm ci
    
    - name: Run Backend Tests
      working-directory: ./backend
      env:
        NODE_ENV: test
        DB_HOST: localhost
        DB_USER: postgres
        DB_PASS: postgres
        DB_NAME: test_db
        JWT_SECRET: test_jwt_secret
      run: |
        npm run migrate
        # Add npm test when you have tests
    
    - name: Install Frontend Dependencies
      working-directory: ./frontend
      run: npm ci
    
    - name: Build Frontend
      working-directory: ./frontend
      run: npm run build

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    strategy:
      matrix:
        component: [backend, frontend, whatsapp-bot]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-${{ matrix.component }}
        tags: |
          type=ref,event=branch
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: ./${{ matrix.component }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Production Server
      uses: appleboy/ssh-action@v0.1.7
      with:
        host: ${{ secrets.PRODUCTION_HOST }}
        username: ${{ secrets.PRODUCTION_USER }}
        key: ${{ secrets.PRODUCTION_SSH_KEY }}
        script: |
          cd /opt/cosmetics-ecommerce
          
          # Update images in docker-compose.yml to use new tags
          sed -i "s|image: .*backend.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:main|g" deployment/docker-compose.yml
          sed -i "s|image: .*frontend.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-frontend:main|g" deployment/docker-compose.yml
          sed -i "s|image: .*whatsapp-bot.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-whatsapp-bot:main|g" deployment/docker-compose.yml
          
          # Pull latest images and restart services
          docker-compose -f deployment/docker-compose.yml pull
          docker-compose -f deployment/docker-compose.yml up -d
          
          # Run database migrations
          docker-compose -f deployment/docker-compose.yml exec -T backend npm run migrate
          
          # Clean up old images
          docker image prune -f
          
          echo "✅ Deployment completed successfully"
```

## scripts/setup-nginx-ssl.sh

```bash
#!/bin/bash

# Configuration - Update these variables
DOMAIN="yourdomain.com"
EMAIL="your-email@domain.com"

echo "🚀 Setting up Nginx with SSL for $DOMAIN"

# Update system
sudo apt update && sudo apt upgrade -y

# Install Nginx
sudo apt install -y nginx

# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Create Nginx configuration
sudo tee /etc/nginx/sites-available/$DOMAIN > /dev/null <<EOF
server {
    listen 80;
    server_name $DOMAIN www.$DOMAIN;

    # Backend API proxy
    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_cache_bypass \$http_upgrade;
    }

    # Static files (labels)
    location /labels {
        proxy_pass http://localhost:5000;
    }

    # Frontend
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_cache_bypass \$http_upgrade;
    }
}
EOF

# Enable site
sudo ln -sf /etc/nginx/sites-available/$DOMAIN /etc/nginx/sites-enabled/

# Remove default site
sudo rm -f /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

if [ $? -eq 0 ]; then
    echo "✅ Nginx configuration is valid"
    sudo systemctl reload nginx
else
    echo "❌ Nginx configuration error"
    exit 1
fi

# Obtain SSL certificate
echo "📜 Obtaining SSL certificate for $DOMAIN..."
sudo certbot --nginx -d $DOMAIN -d www.$DOMAIN --non-interactive --agree-tos -m $EMAIL

if [ $? -eq 0 ]; then
    echo "✅ SSL certificate obtained successfully"
else
    echo "❌ Failed to obtain SSL certificate"
    exit 1
fi

# Setup auto-renewal
echo "⚙️ Setting up SSL certificate auto-renewal..."
(crontab -l 2>/dev/null; echo "0 12 * * * /usr/bin/certbot renew --quiet") | crontab -

# Configure firewall
echo "🔥 Configuring UFW firewall..."
sudo ufw allow 'Nginx Full'
sudo ufw allow ssh
sudo ufw --force enable

echo ""
echo "🎉 Setup completed successfully!"
echo "📋 Summary:"
echo "   Domain: https://$DOMAIN"
echo "   SSL: Enabled with auto-renewal"
echo "   Firewall: UFW enabled"
echo ""
echo "📊 Monitor Nginx: sudo systemctl status nginx"
echo "🔍 Check SSL: sudo certbot certificates"
echo "📱 View logs: sudo tail -f /var/log/nginx/access.log"
```

## scripts/ecosystem.config.js

```js
module.exports = {
  apps: [
    {
      name: 'cosmetics-backend',
      script: './backend/app.js',
      cwd: '/opt/cosmetics-ecommerce',
      instances: 1,
      autorestart: true,
      watch: false,
      max_memory_restart: '1G',
      env: {
        NODE_ENV: 'production',
        PORT: 5000
      },
      error_file: '/var/log/pm2/backend-error.log',
      out_file: '/var/log/pm2/backend-out.log',
      log_file: '/var/log/pm2/backend-combined.log'
    },
    {
      name: 'whatsapp-bot',
      script: './whatsapp-bot/whatsappClient.js',
      cwd: '/opt/cosmetics-ecommerce',
      instances: 1,
      autorestart: true,
      watch: false,
      max_memory_restart: '500M',
      env: {
        NODE_ENV: 'production'
      },
      error_file: '/var/log/pm2/whatsapp-error.log',
      out_file: '/var/log/pm2/whatsapp-out.log',
      log_file: '/var/log/pm2/whatsapp-combined.log'
    }
  ],

  deploy: {
    production: {
      user: 'deploy',
      host: 'your-server-ip',
      ref: 'origin/main',
      repo: 'git@github.com:yourusername/cosmetics-ecommerce.git',
      path: '/opt/cosmetics-ecommerce',
      'pre-deploy-local': '',
      'post-deploy': 'npm install --prefix backend && npm install --prefix whatsapp-bot && pm2 reload ecosystem.config.js --env production',
      'pre-setup': ''
    }
  }
};
```

## README.md

```markdown
# Cosmetics E-commerce Platform

A complete e-commerce solution with WhatsApp integration, built with Node.js, React, and PostgreSQL.

## Features

- 🛍️ **E-commerce**: Product catalog, shopping cart, order management
- 📱 **WhatsApp Integration**: Automated customer support and order tracking
- 💳 **Payment Gateway**: Cashfree payment integration
- 🏷️ **Label Printing**: Barcode label generation for products
- 🔐 **Authentication**: JWT-based user authentication
- 📊 **Admin Dashboard**: Product and order management
- 🐳 **Docker Ready**: Complete containerization support

## Quick Start

### Prerequisites

- Node.js 18+
- PostgreSQL 12+
- Docker & Docker Compose (optional)

### Local Development

1. **Clone the repository**
   \`\`\`bash
   git clone https://github.com/yourusername/cosmetics-ecommerce.git
   cd cosmetics-ecommerce
   \`\`\`

2. **Setup Backend**
   \`\`\`bash
   cd backend
   npm install
   cp .env.example .env
   # Edit .env with your configuration
   npm run migrate
   npm run dev
   \`\`\`

3. **Setup Frontend**
   \`\`\`bash
   cd frontend
   npm install
   npm start
   \`\`\`

4. **Setup WhatsApp Bot**
   \`\`\`bash
   cd whatsapp-bot
   npm install
   npm start
   # Scan QR code with WhatsApp Business app
   \`\`\`

### Docker Deployment

1. **Configure environment**
   \`\`\`bash
   cd deployment
   cp .env.example .env
   # Edit .env with your configuration
   \`\`\`

2. **Deploy with Docker**
   \`\`\`bash
   chmod +x start.sh
   ./start.sh
   \`\`\`

## Configuration

### Environment Variables

Create \`.env\` files in each directory with the following variables:

#### Backend (.env)
- \`DB_HOST\` - Database host
- \`DB_USER\` - Database username  
- \`DB_PASS\` - Database password
- \`DB_NAME\` - Database name
- \`JWT_SECRET\` - JWT signing secret
- \`CASHFREE_CLIENT_ID\` - Cashfree client ID
- \`CASHFREE_CLIENT_SECRET\` - Cashfree client secret
- \`BASE_URL\` - Application base URL

#### WhatsApp Bot (.env)
- \`BACKEND_URL\` - Backend API URL
- \`WEBSITE_URL\` - Website URL

### Database Setup

1. Create PostgreSQL database
2. Run migrations: \`npm run migrate\`
3. (Optional) Seed data: \`npm run seed\`

## API Documentation

### Authentication Endpoints
- \`POST /api/auth/register\` - User registration
- \`POST /api/auth/login\` - User login
- \`GET /api/auth/profile\` - Get user profile

### Product Endpoints
- \`GET /api/products\` - List products
- \`GET /api/products/:id\` - Get product by ID
- \`POST /api/products\` - Create product (admin)
- \`PUT /api/products/:id\` - Update product (admin)

### Order Endpoints
- \`POST /api/orders\` - Create order
- \`GET /api/orders/:id\` - Get order by ID
- \`GET /api/orders/status/:orderId\` - Get order status
- \`GET /api/orders/myorders\` - Get user orders

### Payment Endpoints
- \`POST /api/payment/create-order\` - Create payment order
- \`POST /api/payment/callback\` - Payment webhook
- \`GET /api/payment/verify/:orderId\` - Verify payment

## WhatsApp Bot Commands

Users can interact with the bot using these commands:

- \`help\` - Show available commands
- \`order status\` - Check order status
- \`ORD123456\` - Get specific order details
- \`contact\` - Get support information
- \`products\` - View product categories

## Deployment

### Production Deployment

1. **Setup VPS** (Ubuntu 20.04+ recommended)
2. **Install dependencies**: Docker, Docker Compose
3. **Clone repository**: \`git clone <repo-url>\`
4. **Configure**: Copy and edit \`.env\` files
5. **Deploy**: Run \`./deployment/start.sh\`
6. **Setup domain**: Configure DNS and SSL

### SSL Setup

Use the provided script to setup Nginx with SSL:

\`\`\`bash
chmod +x scripts/setup-nginx-ssl.sh
sudo ./scripts/setup-nginx-ssl.sh
\`\`\`

## Monitoring

### Health Checks
- Backend: \`GET /health\`
- Database: Built-in PostgreSQL health checks
- WhatsApp: Process monitoring via PM2/Docker

### Logs
- Backend: \`docker-compose logs backend\`
- WhatsApp Bot: \`docker-compose logs whatsapp-bot\`
- Nginx: \`sudo tail -f /var/log/nginx/access.log\`

## Development

### File Structure
\`\`\`
cosmetics-ecommerce/
├── backend/          # Node.js API server
├── frontend/         # React web application  
├── whatsapp-bot/     # WhatsApp automation
├── deployment/       # Docker compose & configs
├── scripts/          # Setup & deployment scripts
└── .github/          # CI/CD workflows
\`\`\`

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests (if applicable)
5. Submit a pull request

## Support

For support and questions:
- 📧 Email: support@yourcompany.com
- 📱 WhatsApp: Message the bot
- 🐛 Issues: GitHub Issues

## License

MIT License - see LICENSE file for details.
\`\`\`

## .gitignore

\`\`\`
# Dependencies
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Production builds
build/
dist/

# Environment files
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Database
*.db
*.sqlite

# Logs
logs/
*.log

# Runtime data
pids/
*.pid
*.seed
*.pid.lock

# Coverage
coverage/
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
*~

# Temporary files
tmp/
temp/

# WhatsApp auth
auth_info.json

# Labels
public/labels/*.zip
public/labels/*.txt
public/labels/*.png

# Docker volumes
postgres_data/
backend_labels/
whatsapp_auth/

# SSL certificates
*.pem
*.key
*.crt

# Backup files
*.backup
*.bak
\`\`\`
```