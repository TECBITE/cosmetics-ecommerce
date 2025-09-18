# FRONTEND COMPONENTS PART 3 & DEPLOYMENT FILES

## frontend/src/components/PaymentSuccess.js

```js
import React from 'react';
import { Link } from 'react-router-dom';

function PaymentSuccess() {
  return (
    <div style={{ textAlign: 'center', marginTop: '50px' }}>
      <div className="card" style={{ maxWidth: '500px', margin: '0 auto' }}>
        <div style={{ fontSize: '4em', marginBottom: '20px' }}>✅</div>
        
        <h1 style={{ color: '#28a745', marginBottom: '20px' }}>
          Payment Successful!
        </h1>
        
        <p style={{ marginBottom: '30px', fontSize: '1.1em' }}>
          Thank you for your purchase! Your order has been placed successfully.
          We will send you order updates via WhatsApp and email.
        </p>
        
        <div style={{ marginBottom: '30px' }}>
          <p>📱 You can check your order status by messaging us on WhatsApp</p>
          <p>📧 A confirmation email has been sent to your email address</p>
        </div>
        
        <Link to="/" className="btn" style={{ marginRight: '10px' }}>
          Continue Shopping
        </Link>
      </div>
    </div>
  );
}

export default PaymentSuccess;
```

## frontend/src/components/LabelPrint.js

