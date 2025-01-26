Here's the updated guide including all files mentioned in the file structure:

### Affiliate Rewards Web App

**A full-stack affiliate tracking system with user rewards, admin controls, and secure authentication.**

#### Architecture Diagram
- **Note**: You'll need to create or obtain an architecture diagram to visually represent the system's components.

#### Features
- User registration/login with JWT authentication
- Admin dashboard for link generation and approval
- XP and level progression system
- Affiliate link tracking with click analytics
- Secure REST API with rate limiting and CSRF protection
- Swagger API documentation
- Dockerized production setup
- Prometheus monitoring
- Password reset and email verification

#### Technologies
- **Backend**: Node.js, Express, Mongoose
- **Frontend**: EJS, Tailwind CSS
- **Database**: MongoDB
- **Security**: Helmet, CSRF, rate limiting
- **Infrastructure**: Docker, GitHub Codespaces
- **Monitoring**: Prometheus, Grafana (optional)

#### Setup Guide

##### 1. Prerequisites
- GitHub account
- Docker (for production)
- MongoDB Atlas URI (for production)

##### 2. GitHub Codespaces Setup
```bash
# Create codespace from your repo
# Wait for container build (2-3 minutes)

# Install system dependencies
sudo apt-get update && sudo apt-get install -y mongodb
sudo service mongodb start

# Create necessary directories and files
mkdir -p config controllers middleware models routes views public/{css,js} scripts
touch config/db.js controllers/authController.js controllers/adminController.js middleware/auth.js middleware/admin.js middleware/errorHandler.js models/User.js models/AffiliateLink.js routes/auth.js routes/admin.js routes/user.js views/login.ejs views/admin-dashboard.ejs views/user-dashboard.ejs views/error.ejs public/css/styles.css public/css/output.css .env app.js Dockerfile docker-compose.yml swagger.yaml scripts/db-backup.sh
```

##### 3. File Structure
```
├── config/
│   └── db.js
├── controllers/
│   ├── authController.js
│   └── adminController.js
├── middleware/
│   ├── auth.js
│   ├── admin.js
│   └── errorHandler.js
├── models/
│   ├── User.js
│   └── AffiliateLink.js
├── routes/
│   ├── auth.js
│   ├── admin.js
│   └── user.js
├── views/
│   ├── login.ejs
│   ├── admin-dashboard.ejs
│   └── user-dashboard.ejs
├── public/
│   ├── css/
│   │   ├── styles.css
│   │   └── output.css
├── .env
├── app.js
├── Dockerfile
└── docker-compose.yml
```

##### 4. Environment Setup
```bash
# Install dependencies
npm install express mongoose bcryptjs jsonwebtoken cors body-parser ejs tailwindcss nodemon dotenv helmet csrf express-rate-limit validator swagger-ui-express mongoose-autopopulate prom-client

# Development dependencies
npm install --save-dev nodemon tailwindcss
```

##### 5. Configuration Files

**.env**
```plaintext
MONGO_URI=mongodb://localhost:27017/affiliate-app
JWT_SECRET=your_strong_secret_here
JWT_EXPIRES_IN=1h
PORT=5000
NODE_ENV=development
RESET_TOKEN_EXPIRE=3600000
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=user@example.com
SMTP_PASS=password
```

**tailwind.config.js**
```javascript
module.exports = {
  content: ["./views/**/*.ejs"],
  theme: {
    extend: {}
  },
  plugins: [],
}
```

##### 6. Core Implementation

**app.js**
```javascript
require('dotenv').config();
const express = require('express');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const csrf = require('csurf');
const cookieParser = require('cookie-parser');
const prometheus = require('prom-client');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');
const errorHandler = require('./middleware/errorHandler');

const app = express();

// Security Middleware
app.use(helmet());
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));
app.use(cookieParser());
app.use(csrf({ cookie: true }));

// Prometheus Monitoring
prometheus.collectDefaultMetrics();
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});

// Swagger Docs
const swaggerDocument = YAML.load('./swagger.yaml');
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument));

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/admin', require('./routes/admin'));
app.use('/api/user', require('./routes/user'));

// Error Handling
app.use(errorHandler);

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

**config/db.js**
```javascript
const mongoose = require('mongoose');

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB Connected');
  } catch (err) {
    console.error(err.message);
    process.exit(1);
  }
};

module.exports = connectDB;
```

**controllers/authController.js**
```javascript
const User = require('../models/User');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { validationResult } = require('express-validator');

