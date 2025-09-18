# FRONTEND COMPONENTS PART 2

## frontend/src/components/ProductDetail.js

```js
import React, { useEffect, useState } from 'react';
import { useParams, Link } from 'react-router-dom';

function ProductDetail({ addToCart }) {
  const { id } = useParams();
  const [product, setProduct] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState('');
  const [quantity, setQuantity] = useState(1);

  useEffect(() => {
    fetchProduct();
  }, [id]);

  const fetchProduct = async () => {
    try {
      const response = await fetch(`/api/products/${id}`);
      if (response.ok) {
        const data = await response.json();
        setProduct(data);
      } else {
        setError('Product not found');
      }
    } catch (err) {
      setError('Error loading product');
    } finally {
      setLoading(false);
    }
  };

  const handleAddToCart = () => {
    for (let i = 0; i < quantity; i++) {
      addToCart(product);
    }
    alert(`Added ${quantity} ${product.name}(s) to cart!`);
  };

  if (loading) return <div className="card">Loading product...</div>;
  if (error) return <div className="alert alert-error">{error}</div>;
  if (!product) return <div className="alert alert-error">Product not found</div>;

  return (
    <div style={{ marginTop: '20px' }}>
      <Link to="/" className="btn" style={{ marginBottom: '20px' }}>
        ← Back to Products
      </Link>
      
      <div className="card">
        <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: '30px' }}>
          <div>
            {product.imageUrl ? (
              <img 
                src={product.imageUrl} 
                alt={product.name}
                style={{ width: '100%', borderRadius: '8px' }}
              />
            ) : (
              <div style={{ 
                height: '300px', 
                background: '#f0f0f0', 
                display: 'flex', 
                alignItems: 'center', 
                justifyContent: 'center',
                borderRadius: '8px'
              }}>
                No Image Available
              </div>
            )}
          </div>
          
          <div>
            <h1>{product.name}</h1>
            
            <p style={{ fontSize: '1.5em', color: '#007bff', margin: '15px 0' }}>
              ₹{product.price}
            </p>
            
            <p style={{ marginBottom: '20px', lineHeight: '1.6' }}>
              {product.description}
            </p>
            
            <div style={{ marginBottom: '20px' }}>
              <strong>Category:</strong> {product.category || 'General'}
            </div>
            
            <div style={{ marginBottom: '20px' }}>
              <strong>Stock:</strong> 
              <span style={{ color: product.stock > 0 ? '#28a745' : '#dc3545', marginLeft: '10px' }}>
                {product.stock > 0 ? `${product.stock} available` : 'Out of stock'}
              </span>
            </div>

            {product.sku && (
              <div style={{ marginBottom: '20px' }}>
                <strong>SKU:</strong> {product.sku}
              </div>
            )}
            
            {product.stock > 0 && (
              <div style={{ marginBottom: '20px' }}>
                <label style={{ display: 'block', marginBottom: '10px' }}>
                  <strong>Quantity:</strong>
                </label>
                <input 
                  type="number" 
                  min="1" 
                  max={product.stock}
                  value={quantity}
                  onChange={(e) => setQuantity(parseInt(e.target.value) || 1)}
                  className="form-control"
                  style={{ width: '100px', display: 'inline-block', marginRight: '15px' }}
                />
                <button 
                  className="btn"
                  onClick={handleAddToCart}
                >
                  Add to Cart
                </button>
              </div>
            )}
            
            {product.stock === 0 && (
              <div className="alert alert-error">
                This product is currently out of stock
              </div>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}

export default ProductDetail;
```

## frontend/src/components/Register.js

