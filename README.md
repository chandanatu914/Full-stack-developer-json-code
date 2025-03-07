const express = require('express');
const mongoose = require('mongoose');
const axios = require('axios');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

// MongoDB connection
mongoose.connect('mongodb://localhost:27017/mern-challenge')
    .then(() => console.log('MongoDB connected'))
    .catch(err => console.error('MongoDB Error:', err));

// Transaction Schema
const transactionSchema = new mongoose.Schema({
    title: String,
    description: String,
    price: Number,
    category: String,
    dateOfSale: Date,
    sold: Boolean
});
const Transaction = mongoose.model('Transaction', transactionSchema);

// Initialize Database with Seed Data
app.get('/api/init', async (req, res) => {
    try {
        const { data } = await axios.get('https://s3.amazonaws.com/roxiler.com/product_transaction.json');
        await Transaction.deleteMany();
        await Transaction.insertMany(data);
        res.json({ message: 'Database initialized successfully' });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Get Paginated Transactions with Search
app.get('/api/transactions', async (req, res) => {
    try {
        const { page = 1, perPage = 10, search = '', month } = req.query;
        const monthNumber = new Date(`${month} 1, 2023`).getMonth() + 1;

        const query = {
            dateOfSale: { $gte: new Date(`2023-${monthNumber}-01`), $lt: new Date(`2023-${monthNumber + 1}-01`) }
        };

        if (search) {
            query.$or = [
                { title: { $regex: search, $options: 'i' } },
                { description: { $regex: search, $options: 'i' } },
                { price: isNaN(search) ? undefined : Number(search) }
            ];
        }

        const transactions = await Transaction.find(query)
            .skip((page - 1) * perPage)
            .limit(Number(perPage));
        res.json(transactions);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Get Statistics for Selected Month
app.get('/api/statistics', async (req, res) => {
    try {
        const { month } = req.query;
        const monthNumber = new Date(`${month} 1, 2023`).getMonth() + 1;

        const query = {
            dateOfSale: { $gte: new Date(`2023-${monthNumber}-01`), $lt: new Date(`2023-${monthNumber + 1}-01`) }
        };

        const [totalSale, notSoldItems] = await Promise.all([
            Transaction.aggregate([
                { $match: query },
                {
                    $group: {
                        _id: null,
                        totalAmount: { $sum: '$price' },
                        soldItems: { $sum: { $cond: ['$sold', 1, 0] } }
                    }
                }
            ]),
            Transaction.countDocuments({ ...query, sold: false })
        ]);

        res.json({
            totalAmount: totalSale[0]?.totalAmount || 0,
            soldItems: totalSale[0]?.soldItems || 0,
            notSoldItems
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Get Bar Chart Data
app.get('/api/barchart', async (req, res) => {
    try {
        const { month } = req.query;
        const monthNumber = new Date(`${month} 1, 2023`).getMonth() + 1;

        const query = {
            dateOfSale: { $gte: new Date(`2023-${monthNumber}-01`), $lt: new Date(`2023-${monthNumber + 1}-01`) }
        };

        const priceRanges = [
            { range: '0-100', min: 0, max: 100 },
            { range: '101-200', min: 101, max: 200 },
            { range: '201-300', min: 201, max: 300 },
            { range: '301-400', min: 301, max: 400 },
            { range: '401-500', min: 401, max: 500 },
            { range: '501-600', min: 501, max: 600 },
            { range: '601-700', min: 601, max: 700 },
            { range: '701-800', min: 701, max: 800 },
            { range: '801-900', min: 801, max: 900 },
            { range: '901-above', min: 901, max: Infinity }
        ];

        const result = await Promise.all(priceRanges.map(async ({ range, min, max }) => {
            const count = await Transaction.countDocuments({ ...query, price: { $gte: min, $lte: max } });
            return { range, count };
        }));

        res.json(result);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Get Pie Chart Data
app.get('/api/piechart', async (req, res) => {
    try {
        const { month } = req.query;
        const monthNumber = new Date(`${month} 1, 2023`).getMonth() + 1;

        const query = {
            dateOfSale: { $gte: new Date(`2023-${monthNumber}-01`), $lt: new Date(`2023-${monthNumber + 1}-01`) }
        };

        const categoryData = await Transaction.aggregate([
            { $match: query },
            { $group: { _id: '$category', itemCount: { $sum: 1 } } }
        ]);

        res.json(categoryData);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Combined API: Statistics + Bar Chart + Pie Chart
app.get('/api/combined', async (req, res) => {
    try {
        const { month } = req.query;

        const [statistics, barChart, pieChart] = await Promise.all([
            axios.get(`http://localhost:5000/api/statistics?month=${month}`),
            axios.get(`http://localhost:5000/api/barchart?month=${month}`),
            axios.get(`http://localhost:5000/api/piechart?month=${month}`)
        ]);

        res.json({
            statistics: statistics.data,
            barChart: barChart.data,
            pieChart: pieChart.data
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Start Server
const PORT = 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
