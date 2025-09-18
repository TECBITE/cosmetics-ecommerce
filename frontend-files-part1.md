# FRONTEND REACT FILES

## frontend/package.json

```json
{
  "name": "cosmetics-frontend",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "@testing-library/jest-dom": "^5.16.4",
    "@testing-library/react": "^13.3.0",
    "@testing-library/user-event": "^13.5.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.3.0",
    "react-scripts": "5.0.1",
    "web-vitals": "^2.1.4"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ]
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  },
  "proxy": "http://localhost:5000"
}
```

## frontend/public/index.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="icon" href="%PUBLIC_URL%/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="theme-color" content="#000000" />
    <meta name="description" content="Cosmetics E-commerce Platform" />
    <title>Cosmetics Shop</title>
    <style>
      * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
      }
      
      body {
        font-family: 'Arial', sans-serif;
        line-height: 1.6;
        color: #333;
        background-color: #f8f9fa;
      }
      
      .container {
        max-width: 1200px;
        margin: 0 auto;
        padding: 0 20px;
      }
      
      .btn {
        background: #007bff;
        color: white;
        padding: 10px 20px;
        border: none;
        border-radius: 5px;
        cursor: pointer;
        text-decoration: none;
        display: inline-block;
        transition: background-color 0.3s;
      }
      
      .btn:hover {
        background: #0056b3;
      }
      
      .btn-danger {
        background: #dc3545;
      }
      
      .btn-danger:hover {
        background: #c82333;
      }
      
      .form-group {
        margin-bottom: 15px;
      }
      
      .form-control {
        width: 100%;
        padding: 10px;
        border: 1px solid #ddd;
        border-radius: 5px;
        font-size: 16px;
      }
      
      .form-control:focus {
        outline: none;
        border-color: #007bff;
        box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.25);
      }
      
      .card {
        background: white;
        border-radius: 8px;
        padding: 20px;
        margin-bottom: 20px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
      }
      
      .product-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
        gap: 20px;
        margin: 20px 0;
      }
      
      .product-card {
        border: 1px solid #ddd;
        border-radius: 8px;
        overflow: hidden;
        transition: transform 0.3s, box-shadow 0.3s;
      }
      
      .product-card:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
      }
      
      .navbar {
        background: #343a40;
        color: white;
        padding: 1rem 0;
      }
      
      .navbar a {
        color: white;
        text-decoration: none;
        margin-right: 20px;
      }
      
      .navbar a:hover {
        text-decoration: underline;
      }
      
      .alert {
        padding: 12px;
        border-radius: 5px;
        margin-bottom: 20px;
      }
      
      .alert-success {
        background: #d4edda;
        border: 1px solid #c3e6cb;
        color: #155724;
      }
      
      .alert-error {
        background: #f8d7da;
        border: 1px solid #f5c6cb;
        color: #721c24;
      }
    </style>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>
  </body>
</html>
```

## frontend/src/index.js

```js
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

## frontend/src/App.js