```js
import React, { useState } from 'react';
import { Link } from 'react-router-dom';

function Register({ onRegister }) {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    password: '',
    phone: ''
  });
  const [message, setMessage] = useState('');
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setMessage('');
    setLoading(true);

    try {
      const response = await fetch('/api/auth/register', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData),
      });

      const data = await response.json();
      
      if (response.ok) {
        localStorage.setItem('token', data.token);
        setMessage('Registration successful!');
        onRegister(data);
      } else {
        setMessage(data.error || 'Registration failed');
      }
    } catch (error) {
      setMessage('Error: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ maxWidth: '400px', margin: '40px auto' }}>
      <div className="card">
        <h2 style={{ textAlign: 'center', marginBottom: '30px' }}>Create Account</h2>
        
        {message && (
          <div className={`alert ${message.includes('successful') ? 'alert-success' : 'alert-error'}`}>
            {message}
          </div>
        )}
        
        <form onSubmit={handleSubmit}>
          <div className="form-group">
            <label>Full Name</label>
            <input 
              type="text" 
              name="name" 
              placeholder="Enter your full name"
              value={formData.name}
              onChange={handleChange}
              required 
              className="form-control"
            />
          </div>
          
          <div className="form-group">
            <label>Email Address</label>
            <input 
              type="email" 
              name="email" 
              placeholder="Enter your email"
              value={formData.email}
              onChange={handleChange}
              required 
              className="form-control"
            />
          </div>
          
          <div className="form-group">
            <label>Phone Number</label>
            <input 
              type="tel" 
              name="phone" 
              placeholder="Enter your phone number"
              value={formData.phone}
              onChange={handleChange}
              className="form-control"
            />
          </div>
          
          <div className="form-group">
            <label>Password</label>
            <input 
              type="password" 
              name="password" 
              placeholder="Create a password"
              value={formData.password}
              onChange={handleChange}
              required 
              className="form-control"
            />
          </div>
          
          <button 
            type="submit" 
            className="btn" 
            style={{ width: '100%', marginBottom: '15px' }}
            disabled={loading}
          >
            {loading ? 'Creating Account...' : 'Register'}
          </button>
        </form>
        
        <p style={{ textAlign: 'center' }}>
          Already have an account? <Link to="/login">Login here</Link>
        </p>
      </div>
    </div>
  );
}

export default Register;
```

## frontend/src/components/Login.js

```js
import React, { useState } from 'react';
import { Link } from 'react-router-dom';

function Login({ onLogin }) {
  const [formData, setFormData] = useState({
    email: '',
    password: ''
  });
  const [message, setMessage] = useState('');
  const [loading, setLoading] = useState(false);

  const handleChange = (e) => {
    setFormData({ ...formData, [e.target.name]: e.target.value });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setMessage('');
    setLoading(true);

    try {
      const response = await fetch('/api/auth/login', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData),
      });

      const data = await response.json();
      
      if (response.ok) {
        localStorage.setItem('token', data.token);
        setMessage('Login successful!');
        onLogin(data);
      } else {
        setMessage(data.error || 'Login failed');
      }
    } catch (error) {
      setMessage('Error: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ maxWidth: '400px', margin: '40px auto' }}>
      <div className="card">
        <h2 style={{ textAlign: 'center', marginBottom: '30px' }}>Login</h2>
        
        {message && (
          <div className={`alert ${message.includes('successful') ? 'alert-success' : 'alert-error'}`}>
            {message}
          </div>
        )}
        
        <form onSubmit={handleSubmit}>
          <div className="form-group">
            <label>Email Address</label>
            <input 
              type="email" 
              name="email" 
              placeholder="Enter your email"
              value={formData.email}
              onChange={handleChange}
              required 
              className="form-control"
            />
          </div>
          
          <div className="form-group">
            <label>Password</label>
            <input 
              type="password" 
              name="password" 
              placeholder="Enter your password"
              value={formData.password}
              onChange={handleChange}
              required 
              className="form-control"
            />
          </div>
          
          <button 
            type="submit" 
            className="btn" 
            style={{ width: '100%', marginBottom: '15px' }}
            disabled={loading}
          >
            {loading ? 'Logging in...' : 'Login'}
          </button>
        </form>
        
        <p style={{ textAlign: 'center' }}>
          Don't have an account? <Link to="/register">Register here</Link>
        </p>
      </div>
    </div>
  );
}

export default Login;
```

## frontend/src/components/Checkout.js

