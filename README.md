const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');

const authRoutes = require('./routes/auth');
const inventoryRoutes = require('./routes/inventory');
const orderRoutes = require('./routes/order');

const app = express();
app.use(cors());
app.use(bodyParser.json());

app.use('/auth', authRoutes);
app.use('/inventory', inventoryRoutes);
app.use('/order', orderRoutes);

app.listen(5000, () => {
    console.log('Backend running on http://localhost:5000');
});
const express = require('express');
const router = express.Router();
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./db.sqlite');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

const SECRET = 'secret123';

// 用户表
db.run(`CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE,
    password TEXT,
    role TEXT
)`);

// 注册
router.post('/register', async (req, res) => {
    const { username, password, role } = req.body;
    const hashed = await bcrypt.hash(password, 10);
    db.run(`INSERT INTO users (username, password, role) VALUES (?, ?, ?)`, [username, hashed, role], function(err) {
        if (err) return res.status(400).json({ error: err.message });
        res.json({ message: '注册成功', userId: this.lastID });
    });
});

// 登录
router.post('/login', (req, res) => {
    const { username, password } = req.body;
    db.get(`SELECT * FROM users WHERE username=?`, [username], async (err, user) => {
        if (!user) return res.status(400).json({ error: '用户不存在' });
        const match = await bcrypt.compare(password, user.password);
        if (!match) return res.status(400).json({ error: '密码错误' });
        const token = jwt.sign({ id: user.id, role: user.role }, SECRET);
        res.json({ token, role: user.role });
    });
});

module.exports = router;
const express = require('express');
const router = express.Router();
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./db.sqlite');

// 库存表
db.run(`CREATE TABLE IF NOT EXISTS inventory (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    quantity INTEGER,
    price REAL,
    sellerId INTEGER
)`);

// 发布库存
router.post('/add', (req, res) => {
    const { name, quantity, price, sellerId } = req.body;
    db.run(`INSERT INTO inventory (name, quantity, price, sellerId) VALUES (?, ?, ?, ?)`, [name, quantity, price, sellerId], function(err) {
        if (err) return res.status(400).json({ error: err.message });
        res.json({ message: '库存发布成功', id: this.lastID });
    });
});

// 获取所有库存
router.get('/', (req, res) => {
    db.all(`SELECT * FROM inventory`, [], (err, rows) => {
        if (err) return res.status(400).json({ error: err.message });
        res.json(rows);
    });
});

module.exports = router;
const express = require('express');
const router = express.Router();
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./db.sqlite');

// 订单表
db.run(`CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    buyerId INTEGER,
    inventoryId INTEGER,
    quantity INTEGER,
    status TEXT
)`);

// 下单
router.post('/create', (req, res) => {
    const { buyerId, inventoryId, quantity } = req.body;
    // 假支付流程
    db.run(`INSERT INTO orders (buyerId, inventoryId, quantity, status) VALUES (?, ?, ?, ?)`, [buyerId, inventoryId, quantity, 'paid'], function(err) {
        if (err) return res.status(400).json({ error: err.message });
        res.json({ message: '支付成功，订单创建完成', orderId: this.lastID });
    });
});

// 查看订单
router.get('/:buyerId', (req, res) => {
    const { buyerId } = req.params;
    db.all(`SELECT * FROM orders WHERE buyerId=?`, [buyerId], (err, rows) => {
        if (err) return res.status(400).json({ error: err.message });
        res.json(rows);
    });
});

module.exports = router;
import { useState } from 'react';
import Login from './pages/Login';
import Register from './pages/Register';
import SellerDashboard from './pages/SellerDashboard';
import BuyerDashboard from './pages/BuyerDashboard';

export default function App() {
  const [token, setToken] = useState(null);
  const [role, setRole] = useState(null);

  if (!token) {
    return (
      <div>
        <Login setToken={(t) => setToken(t)} setRole={setRole} />
        <Register />
      </div>
    )
  }

  if (role === 'seller') return <SellerDashboard token={token} />
  return <BuyerDashboard token={token} />
}