```js
import React, { useState, useEffect } from 'react';
import { BrowserRouter as Router, Routes, Route, Link } from 'react-router-dom';
import ProductList from './components/ProductList';
import ProductDetail from './components/ProductDetail';
import Register from './components/Register';
import Login from './components/Login';
import Checkout from './components/Checkout';
import PaymentSuccess from './components/PaymentSuccess';
import LabelPrint from './components/LabelPrint';

function App() {
  const [user, setUser] = useState(null);
  const [cart, setCart] = useState([]);

  useEffect(() => {
    // Check for stored user token
    const token = localStorage.getItem('token');
    const userData = localStorage.getItem('user');
    
    if (token && userData) {
      setUser(JSON.parse(userData));
    }

    // Load cart from localStorage
    const savedCart = localStorage.getItem('cart');
    if (savedCart) {
      setCart(JSON.parse(savedCart));
    }
  }, []);

  const handleLogin = (userData) => {
    setUser(userData);
    localStorage.setItem('user', JSON.stringify(userData));
  };

  const handleLogout = () => {
    setUser(null);
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    setCart([]);
    localStorage.removeItem('cart');
  };

  const addToCart = (product) => {
    const existingItem = cart.find(item => item.id === product.id);
    let newCart;
    
    if (existingItem) {
      newCart = cart.map(item => 
        item.id === product.id 
          ? { ...item, quantity: item.quantity + 1 }
          : item
      );
    } else {
      newCart = [...cart, { ...product, quantity: 1 }];
    }
    
    setCart(newCart);
    localStorage.setItem('cart', JSON.stringify(newCart));
  };

  const removeFromCart = (productId) => {
    const newCart = cart.filter(item => item.id !== productId);
    setCart(newCart);
    localStorage.setItem('cart', JSON.stringify(newCart));
  };

  const updateQuantity = (productId, quantity) => {
    if (quantity <= 0) {
      removeFromCart(productId);
      return;
    }
    
    const newCart = cart.map(item =>
      item.id === productId ? { ...item, quantity } : item
    );
    setCart(newCart);
    localStorage.setItem('cart', JSON.stringify(newCart));
  };

  const getTotalItems = () => {
    return cart.reduce((total, item) => total + item.quantity, 0);
  };

  const getTotalPrice = () => {
    return cart.reduce((total, item) => total + (item.price * item.quantity), 0).toFixed(2);
  };

  return (
    <Router>
      <div className="App">
        {/* Navigation Bar */}
        <nav className="navbar">
          <div className="container">
            <Link to="/" style={{ fontSize: '1.5em', fontWeight: 'bold' }}>
              🛍️ Cosmetics Shop
            </Link>
            
            <div style={{ display: 'flex', alignItems: 'center', gap: '20px' }}>
              <Link to="/">Products</Link>
              
              {user ? (
                <>
                  <Link to="/cart">Cart ({getTotalItems()})</Link>
                  {user.role === 'admin' && <Link to="/labels">Print Labels</Link>}
                  <span>Hello, {user.name}!</span>
                  <button onClick={handleLogout} className="btn btn-danger">
                    Logout
                  </button>
                </>
              ) : (
                <>
                  <Link to="/login">Login</Link>
                  <Link to="/register">Register</Link>
                </>
              )}
            </div>
          </div>
        </nav>

        {/* Main Content */}
        <div className="container">
          <Routes>
            <Route 
              path="/" 
              element={<ProductList addToCart={addToCart} />} 
            />
            <Route 
              path="/product/:id" 
              element={<ProductDetail addToCart={addToCart} />} 
            />
            <Route 
              path="/register" 
              element={<Register onRegister={handleLogin} />} 
            />
            <Route 
              path="/login" 
              element={<Login onLogin={handleLogin} />} 
            />
            <Route 
              path="/cart" 
              element={
                <Checkout 
                  cart={cart}
                  user={user}
                  removeFromCart={removeFromCart}
                  updateQuantity={updateQuantity}
                  getTotalPrice={getTotalPrice}
                />
              } 
            />
            <Route 
              path="/payment/success" 
              element={<PaymentSuccess />} 
            />
            {user && user.role === 'admin' && (
              <Route 
                path="/labels" 
                element={<LabelPrint />} 
              />
            )}
          </Routes>
        </div>
      </div>
    </Router>
  );
}

export default App;
```

## frontend/src/components/ProductList.js

```js
import React, { useEffect, useState } from 'react';
import { Link } from 'react-router-dom';

function ProductList({ addToCart }) {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');

  useEffect(() => {
    fetchProducts();
  }, []);

  const fetchProducts = async () => {
    try {
      const response = await fetch('/api/products');
      if (response.ok) {
        const data = await response.json();
        setProducts(data);
      } else {
        setError('Failed to fetch products');
      }
    } catch (err) {
      setError('Error connecting to server');
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div className="card">Loading products...</div>;
  if (error) return <div className="alert alert-error">{error}</div>;

  return (
    <div>
      <h1>Our Products</h1>
      
      {products.length === 0 ? (
        <div className="card">No products available</div>
      ) : (
        <div className="product-grid">
          {products.map(product => (
            <div key={product.id} className="product-card">
              {product.imageUrl && (
                <img 
                  src={product.imageUrl} 
                  alt={product.name}
                  style={{ width: '100%', height: '200px', objectFit: 'cover' }}
                />
              )}
              
              <div style={{ padding: '15px' }}>
                <h3>
                  <Link 
                    to={`/product/${product.id}`}
                    style={{ color: '#333', textDecoration: 'none' }}
                  >
                    {product.name}
                  </Link>
                </h3>
                
                <p style={{ color: '#666', marginBottom: '10px' }}>
                  {product.description?.substring(0, 100)}
                  {product.description?.length > 100 && '...'}
                </p>
                
                <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
                  <span style={{ fontSize: '1.2em', fontWeight: 'bold', color: '#007bff' }}>
                    ₹{product.price}
                  </span>
                  
                  <span style={{ color: product.stock > 0 ? '#28a745' : '#dc3545' }}>
                    {product.stock > 0 ? `${product.stock} in stock` : 'Out of stock'}
                  </span>
                </div>
                
                <button 
                  className="btn"
                  style={{ width: '100%', marginTop: '10px' }}
                  onClick={() => addToCart(product)}
                  disabled={product.stock === 0}
                >
                  {product.stock > 0 ? 'Add to Cart' : 'Out of Stock'}
                </button>
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

export default ProductList;
```