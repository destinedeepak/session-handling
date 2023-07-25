# Session Handling in Node.js with Express, MongoDB

## Introduction

Session handling is a critical aspect of web applications as it allows for the persistence of user-specific data across multiple HTTP requests. In this guide, we'll explore how to handle sessions in a Node.js and Express.js application using MongoDB as the session store. Additionally, we'll implement role-based authorization to differentiate between "admin" and "user" access.

## Prerequisites

1. Node.js and npm (Node Package Manager) installed on your machine.

2. MongoDB installed and running on your system or use a cloud-based MongoDB service like MongoDB Atlas.

3. Install the required dependencies (Express.js, Mongoose, and `connect-mongodb-session`):


## MongoDB Connection Setup

```javascript
// db.js

const mongoose = require('mongoose');

// Replace 'mongodb://localhost/session-handling' with your MongoDB connection string
const connectionString = 'mongodb://localhost/session-handling';

mongoose.connect(connectionString, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const db = mongoose.connection;

db.on('error', console.error.bind(console, 'MongoDB connection error:'));
db.once('open', () => {
  console.log('Connected to MongoDB!');
});

module.exports = db;
```

## User Schema and Model
```javascript
// user.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
});

const User = mongoose.model('User', userSchema);

module.exports = User;
```

## Configuring Sessions and Authorization
```javascript
//Set up the Express.js application, configure sessions, and handle authorization:
//Just for better understanding, putting all configuration together in same file.
// app.js

const express = require('express');
const session = require('express-session');
const MongoDBStore = require('connect-mongodb-session')(session);

const db = require('./db');
const User = require('./user');

const app = express();

// Configure the MongoDB session store
const store = new MongoDBStore({
  uri: 'mongodb://localhost/session-handling', // Replace with your MongoDB connection string
  collection: 'sessions',
});

// Catch session store errors
store.on('error', function (error) {
  console.error('Session store error:', error);
});

// Configure the session middleware
app.use(
  session({
    secret: 'your-secret-key', // Replace with a secure and unique secret key
    resave: false,
    saveUninitialized: false,
    store: store,
    cookie: {
      maxAge: 1000 * 60 * 60 * 24, // Session will expire after 1 day
    },
  })
);

// Middleware function to check if the user is an admin
const isAdmin = (req, res, next) => {
  if (req.session.userId) {
    User.findById(req.session.userId)
      .then(user => {
        if (user && user.role === 'admin') {
          next(); // User is an admin, proceed to the next middleware or route handler
        } else {
          res.status(403).json({ error: 'Unauthorized' }); // User is not an admin, access denied
        }
      })
      .catch(err => {
        res.status(500).json({ error: 'Server error' });
      });
  } else {
    res.status(401).json({ error: 'Unauthorized' }); // User is not logged in, access denied
  }
};

// Route for user registration
app.post('/register', async (req, res) => {
  const { username, password, role } = req.body;

  try {
    const user = await User.create({ username, password, role });
    res.status(201).json(user);
  } catch (err) {
    res.status(400).json({ error: 'Failed to create user' });
  }
});

// Route for user login
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  try {
    const user = await User.findOne({ username, password });
    if (user) {
      req.session.userId = user._id;
      res.status(200).json({ message: 'Login successful' });
    } else {
      res.status(401).json({ error: 'Invalid credentials' });
    }
  } catch (err) {
    res.status(500).json({ error: 'Server error' });
  }
});

// Route for viewing user profile (only accessible to authenticated users)
app.get('/profile', (req, res) => {
  if (req.session.userId) {
    User.findById(req.session.userId)
      .then(user => {
        if (user) {
          res.status(200).json(user);
        } else {
          res.status(404).json({ error: 'User not found' });
        }
      })
      .catch(err => {
        res.status(500).json({ error: 'Server error' });
      });
  } else {
    res.status(401).json({ error: 'Unauthorized' });
  }
});

// Route for accessing admin dashboard (only accessible to admins)
app.get('/admin', isAdmin, (req, res) => {
  res.status(200).json({ message: 'Welcome, admin!' });
});

// Start the server
const port = 3000;
app.listen(port, () => {
  console.log(`Server is running on http://localhost:${port}`);
});

```

## Setup and Options for Session Handling in Node.js with Express and MongoDB

**Express-session** is an Express.js middleware that enables session handling in web applications. It provides an easy way to manage user sessions and store session data between HTTP requests.

### Session Options explained

- `secret`: A required option that specifies the session signing secret. It should be a secure and unique key used to sign the session ID cookie to prevent tampering.

- `resave`: An optional boolean option that determines whether to save the session data to the session store even if the session was not modified during the request. Setting it to `false` can improve performance.

- `saveUninitialized`: An optional boolean option that determines whether to save an uninitialized session to the store. If set to `false`, the session will not be saved until it is modified.

- `store`: An optional session store where session data will be stored. In this case, we use **connect-mongodb-session** as the session store to store session data in MongoDB.

- `cookie`: An object containing options for the session cookie, such as `maxAge` to set the expiration time for the session.

### Connect-mongodb-session

**Connect-mongodb-session** is a MongoDB-based session store for Express.js applications. It allows the application to store session data in a MongoDB database.

### MongoDBStore

**MongoDBStore** is a class provided by **connect-mongodb-session** that is used to create a new MongoDB-based session store. It takes the MongoDB connection URI and collection name as options.

### Mongoose

**Mongoose** is an Object Data Modeling (ODM) library for MongoDB and Node.js. It is used here to connect to the MongoDB database and define the user schema and model.

### User Schema and Model

The **user schema** defines the structure of the user data, including fields like `username`, `password`, and `role`. The **user model** is a Mongoose model that interacts with the MongoDB database for CRUD operations on user data.

### isAdmin Middleware

**isAdmin** is a middleware function that checks if the user is an admin. It queries the database to check the user's role and grants access to admin-specific routes if the user is an admin.








