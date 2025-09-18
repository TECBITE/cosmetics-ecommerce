# BACKEND FILES

## backend/package.json

```json
{
  "name": "cosmetics-backend",
  "version": "1.0.0",
  "description": "E-commerce backend with WhatsApp integration",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js",
    "migrate": "npx sequelize-cli db:migrate"
  },
  "dependencies": {
    "express": "^4.18.2",
    "sequelize": "^6.32.1",
    "pg": "^8.11.0",
    "pg-hstore": "^2.3.4",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.0",
    "dotenv": "^16.0.3",
    "express-async-handler": "^1.2.0",
    "cors": "^2.8.5",
    "helmet": "^6.1.5",
    "morgan": "^1.10.0",
    "jszip": "^3.10.1",
    "node-fetch": "^2.6.11"
  },
  "devDependencies": {
    "sequelize-cli": "^6.6.1",
    "nodemon": "^2.0.22"
  }
}
```

## backend/.sequelizerc

```js
const path = require('path');

module.exports = {
  'config': path.resolve('config', 'config.js'),
  'models-path': path.resolve('models'),
  'seeders-path': path.resolve('seeders'),
  'migrations-path': path.resolve('migrations')
};
```

## backend/config/config.js

```js
require('dotenv').config();

module.exports = {
  development: {
    username: process.env.DB_USER || "postgres",
    password: process.env.DB_PASS || "password",
    database: process.env.DB_NAME || "cosmetics_dev",
    host: process.env.DB_HOST || "127.0.0.1",
    dialect: "postgres"
  },
  test: {
    username: process.env.DB_USER || "postgres",
    password: process.env.DB_PASS || "password",
    database: process.env.DB_NAME_TEST || "cosmetics_test",
    host: process.env.DB_HOST || "127.0.0.1",
    dialect: "postgres"
  },
  production: {
    username: process.env.DB_USER,
    password: process.env.DB_PASS,
    database: process.env.DB_NAME,
    host: process.env.DB_HOST,
    dialect: "postgres",
    dialectOptions: {
      ssl: {
        require: true,
        rejectUnauthorized: false
      }
    }
  }
};
```

## backend/utils/loadSecrets.js

```js
const fs = require('fs');

function loadDockerSecret(secretName) {
  try {
    return fs.readFileSync(`/run/secrets/${secretName}`, 'utf8').trim();
  } catch {
    return process.env[secretName.toUpperCase()] || '';
  }
}

// Load secrets into process.env
process.env.MONGODB_URI = loadDockerSecret('mongodb_uri');
process.env.JWT_SECRET = loadDockerSecret('jwt_secret');
process.env.CASHFREE_CLIENT_ID = loadDockerSecret('cashfree_client_id');
process.env.CASHFREE_CLIENT_SECRET = loadDockerSecret('cashfree_client_secret');

module.exports = { loadDockerSecret };
```

## backend/models/index.js

```js
'use strict';

const fs = require('fs');
const path = require('path');
const Sequelize = require('sequelize');
const basename = path.basename(__filename);
const env = process.env.NODE_ENV || 'development';
const config = require(__dirname + '/../config/config.js')[env];
const db = {};

let sequelize;
if (config.use_env_variable) {
  sequelize = new Sequelize(process.env[config.use_env_variable], config);
} else {
  sequelize = new Sequelize(config.database, config.username, config.password, config);
}

fs
  .readdirSync(__dirname)
  .filter(file => {
    return (file.indexOf('.') !== 0) && (file !== basename) && (file.slice(-3) === '.js');
  })
  .forEach(file => {
    const model = require(path.join(__dirname, file))(sequelize, Sequelize.DataTypes);
    db[model.name] = model;
  });

Object.keys(db).forEach(modelName => {
  if (db[modelName].associate) {
    db[modelName].associate(db);
  }
});

db.sequelize = sequelize;
db.Sequelize = Sequelize;

module.exports = db;
```

## backend/models/user.js

```js
'use strict';
const { Model } = require('sequelize');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

module.exports = (sequelize, DataTypes) => {
  class User extends Model {
    static associate(models) {
      // define associations here
    }

    async matchPassword(enteredPassword) {
      return await bcrypt.compare(enteredPassword, this.password);
    }

    getSignedJwtToken() {
      return jwt.sign({ id: this.id }, process.env.JWT_SECRET, {
        expiresIn: '30d',
      });
    }
  }
  
  User.init({
    name: { type: DataTypes.STRING, allowNull: false },
    email: { type: DataTypes.STRING, allowNull: false, unique: true },
    password: { type: DataTypes.STRING, allowNull: false },
    phone: DataTypes.STRING,
    role: { type: DataTypes.STRING, defaultValue: 'user' }
  }, {
    sequelize,
    modelName: 'User',
    hooks: {
      beforeSave: async (user) => {
        if (user.changed('password')) {
          const salt = await bcrypt.genSalt(10);
          user.password = await bcrypt.hash(user.password, salt);
        }
      }
    }
  });
  
  return User;
};
```

## backend/models/product.js

```js
'use strict';
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class Product extends Model {
    static associate(models) {
      // define associations here
    }
  }
  
  Product.init({
    name: { type: DataTypes.STRING, allowNull: false },
    description: DataTypes.TEXT,
    price: { type: DataTypes.DECIMAL(10, 2), allowNull: false },
    stock: { type: DataTypes.INTEGER, allowNull: false, defaultValue: 0 },
    category: DataTypes.STRING,
    imageUrl: DataTypes.STRING,
    sku: DataTypes.STRING,
    isActive: { type: DataTypes.BOOLEAN, defaultValue: true }
  }, {
    sequelize,
    modelName: 'Product',
  });
  
  return Product;
};
```

## backend/models/order.js

```js
'use strict';
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class Order extends Model {
    static associate(models) {
      Order.belongsTo(models.User, { foreignKey: 'userId' });
    }
  }
  
  Order.init({
    orderId: { type: DataTypes.STRING, unique: true, allowNull: false },
    userId: DataTypes.INTEGER,
    status: { type: DataTypes.STRING, defaultValue: 'Pending' },
    totalAmount: { type: DataTypes.DECIMAL(10, 2), allowNull: false },
    customerName: DataTypes.STRING,
    customerEmail: DataTypes.STRING,
    customerPhone: DataTypes.STRING,
    shippingAddress: DataTypes.TEXT,
    paymentStatus: { type: DataTypes.STRING, defaultValue: 'Pending' },
    paymentReference: DataTypes.STRING,
    items: DataTypes.JSON // Store order items as JSON
  }, {
    sequelize,
    modelName: 'Order',
  });
  
  return Order;
};
```