```js
import React, { useEffect, useState } from 'react';

function LabelPrint() {
  const [products, setProducts] = useState([]);
  const [selected, setSelected] = useState(new Set());
  const [message, setMessage] = useState('');
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    fetchProducts();
  }, []);

  const fetchProducts = async () => {
    try {
      const response = await fetch('/api/products', {
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        }
      });
      
      if (response.ok) {
        const data = await response.json();
        setProducts(data);
      } else {
        setMessage('Failed to load products');
      }
    } catch (error) {
      setMessage('Error loading products');
    }
  };

  const toggleSelect = (productId) => {
    const newSelected = new Set(selected);
    if (newSelected.has(productId)) {
      newSelected.delete(productId);
    } else {
      newSelected.add(productId);
    }
    setSelected(newSelected);
  };

  const selectAll = () => {
    if (selected.size === products.length) {
      setSelected(new Set());
    } else {
      setSelected(new Set(products.map(p => p.id)));
    }
  };

  const handlePrintBatch = async () => {
    if (selected.size === 0) {
      setMessage('Please select at least one product');
      return;
    }

    setLoading(true);
    setMessage('Generating labels...');

    try {
      const response = await fetch('/api/label/print-batch', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        },
        body: JSON.stringify({ productIds: Array.from(selected) }),
      });

      const data = await response.json();

      if (response.ok) {
        setMessage(`Labels generated successfully! ${data.message}`);
        
        // Auto download
        const link = document.createElement('a');
        link.href = data.labelZipUrl;
        link.download = 'product-labels.zip';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        
        setSelected(new Set());
      } else {
        setMessage('Error: ' + data.message);
      }
    } catch (error) {
      setMessage('Error generating labels: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  const handlePrintSingle = async (productId) => {
    try {
      const response = await fetch(`/api/label/print-single/${productId}`, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${localStorage.getItem('token')}`
        }
      });

      const data = await response.json();

      if (response.ok) {
        // Auto download
        const link = document.createElement('a');
        link.href = data.labelUrl;
        link.download = `product-${productId}-label.txt`;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        
        setMessage(data.message);
      } else {
        setMessage('Error: ' + data.message);
      }
    } catch (error) {
      setMessage('Error generating label: ' + error.message);
    }
  };

  return (
    <div style={{ marginTop: '20px' }}>
      <h1>Label Printing</h1>
      <p>Generate and print barcode labels for your products</p>
      
      {message && (
        <div className={`alert ${message.includes('successfully') || message.includes('Generated') ? 'alert-success' : 'alert-error'}`}>
          {message}
        </div>
      )}
      
      <div className="card">
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: '20px' }}>
          <h3>Select Products for Batch Printing</h3>
          
          <div>
            <button 
              onClick={selectAll}
              className="btn"
              style={{ marginRight: '10px' }}
            >
              {selected.size === products.length ? 'Deselect All' : 'Select All'}
            </button>
            
            <button 
              onClick={handlePrintBatch}
              className="btn"
              disabled={loading || selected.size === 0}
            >
              {loading ? 'Generating...' : `Print Selected (${selected.size})`}
            </button>
          </div>
        </div>
        
        {products.length === 0 ? (
          <p>No products available</p>
        ) : (
          <div style={{ maxHeight: '400px', overflowY: 'auto' }}>
            {products.map(product => (
              <div key={product.id} style={{ 
                display: 'flex', 
                justifyContent: 'space-between', 
                alignItems: 'center',
                padding: '10px',
                borderBottom: '1px solid #eee'
              }}>
                <label style={{ display: 'flex', alignItems: 'center', cursor: 'pointer', flex: 1 }}>
                  <input
                    type="checkbox"
                    onChange={() => toggleSelect(product.id)}
                    checked={selected.has(product.id)}
                    style={{ marginRight: '10px' }}
                  />
                  
                  <div>
                    <strong>{product.name}</strong>
                    <div style={{ fontSize: '0.9em', color: '#666' }}>
                      ₹{product.price} | Stock: {product.stock} | SKU: {product.sku || product.id}
                    </div>
                  </div>
                </label>
                
                <button 
                  onClick={() => handlePrintSingle(product.id)}
                  className="btn"
                  style={{ padding: '5px 15px' }}
                >
                  Print Single
                </button>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  );
}

export default LabelPrint;
```

## frontend/Dockerfile

```dockerfile
# Build stage
FROM node:18-alpine as build

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --silent

# Copy source code
COPY . .

# Build the app
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy build files
COPY --from=build /app/build /usr/share/nginx/html

# Copy nginx config
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## frontend/nginx.conf

```nginx
server {
    listen 80;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    
    # API proxy to backend
    location /api {
        proxy_pass http://backend:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Static files
    location /labels {
        proxy_pass http://backend:5000;
    }
    
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

---

# DEPLOYMENT FILES

## deployment/docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: cosmetics-db
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - cosmetics-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Backend API
  backend:
    build:
      context: ../backend
      dockerfile: Dockerfile
    container_name: cosmetics-backend
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - DB_USER=${DB_USER}
      - DB_PASS=${DB_PASS}
      - DB_NAME=${DB_NAME}
      - JWT_SECRET=${JWT_SECRET}
      - CASHFREE_CLIENT_ID=${CASHFREE_CLIENT_ID}
      - CASHFREE_CLIENT_SECRET=${CASHFREE_CLIENT_SECRET}
      - BASE_URL=${BASE_URL}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - cosmetics-network
    restart: unless-stopped
    volumes:
      - backend_labels:/usr/src/app/public/labels
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # WhatsApp Bot
  whatsapp-bot:
    build:
      context: ../whatsapp-bot
      dockerfile: Dockerfile
    container_name: whatsapp-bot
    environment:
      - NODE_ENV=production
      - BACKEND_URL=http://backend:5000
      - WEBSITE_URL=${BASE_URL}
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - cosmetics-network
    restart: unless-stopped
    volumes:
      - whatsapp_auth:/usr/src/app

  # Frontend
  frontend:
    build:
      context: ../frontend
      dockerfile: Dockerfile
    container_name: cosmetics-frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - cosmetics-network
    restart: unless-stopped

  # Nginx (Optional - for SSL termination)
  nginx:
    image: nginx:alpine
    container_name: cosmetics-nginx
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - frontend
    networks:
      - cosmetics-network
    restart: unless-stopped

volumes:
  postgres_data:
  backend_labels:
  whatsapp_auth:

networks:
  cosmetics-network:
    driver: bridge
```

## deployment/.env.example

```env
# Database Configuration
DB_HOST=postgres
DB_USER=cosmetics_user
DB_PASS=secure_password_here
DB_NAME=cosmetics_ecommerce

# JWT Secret (generate a strong secret)
JWT_SECRET=your_super_secure_jwt_secret_key_here_make_it_long_and_random

# Cashfree Payment Gateway
CASHFREE_CLIENT_ID=your_cashfree_client_id
CASHFREE_CLIENT_SECRET=your_cashfree_client_secret

# Application URL
BASE_URL=https://yourdomain.com

# Docker Compose Project Name
COMPOSE_PROJECT_NAME=cosmetics-ecommerce
```

## deployment/start.sh

```bash
#!/bin/bash

echo "🚀 Starting Cosmetics E-commerce Platform..."

# Check if .env file exists
if [ ! -f .env ]; then
    echo "❌ .env file not found. Please copy .env.example to .env and configure it."
    exit 1
fi

# Pull latest Docker images
echo "📦 Pulling latest Docker images..."
docker-compose pull

# Run database migrations
echo "🗄️ Running database migrations..."
docker-compose run --rm backend npm run migrate

# Start all services
echo "🐳 Starting Docker containers..."
docker-compose up -d

# Wait for services to be ready
echo "⏳ Waiting for services to start..."
sleep 30

# Check service health
echo "🏥 Checking service health..."
docker-compose ps

echo "✅ Deployment complete!"
echo "🌐 Frontend: http://localhost"
echo "🔧 Backend API: http://localhost/api"
echo "📱 WhatsApp bot should be running (check logs: docker-compose logs whatsapp-bot)"

echo ""
echo "📋 Next steps:"
echo "1. Scan WhatsApp QR code: docker-compose logs whatsapp-bot"
echo "2. Create an admin user via API or database"
echo "3. Add some products to start selling"
echo ""
echo "📊 Monitor logs: docker-compose logs -f"
echo "🛑 Stop services: docker-compose down"
```

## deployment/migrate.sh

```bash
#!/bin/bash

echo "🗄️ Running database migrations..."

# Run migrations in backend container
docker-compose exec backend npm run migrate

echo "✅ Database migrations completed"
```

## deployment/nginx.conf (for SSL termination)

```nginx
events {
    worker_connections 1024;
}

http {
    upstream frontend {
        server frontend:80;
    }

    server {
        listen 80;
        server_name yourdomain.com www.yourdomain.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name yourdomain.com www.yourdomain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```