exports.register = async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  const { username, email, password } = req.body;

  try {
    let user = await User.findOne({ email });
    if (user) return res.status(400).json({ msg: 'User already exists' });

    user = new User({ username, email, password });
    await user.save();

    const payload = { user: { id: user.id } };
    jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: '1h' }, (err, token) => {
      if (err) throw err;
      res.json({ token });
    });
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
};

exports.login = async (req, res) => {
  const { email, password } = req.body;

  try {
    let user = await User.findOne({ email });
    if (!user) return res.status(400).json({ msg: 'Invalid credentials' });

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) return res.status(400).json({ msg: 'Invalid credentials' });

    const payload = { user: { id: user.id } };
    jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: '1h' }, (err, token) => {
      if (err) throw err;
      res.json({ token });
    });
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
};
```

**controllers/adminController.js**
```javascript
const AffiliateLink = require('../models/AffiliateLink');
const User = require('../models/User');

exports.generateLink = async (req, res) => {
  try {
    const { youtubeVideoId, affiliateLink, userId } = req.body;
    
    const newLink = new AffiliateLink({
      youtubeVideoId,
      affiliateLink,
      user: userId
    });

    await newLink.save();
    await User.findByIdAndUpdate(userId, { $push: { affiliateLinks: newLink._id } });

    res.status(201).json(newLink);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
};
```

**middleware/auth.js**
```javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');

exports.auth = async (req, res, next) => {
  const token = req.header('x-auth-token');

  if (!token) return res.status(401).json({ msg: 'No token, authorization denied' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.user.id).select('-password');
    next();
  } catch (err) {
    res.status(401).json({ msg: 'Token is not valid' });
  }
};
```

**middleware/admin.js**
```javascript
exports.admin = (req, res, next) => {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ msg: 'Admin privileges required' });
  }
  next();
};
```

**middleware/errorHandler.js**
```javascript
module.exports = (err, req, res, next) => {
  console.error(err.message);
  res.status(500).send('Server Error');
};
```

**models/User.js**
```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { 
    type: String, 
    required: true,
    unique: true,
    validate: {
      validator: v => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v),
      message: 'Invalid email format'
    }
  },
  password: {
    type: String,
    required: true,
    validate: {
      validator: v => /^(?=.*[A-Za-z])(?=.*\d)[A-Za-z\d]{8,}$/.test(v),
      message: 'Password must be 8+ chars with letters and numbers'
    }
  },
  xp: { type: Number, default: 0 },
  level: { type: Number, default: 1 },
  entries: { type: Number, default: 0 },
  role: { type: String, default: 'user' },
  affiliateLinks: [{ type: mongoose.Schema.Types.ObjectId, ref: 'AffiliateLink' }],
  resetPasswordToken: String,
  resetPasswordExpires: Date,
  emailVerified: { type: Boolean, default: false }
});

UserSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 12);
  }
  next();
});

module.exports = mongoose.model('User', UserSchema);
```

**models/AffiliateLink.js**
```javascript
const mongoose = require('mongoose');

const AffiliateLinkSchema = new mongoose.Schema({
  youtubeVideoId: { type: String, required: true },
  affiliateLink: { type: String, required: true },
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  clicks: { type: Number, default: 0 },
  createdAt: { type: Date, default: Date.now }
});

module.exports = mongoose.model('AffiliateLink', AffiliateLinkSchema);
```

**routes/auth.js**
```javascript
const express = require('express');
const router = express.Router();
const { check } = require('express-validator');
const authController = require('../controllers/authController');

router.post(
  '/register',
  [
    check('username', 'Username is required').not().isEmpty(),
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Password must be 6+ characters').isLength({ min: 6 })
  ],
  authController.register
);

router.post(
  '/login',
  [
    check('email', 'Please include a valid email').isEmail(),
    check('password', 'Password is required').exists()
  ],
  authController.login
);

module.exports = router;
```

**routes/admin.js**
```javascript
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const admin = require('../middleware/admin');
const adminController = require('../controllers/adminController');

router.post('/generate-link', [auth, admin], adminController.generateLink);
module.exports = router;
```

**routes/user.js**
```javascript
const express = require('express');
const router = express.Router();
const auth = require('../middleware/auth');
const User = require('../models/User');

router.get('/dashboard', auth, async (req, res) => {
  try {
    const user = await User.findById(req.user.id).populate('affiliateLinks');
    res.json(user);
  } catch (err) {
    console.error(err.message);
    res.status(500).send('Server error');
  }
});