```js
import React, { useState } from 'react';

function Checkout({ cart, user, removeFromCart, updateQuantity, getTotalPrice }) {
  const [loading, setLoading] = useState(false);
  const [message, setMessage] = useState('');
  const [customerInfo, setCustomerInfo] = useState({
    name: user?.name || '',
    email: user?.email || '',
    phone: user?.phone || '',
    address: ''
  });

  const handleChange = (e) => {
    setCustomerInfo({ ...customerInfo, [e.target.name]: e.target.value });
  };

  const handlePlaceOrder = async () => {
    if (cart.length === 0) {
      setMessage('Your cart is empty');
      return;
    }

    if (!customerInfo.name || !customerInfo.email || !customerInfo.phone) {
      setMessage('Please fill in all required fields');
      return;
    }

    setLoading(true);
    setMessage('');

    try {
      // Create order
      const orderData = {
        orderItems: cart.map(item => ({
          productId: item.id,
          name: item.name,
          price: item.price,
          quantity: item.quantity
        })),
        totalPrice: parseFloat(getTotalPrice()),
        customerName: customerInfo.name,
        customerEmail: customerInfo.email,
        customerPhone: customerInfo.phone,
        shippingAddress: customerInfo.address
      };

      const orderResponse = await fetch('/api/orders', {
        method: 'POST',
        headers: { 
          'Content-Type': 'application/json',
          'Authorization': user ? `Bearer ${localStorage.getItem('token')}` : ''
        },
        body: JSON.stringify(orderData),
      });

      if (!orderResponse.ok) {
        throw new Error('Failed to create order');
      }

      const order = await orderResponse.json();

      // Create payment
      const paymentData = {
        amount: parseFloat(getTotalPrice()),
        orderId: order.orderId,
        customerName: customerInfo.name,
        customerEmail: customerInfo.email,
        customerPhone: customerInfo.phone
      };

      const paymentResponse = await fetch('/api/payment/create-order', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(paymentData),
      });

      const paymentResult = await paymentResponse.json();

      if (paymentResponse.ok) {
        // Redirect to payment gateway (in real implementation)
        alert(`Order placed successfully! Order ID: ${order.orderId}\nRedirecting to payment...`);
        
        // Clear cart
        localStorage.removeItem('cart');
        window.location.href = '/payment/success';
      } else {
        setMessage(paymentResult.message || 'Payment processing failed');
      }

    } catch (error) {
      setMessage('Error placing order: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  if (cart.length === 0) {
    return (
      <div className="card" style={{ marginTop: '20px', textAlign: 'center' }}>
        <h2>Your Cart is Empty</h2>
        <p>Add some products to your cart to continue shopping.</p>
      </div>
    );
  }

  return (
    <div style={{ marginTop: '20px' }}>
      <h1>Shopping Cart</h1>
      
      {message && (
        <div className={`alert ${message.includes('successfully') ? 'alert-success' : 'alert-error'}`}>
          {message}
        </div>
      )}
      
      <div style={{ display: 'grid', gridTemplateColumns: '2fr 1fr', gap: '30px' }}>
        {/* Cart Items */}
        <div>
          <div className="card">
            <h3>Order Items</h3>
            {cart.map(item => (
              <div key={item.id} style={{ 
                display: 'flex', 
                justifyContent: 'space-between', 
                alignItems: 'center',
                padding: '15px 0',
                borderBottom: '1px solid #eee'
              }}>
                <div>
                  <h4>{item.name}</h4>
                  <p>₹{item.price} each</p>
                </div>
                
                <div style={{ display: 'flex', alignItems: 'center', gap: '10px' }}>
                  <button 
                    onClick={() => updateQuantity(item.id, item.quantity - 1)}
                    className="btn"
                    style={{ padding: '5px 10px' }}
                  >
                    -
                  </button>
                  
                  <span style={{ minWidth: '30px', textAlign: 'center' }}>
                    {item.quantity}
                  </span>
                  
                  <button 
                    onClick={() => updateQuantity(item.id, item.quantity + 1)}
                    className="btn"
                    style={{ padding: '5px 10px' }}
                  >
                    +
                  </button>
                  
                  <button 
                    onClick={() => removeFromCart(item.id)}
                    className="btn btn-danger"
                    style={{ padding: '5px 10px', marginLeft: '10px' }}
                  >
                    Remove
                  </button>
                </div>
                
                <div style={{ fontWeight: 'bold' }}>
                  ₹{(item.price * item.quantity).toFixed(2)}
                </div>
              </div>
            ))}
            
            <div style={{ textAlign: 'right', marginTop: '20px', fontSize: '1.2em' }}>
              <strong>Total: ₹{getTotalPrice()}</strong>
            </div>
          </div>
        </div>
        
        {/* Customer Information */}
        <div>
          <div className="card">
            <h3>Customer Information</h3>
            
            <div className="form-group">
              <label>Full Name *</label>
              <input 
                type="text"
                name="name"
                value={customerInfo.name}
                onChange={handleChange}
                className="form-control"
                required
              />
            </div>
            
            <div className="form-group">
              <label>Email *</label>
              <input 
                type="email"
                name="email"
                value={customerInfo.email}
                onChange={handleChange}
                className="form-control"
                required
              />
            </div>
            
            <div className="form-group">
              <label>Phone *</label>
              <input 
                type="tel"
                name="phone"
                value={customerInfo.phone}
                onChange={handleChange}
                className="form-control"
                required
              />
            </div>
            
            <div className="form-group">
              <label>Shipping Address</label>
              <textarea 
                name="address"
                value={customerInfo.address}
                onChange={handleChange}
                className="form-control"
                rows="3"
                placeholder="Enter your complete address"
              />
            </div>
            
            <button 
              onClick={handlePlaceOrder}
              className="btn"
              style={{ width: '100%' }}
              disabled={loading}
            >
              {loading ? 'Processing...' : `Place Order - ₹${getTotalPrice()}`}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}

export default Checkout;
```