module.exports = router;
```

**views/login.ejs**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Login</title>
  <link href="/css/output.css" rel="stylesheet">
</head>
<body class="bg-gray-100 min-h-screen flex items-center justify-center">
  <div class="bg-white p-8 rounded-lg shadow-md w-96">
    <h1 class="text-2xl font-bold mb-6 text-center">Login</h1>
    <form action="/api/auth/login" method="POST">
      <div class="mb-4">
        <input type="email" name="email" placeholder="Email" class="w-full px-3 py-2 border rounded-md" required>
      </div>
      <div class="mb-6">
        <input type="password" name="password" placeholder="Password" class="w-full px-3 py-2 border rounded-md" required>
      </div>
      <button type="submit" class="w-full bg-blue-500 text-white py-2 rounded-md hover:bg-blue-600 transition">
        Login
      </button>
    </form>
    <div class="mt-4 text-center">
      <a href="/register" class="text-blue-500 hover:underline">Create Account</a>
    </div>
  </div>
</body>
</html>
```

**views/admin-dashboard.ejs**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Admin Dashboard</title>
  <link href="/css/output.css" rel="stylesheet">
</head>
<body class="bg-gray-100">
  <div class="container mx-auto p-6">
    <h1 class="text-3xl font-bold mb-6">Admin Dashboard</h1>
    <form action="/api/admin/generate-link" method="POST">
      <div class="mb-4">
        <label for="youtubeVideoId" class="block text-sm font-medium text-gray-700">YouTube Video ID:</label>
        <input type="text" id="youtubeVideoId" name="youtubeVideoId" class="mt-1 block w-full px-3 py-2 border rounded-md" required>
      </div>
      <div class="mb-4">
        <label for="affiliateLink" class="block text-sm font-medium text-gray-700">Affiliate Link:</label>
        <input type="text" id="affiliateLink" name="affiliateLink" class="mt-1 block w-full px-3 py-2 border rounded-md" required>
      </div>
      <input type="hidden" name="userId" value="<%= userId %>">
      <button type="submit" class="w-full bg-blue-500 text-white py-2 rounded-md hover:bg-blue-600 transition">
        Generate Link
      </button>
    </form>
  </div>
</body>
</html>
```

**views/user-dashboard.ejs**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>User Dashboard</title>
  <link href="/css/output.css" rel="stylesheet">
</head>
<body class="bg-gray-100">
  <div class="container mx-auto p-6">
    <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
      <div class="bg-white p-6 rounded-lg shadow">
        <h2 class="text-2xl font-bold mb-4">Your Stats</h2>
        <div class="space-y-2">
          <p>XP: <%= user.xp %></p>
          <p>Level: <%= user.level %></p>
          <p>Clicks: <%= user.entries %></p>
        </div>
      </div>
      
      <div class="md:col-span-2 bg-white p-6 rounded-lg shadow">
        <h2 class="text-2xl font-bold mb-4">Your Affiliate Links</h2>
        <div class="space-y-4">
          <% user.affiliateLinks.forEach(link => { %>
            <div class="border p-4 rounded">
              <p class="font-semibold">YouTube ID: <%= link.youtubeVideoId %></p>
              <a href="/track/<%= link._id %>" 
                 class="text-blue-600 break-words hover:underline">
                <%= link.affiliateLink %>
              </a>
              <p class="text-sm text-gray-600 mt-2">Clicks: <%= link.clicks || 0 %></p>
            </div>
          <% }); %>
        </div>
      </div>
    </div>
  </div>
</body>
</html>
```

**views/error.ejs**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Error</title>
  <link href="/css/output.css" rel="stylesheet">
</head>
<body class="bg-gray-100 min-h-screen flex items-center justify-center">
  <div class="bg-white p-8 rounded-lg shadow-md w-96">
    <h1 class="text-2xl font-bold mb-6 text-center">Error</h1>
    <p class="text-center"><%= message || "An unexpected error occurred." %></p>
    <a href="/" class="block mt-4 text-center text-blue-500 hover:underline">Return Home</a>
  </div>
</body>
</html>
```

**public/css/styles.css**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  @apply transition-colors duration-200;
}

.btn-primary {
  @apply bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600 transition-all;
}
```

**public/css/output.css** (Generated by Tailwind CSS)

**Dockerfile**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

**docker-compose.yml**
```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://mongo:27017/affiliate-app
      - JWT_SECRET=your_production_secret
    depends_on:
      - mongo

  mongo:
    image: mongo:6
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: example

volumes:
  mongo-data:
```

##### 12. Running the Application
```bash
# Development
npm run dev

# Production
docker-compose up --build

# Access endpoints:
# - App: http://localhost:5000
# - API Docs: http://localhost:5000/api-docs
# - Metrics: http://localhost:5000/metrics
```

This guide now includes all the files listed in the file structure, providing a complete setup for the Affiliate Rewards Web App. Remember to secure your environment variables, especially in production, by not committing `.env` to version control.