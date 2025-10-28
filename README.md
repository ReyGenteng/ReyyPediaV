require('dotenv').config();
const express = require('express');
const session = require('express-session');
const nodemailer = require('nodemailer');
const MongoStore = require('connect-mongo');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const multer = require('multer');
const qrcode = require('qrcode');
const fs = require('fs');
const FormData = require('form-data');
const axios = require('axios');
const path = require('path');
const app = express();

const ATLAN_API_KEY = 'xbCfRqkmHgzyTUDmGnjYRiuKqJI3JNzkS1VQh0C1SDmpCndiSQtgTHuNUmHEoAB0sRrTc9ZhnS1FJwTcsTwNJmrFHMMebHvXYbzw';
const BASE_URL = 'https://atlantich2h.com';
const UNTUNG_PERSEN = 100;

mongoose.connect(process.env.MONGODB_URI || 'mongodb://localhost:27017/appdb');
const db = mongoose.connection;

const userSchema = new mongoose.Schema({
fullname: String,
username: { type: String, unique: true },
nomor: String,
email: { type: String, unique: true },
password: String,
profileUrl: String,
saldo: Number,
coin: Number,
apiKey: String,
tanggalDaftar: Date,
role: { type: String, default: 'user' },
isVerified: Boolean,
lastLogin: Date,
referralCode: String,
whitelistIp: {
type: [String],
},
history: [{
aktivitas: String,
nominal: Number,
status: String,
code: String,
tanggal: Date
}]
});

const User = mongoose.model('User', userSchema);

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));

app.use(session({
secret: process.env.SESSION_SECRET || 'your-secret-key',
resave: false,
saveUninitialized: false,
store: MongoStore.create({
mongoUrl: process.env.MONGODB_URI || 'mongodb://localhost:27017/appdb',
ttl: 14 * 24 * 60 * 60
}),
cookie: {
maxAge: 1000 * 60 * 60 * 24 * 14
}
}));

function requireLogin(req, res, next) {
if (!req.session.userId) {
return res.redirect('/auth/login');
}
next();
}

function requireAdmin(req, res, next) {
if (!req.session.userId || req.session.role !== 'admin') {
return res.redirect('/auth/login');
}
next();
}

app.get('/', (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'index.html'));
});
app.get('/auth/forgot-password', (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'forgot.html'));
});

app.get('/auth/register', (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'register.html'));
});

app.post('/auth/register', async (req, res) => {
try {
const existingUser = await User.findOne({ username: req.body.username });
if (existingUser) {
return res.status(400).json({ error: 'Username already exists' });
}


const existingEmail = await User.findOne({ email: req.body.email });
if (existingEmail) {
  return res.status(400).json({ error: 'Email already exists' });
}

let apiKey = generateApiKey();
let apiKeyExists = await User.findOne({ apiKey });

while (apiKeyExists) {
  apiKey = generateApiKey();
  apiKeyExists = await User.findOne({ apiKey });
}

const hashedPassword = await bcrypt.hash(req.body.password, 10);

const newUser = new User({
  fullname: req.body.fullname,
  username: req.body.username,
  nomor: req.body.nomor,
  email: req.body.email,
  password: hashedPassword,
  profileUrl: req.body.profileUrl || 'https://i.pinimg.com/236x/a2/80/e2/a280e2a50bf6240f29b49a72875adee5.jpg',
  saldo: 0,
  coin: 0,
  apiKey: apiKey,
  tanggalDaftar: new Date(),
  role: 'user',
  isVerified: false,
  lastLogin: new Date(),
  referralCode: generateReferralCode(req.body.username),
  history: []
});

await newUser.save();
res.redirect('/auth/login');
} catch (error) {
res.status(400).json({ error: 'Registration failed', message: error.message });
}
});

app.get('/auth/login', (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'login.html'));
});

app.post('/auth/login', async (req, res) => {
try {
const user = await User.findOne({
$or: [
{ username: req.body.usernameOrEmail },
{ email: req.body.usernameOrEmail }
]
});


  if (!user) {
    return res.status(400).json({
      success: false,
      message: 'User not found'
    });
  }

  const validPassword = await bcrypt.compare(req.body.password, user.password);
  if (!validPassword) {
    return res.status(400).json({
      success: false,
      message: 'Invalid password'
    });
  }

  req.session.userId = user._id;
  req.session.role = user.role;
  user.lastLogin = new Date();
  await user.save();

  return res.status(200).json({
    success: true,
    message: 'Login successful',
    role: user.role,
    redirectUrl: user.role === 'admin' ? '/admin' : '/dashboard'
  });

} catch (error) {
  console.error(error);
  return res.status(500).json({
    success: false,
    message: 'Internal server error'
  });
}
});

app.get('/dashboard', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'dashboard.html'));
});
app.get('/transfer', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'transfer.html'));
});

app.get('/profile', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'profile.html'));
});

app.get('/deposit', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'deposit.html'));
});

app.get('/price-list', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'price-list.html'));
});

app.get('/mutation', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'mutation.html'));
});

app.get('/topup', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'topup.html'));
});

app.get('/api-key-page', requireLogin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'api-key-page.html'));
});

app.get('/docs', (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'docs.html'));
});

app.get('/admin', requireAdmin, (req, res) => {
res.sendFile(path.join(__dirname, 'public', 'admin.html'));
});

app.get('/auth/logout', (req, res) => {
req.session.destroy();
res.redirect('/auth/login');
});

function generateApiKey() {
const randomPart = Math.random().toString(36).substring(2, 10) + Math.random().toString(36).substring(2, 10);
return V-Pedia-${randomPart};
}

function generateReferralCode(username) {
return username.toUpperCase() + Math.floor(1000 + Math.random() * 9000);
}

app.get('/profile/users', requireLogin, async (req, res) => {
try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({
success: false,
message: 'User not found'
});
}


return res.status(200).json({
  success: true,
  user: {
    fullname: user.fullname,
    username: user.username,
    nomor: user.nomor,
    email: user.email,
    profileUrl: user.profileUrl,
    saldo: user.saldo,
    coin: user.coin,
    apiKey: user.apiKey,
    tanggalDaftar: user.tanggalDaftar,
    role: user.role,
    isVerified: user.isVerified,
    lastLogin: user.lastLogin,
    referralCode: user.referralCode,
    history: user.history,
    whitelistIp: user.whitelistIp
  }
});
} catch (error) {
console.error(error);
return res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

const uploadMemory = multer({ storage: multer.memoryStorage() }); // ⏳ langsung memory

async function CatBox(buffer, originalname) {
const data = new FormData();
data.append('reqtype', 'fileupload');
data.append('userhash', '');
data.append('fileToUpload', buffer, { filename: originalname });

const config = {
method: 'POST',
url: 'https://catbox.moe/user/api.php',
headers: {
...data.getHeaders(),
'User-Agent': 'Mozilla/5.0 (Android 10; Mobile; rv:131.0) Gecko/131.0 Firefox/131.0',
},
data: data
};
const api = await axios.request(config);
return api.data;
}

app.post('/profile/update-photo', requireLogin, uploadMemory.single('photo'), async (req, res) => {
try {
const file = req.file;
if (!file) {
return res.status(400).json({ success: false, message: 'No file uploaded' });
}


const uploadedUrl = await CatBox(file.buffer, file.originalname);

await User.findByIdAndUpdate(req.session.userId, { profileUrl: uploadedUrl });

return res.status(200).json({
  success: true,
  message: 'Profile photo updated successfully',
  profileUrl: uploadedUrl
});
} catch (error) {
console.error('Error update profile photo:', error);
return res.status(500).json({
success: false,
message: 'Failed to update profile photo'
});
}
});

const otpStore = {};
const transporter = nodemailer.createTransport({
service: 'gmail',
auth: {
user: process.env.EMAIL_USER,
pass: process.env.EMAIL_PASS,
}
});

app.post('/auth/forgot-password', async (req, res) => {
const { email } = req.body;
if (!email) {
return res.status(400).json({ success: false, message: 'Email is required' });
}

const otp = Math.floor(100000 + Math.random() * 900000).toString();
const expires = Date.now() + 10 * 60 * 1000;

otpStore[email] = { otp, expires };

try {
await transporter.sendMail({
from: "Support" <${process.env.EMAIL_USER}>,
to: email,
subject: 'Your OTP for Password Reset',
html:       <div style="font-family: Arial, sans-serif; background-color: #f4f7fc; padding: 20px; border-radius: 8px; max-width: 600px; margin: auto; color: #333;">         <div style="background-color: #ffffff; padding: 30px; border-radius: 8px; box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);">           <h2 style="text-align: center; color: #4CAF50;">Password Reset Request</h2>           <p style="font-size: 16px; color: #555;">Hello,</p>           <p style="font-size: 16px; color: #555;">             We received a request to reset your password. Please use the OTP below to proceed with the reset process:           </p>           <div style="background-color: #f0f8ff; border: 2px solid #4CAF50; display: inline-block; padding: 10px 20px; font-size: 24px; font-weight: bold; margin: 20px 0; border-radius: 8px;">             ${otp}           </div>           <p style="font-size: 16px; color: #555;">             This OTP will expire in 10 minutes. Please use it quickly. If you did not request this change, please ignore this email.           </p>           <p style="font-size: 16px; color: #555;">Thank you,</p>           <p style="font-size: 16px; color: #555;">The Support Team</p>           <footer style="text-align: center; margin-top: 30px; font-size: 12px; color: #aaa;">             <p>This is an automated email. Please do not reply.</p>           </footer>         </div>       </div>    
});


return res.json({ success: true, message: 'OTP has been sent to your email' });
} catch (error) {
console.error('Error sending OTP:', error);
return res.status(500).json({ success: false, message: 'Failed to send OTP' });
}
});

app.post('/auth/reset-password', async (req, res) => {
const { email, otp, newPassword } = req.body;

if (!email || !otp || !newPassword) {
return res.status(400).json({ success: false, message: 'All fields are required' });
}

const otpData = otpStore[email];

if (!otpData) {
return res.status(400).json({ success: false, message: 'No OTP requested for this email' });
}

if (otpData.otp !== otp) {
return res.status(400).json({ success: false, message: 'Invalid OTP' });
}

if (otpData.expires < Date.now()) {
delete otpStore[email];
return res.status(400).json({ success: false, message: 'OTP expired' });
}

try {
const user = await User.findOne({ email });
if (!user) {
return res.status(400).json({ success: false, message: 'User not found' });
}


const hashedPassword = await bcrypt.hash(newPassword, 10);
user.password = hashedPassword;
await user.save();

delete otpStore[email];

return res.json({ success: true, message: 'Password has been reset successfully' });
} catch (error) {
console.error('Error resetting password:', error);
return res.status(500).json({ success: false, message: 'Server error' });
}
});

app.post('/profile/request-email-change', requireLogin, async (req, res) => {
const { newEmail } = req.body;

if (!newEmail) {
return res.status(400).json({ success: false, message: 'New email is required' });
}

const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}

if (newEmail === user.email) {
return res.status(400).json({ success: false, message: 'New email cannot be the same as current email' });
}

const emailUsed = await User.findOne({ email: newEmail });
if (emailUsed) {
return res.status(400).json({ success: false, message: 'Email already in use' });
}

const otp = Math.floor(100000 + Math.random() * 900000).toString();
const expires = Date.now() + 10 * 60 * 1000;

otpStore[change-email:${req.session.userId}] = { otp, newEmail, expires };

try {
await transporter.sendMail({
from: "Support" <${process.env.EMAIL_USER}>,
to: user.email,
subject: 'OTP untuk Perubahan Email',
html:         <h2>Permintaan Ubah Email</h2>         <p>Kami menerima permintaan untuk mengganti email akun Anda.</p>         <p>Gunakan OTP berikut untuk mengonfirmasi:</p>         <div style="font-size:24px; font-weight:bold;">${otp}</div>         <p>OTP ini berlaku selama 10 menit.</p>      
});


return res.json({ success: true, message: 'OTP sent to your current email address' });
} catch (err) {
console.error('Failed to send OTP:', err);
return res.status(500).json({ success: false, message: 'Failed to send OTP' });
}
});

app.post('/profile/confirm-email-change', requireLogin, async (req, res) => {
const { otp } = req.body;
const key = change-email:${req.session.userId};
const data = otpStore[key];

if (!otp || !data) {
return res.status(400).json({ success: false, message: 'No email change request found or expired' });
}

if (data.expires < Date.now()) {
delete otpStore[key];
return res.status(400).json({ success: false, message: 'OTP expired' });
}

if (otp !== data.otp) {
return res.status(400).json({ success: false, message: 'Invalid OTP' });
}

try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}


user.email = data.newEmail;
user.isVerified = false; 
await user.save();

delete otpStore[key];

return res.json({ success: true, message: 'Email updated successfully', newEmail: user.email });
} catch (err) {
console.error('Failed to update email:', err);
return res.status(500).json({ success: false, message: 'Server error' });
}
});

app.post('/profile/change-fullname', requireLogin, async (req, res) => {
const { newFullname } = req.body;

if (!newFullname) {
return res.status(400).json({ success: false, message: 'New fullname is required' });
}

const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}

try {
user.fullname = newFullname; // Mengubah fullname
await user.save();


return res.status(200).json({
  success: true,
  message: 'Fullname has been successfully changed',
  newFullname: user.fullname,
});
} catch (err) {
console.error('Error changing fullname:', err);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});

app.post('/profile/change-username', requireLogin, async (req, res) => {
const { newUsername } = req.body;

if (!newUsername) {
return res.status(400).json({ success: false, message: 'New username is required' });
}

const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}

const existingUsername = await User.findOne({ username: newUsername });
if (existingUsername) {
return res.status(400).json({ success: false, message: 'Username already exists' });
}

try {
user.username = newUsername;
await user.save();


return res.status(200).json({
  success: true,
  message: 'Username has been successfully changed',
  newUsername: user.username,
});
} catch (err) {
console.error('Error changing username:', err);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});

app.post('/profile/change-nomor', requireLogin, async (req, res) => {
const { newNomor } = req.body;

if (!newNomor) {
return res.status(400).json({ success: false, message: 'New nomor is required' });
}

const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}

const phoneRegex = /^[0-9]{10,15}$/;
if (!phoneRegex.test(newNomor)) {
return res.status(400).json({ success: false, message: 'Invalid phone number format' });
}

try {
user.nomor = newNomor;
await user.save();


return res.status(200).json({
  success: true,
  message: 'Nomor has been successfully changed',
  newNomor: user.nomor,
});
} catch (err) {
console.error('Error changing nomor:', err);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});

app.post('/profile/regenerate-apikey', requireLogin, async (req, res) => {
try {
let newApiKey = generateApiKey();


let apiKeyExists = await User.findOne({ apiKey: newApiKey });

while (apiKeyExists) {
  newApiKey = generateApiKey();
  apiKeyExists = await User.findOne({ apiKey: newApiKey });
}

const user = await User.findById(req.session.userId);
if (!user) {
  return res.status(404).json({ success: false, message: 'User not found' });
}

user.apiKey = newApiKey;
await user.save();

return res.status(200).json({
  success: true,
  message: 'API Key has been successfully regenerated',
  newApiKey: newApiKey,
});
} catch (err) {
console.error('Error regenerating API Key:', err);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});

app.post('/profile/change-password', requireLogin, async (req, res) => {
const { oldPassword, newPassword } = req.body;

if (!oldPassword || !newPassword) {
return res.status(400).json({ success: false, message: 'Old password and new password are required' });
}

try {


const user = await User.findById(req.session.userId);
if (!user) {
  return res.status(404).json({ success: false, message: 'User not found' });
}

const isOldPasswordValid = await bcrypt.compare(oldPassword, user.password);
if (!isOldPasswordValid) {
  return res.status(400).json({ success: false, message: 'Old password is incorrect' });
}

const hashedNewPassword = await bcrypt.hash(newPassword, 10);

user.password = hashedNewPassword;
await user.save();

return res.status(200).json({ success: true, message: 'Password has been successfully changed' });
} catch (err) {
console.error('Error changing password:', err);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});

app.get('/data/users', requireAdmin, async (req, res) => {
try {
const users = await User.find({}, '-password -__v');
res.json(users);
} catch (err) {
res.status(500).send('Server Error');
}
});

app.post('/admin/verify-user', requireAdmin, async (req, res) => {
const { username } = req.body;

if (!username) {
return res.status(400).json({
success: false,
message: 'Parameter username wajib diisi'
});
}

try {
const user = await User.findOne({ username });


if (!user) {
  return res.status(404).json({
    success: false,
    message: 'User tidak ditemukan'
  });
}

if (user.isVerified) {
  return res.status(400).json({
    success: false,
    message: 'User sudah terverifikasi sebelumnya'
  });
}

user.isVerified = true;
await user.save();

return res.status(200).json({
  success: true,
  message: `User ${username} berhasil diverifikasi`
});
} catch (err) {
console.error('❌ Error saat memverifikasi user:', err);
return res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

app.post('/profile/send-verification-otp', requireLogin, async (req, res) => {
try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}


if (user.isVerified) {
  return res.status(400).json({ success: false, message: 'Email already verified' });
}

const otp = Math.floor(100000 + Math.random() * 900000).toString();
const expires = Date.now() + 10 * 60 * 1000; // OTP berlaku selama 10 menit

otpStore[`verify-email:${user.email}`] = { otp, expires };

await transporter.sendMail({
  from: `"Support" <${process.env.EMAIL_USER}>`,
  to: user.email,
  subject: 'OTP for Email Verification',
  html: `
    <div style="font-family: Arial, sans-serif; background-color: #f4f7fc; padding: 20px; border-radius: 8px; max-width: 600px; margin: auto; color: #333;">
      <div style="background-color: #ffffff; padding: 30px; border-radius: 8px; box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);">
        <h2 style="text-align: center; color: #4CAF50;">Email Verification</h2>
        <p style="font-size: 16px; color: #555;">Hello,</p>
        <p style="font-size: 16px; color: #555;">
          Please use the OTP below to verify your email address:
        </p>
        <div style="background-color: #f0f8ff; border: 2px solid #4CAF50; display: inline-block; padding: 10px 20px; font-size: 24px; font-weight: bold; margin: 20px 0; border-radius: 8px;">
          ${otp}
        </div>
        <p style="font-size: 16px; color: #555;">
          This OTP will expire in 10 minutes. If you did not request this verification, please ignore this email.
        </p>
        <p style="font-size: 16px; color: #555;">Thank you,</p>
        <p style="font-size: 16px; color: #555;">The Support Team</p>
        <footer style="text-align: center; margin-top: 30px; font-size: 12px; color: #aaa;">
          <p>This is an automated email. Please do not reply.</p>
        </footer>
      </div>
    </div>
  `,
});

return res.json({ success: true, message: 'OTP sent to your email' });
} catch (error) {
console.error('Error sending verification OTP:', error);
return res.status(500).json({ success: false, message: 'Failed to send OTP' });
}
});

app.post('/profile/verify-email', requireLogin, async (req, res) => {
const { otp } = req.body;

try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}


if (user.isVerified) {
  return res.status(400).json({ success: false, message: 'Email already verified' });
}

const otpData = otpStore[`verify-email:${user.email}`];
if (!otpData) {
  return res.status(400).json({ success: false, message: 'No OTP found or expired' });
}

if (otpData.expires < Date.now()) {
  delete otpStore[`verify-email:${user.email}`]; 
  return res.status(400).json({ success: false, message: 'OTP expired' });
}

if (otp !== otpData.otp) {
  return res.status(400).json({ success: false, message: 'Invalid OTP' });
}

user.isVerified = true;
await user.save();

delete otpStore[`verify-email:${user.email}`];

return res.json({ success: true, message: 'Email successfully verified' });
} catch (error) {
console.error('Error verifying email:', error);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});
/*
app.post('/check-user', requireLogin, async (req, res) => {
const { username } = req.body;
if (!username) {
return res.status(400).json({ success: false, message: 'Username is required' });
}
try {
const user = await User.findOne({ username }, 'fullname username profileUrl');
if (!user) {
return res.status(404).json({ success: false, message: 'User not found' });
}
return res.status(200).json({
success: true,
user: {
fullname: user.fullname,
username: user.username,
profileUrl: user.profileUrl
}
});
} catch (error) {
console.error('Error checking user:', error);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});

app.post('/transfer-saldo', requireLogin, async (req, res) => {
const { recipientUsername, amount } = req.body;
if (!recipientUsername || !amount || amount <= 0) {
return res.status(400).json({ success: false, message: 'Invalid recipient or amount' });
}
try {
const sender = await User.findById(req.session.userId);
if (!sender) {
return res.status(404).json({ success: false, message: 'Sender not found' });
}
if (sender.saldo < amount) {
sender.history.push({
aktivitas: 'Transfer',
nominal: amount,
status: 'Gagal - Insufficient balance',
code: 'TRX-FAILED',
tanggal: new Date()
});
await sender.save();
return res.status(400).json({ success: false, message: 'Insufficient balance' });
}
const recipient = await User.findOne({ username: recipientUsername });
if (!recipient) {
sender.history.push({
aktivitas: 'Transfer',
nominal: amount,
status: 'Gagal - Recipient not found',
code: 'TRX-FAILED',
tanggal: new Date()
});
await sender.save();
return res.status(404).json({ success: false, message: 'Recipient not found' });
}


const refId = 'TRX-' + Math.random().toString(36).substring(2, 10).toUpperCase();
const transactionDate = new Date();

sender.saldo -= amount;
recipient.saldo += amount;

sender.history.push({
  aktivitas: `Transfer - From ${sender.username}`,
  nominal: amount,
  status: `Sukses`,
  code: refId,
  tanggal: transactionDate
});

recipient.history.push({
  aktivitas: `Receive  - From ${sender.username}`,
  nominal: amount,
  status: `Sukses`,
  code: refId,
  tanggal: transactionDate
});

await sender.save();
await recipient.save();

return res.status(200).json({
  success: true,
  message: 'Transfer successful',
  newBalance: sender.saldo,
  referenceId: refId,
  transactionDate: transactionDate
});
} catch (error) {
console.error('Error transferring saldo:', error);
return res.status(500).json({ success: false, message: 'Internal server error' });
}
});*/
app.get('/history/all', requireLogin, async (req, res) => {
try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({
success: false,
message: 'User not found'
});
}


return res.status(200).json({
  success: true,
  history: user.history
});
} catch (error) {
console.error('Error fetching history:', error);
return res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.get('/h2h/profile', validateApiKey, async (req, res) => {
try {
const user = req.user;


if (!user) {
  return res.status(404).json({
    success: false,
    message: 'User not found'
  });
}

const userData = {
  fullname: user.fullname,
  username: user.username,
  nomor: user.nomor,
  email: user.email,
  profileUrl: user.profileUrl,
  saldo: user.saldo,
  coin: user.coin,
  apiKey: user.apiKey,
  tanggalDaftar: user.tanggalDaftar,
  role: user.role,
  isVerified: user.isVerified,
  lastLogin: user.lastLogin,
  referralCode: user.referralCode,
  history: user.history
};

res.json({
  success: true,
  message: 'Profile data retrieved successfully',
  data: userData
});
} catch (error) {
console.error('Error fetching profile:', error);
res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.post('/deposit/create', requireLogin, async (req, res) => {
try {
const { nominal } = req.body;
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({
success: false,
message: 'User not found'
});
}
if (!nominal) {
return res.status(400).json({
success: false,
message: 'Parameter nominal is required'
});
}
const reff_id = generateReffId();
const url = ${BASE_URL}/deposit/create;
const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("reff_id", reff_id);
params.append("nominal", nominal);
params.append("type", "ewallet");
params.append("metode", "qrisfast");


const depositResponse = await axios.post(url, params.toString(), {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
  }
});

const depositData = depositResponse.data?.data;
if (!depositData?.id) {
  return res.status(400).json({
    success: false,
    message: 'Failed to create deposit request'
  });
}


res.json({
  success: true,
  message: 'Deposit request created',
  data: {
    ...depositData,
    qr_image: depositData.qr_image
  }
});

const checkDeposit = async () => {
  const statusUrl = `${BASE_URL}/deposit/status`;
  const statusParams = new URLSearchParams();
  statusParams.append("api_key", ATLAN_API_KEY);
  statusParams.append("id", depositData.id);

  try {
    const statusResponse = await axios.post(statusUrl, statusParams.toString(), {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
      }
    });

    const statusData = statusResponse.data?.data;
    const status = statusData?.status;

    if (status === "success") {
      const saldoMasuk = parseInt(statusData.get_balance || 0);
      user.saldo += saldoMasuk;
      user.history.push({
        aktivitas: 'Deposit',
        nominal: saldoMasuk,
        status: 'Sukses',
        code: depositData.id,
        tanggal: new Date()
      });
      await user.save();
      console.log(`✅ Deposit successful: ${saldoMasuk} for ${user.username}`);
      return;
    }

    if (status === "failed") {
      user.history.push({
        aktivitas: 'Deposit',
        nominal: parseInt(nominal),
        status: 'Gagal',
        code: depositData.id,
        tanggal: new Date()
      });
      await user.save();
      console.log(`❌ Deposit failed for ${user.username}`);
      return;
    }

    setTimeout(checkDeposit, 5000);
  } catch (error) {
    console.error('❌ Error checking deposit status:', error);
  }
};

checkDeposit();
} catch (error) {
console.error('❌ Error:', error);
res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.post('/deposit/status', requireLogin, async (req, res) => {
try {
const { trxid } = req.body;
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({
success: false,
message: 'User not found'
});
}
if (!trxid) {
return res.status(400).json({
success: false,
message: 'Parameter trxid is required'
});
}


const url = `${BASE_URL}/deposit/status`;
const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("id", trxid);

const statusResponse = await axios.post(url, params.toString(), {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
  }
});

const statusData = statusResponse.data?.data;
if (!statusData) {
  return res.status(404).json({
    success: false,
    message: 'Data not found from external server'
  });
}

res.json({
  success: true,
  message: 'Deposit status found',
  data: {
    trxid: statusData.reff_id,
    status: statusData.status,
    saldo_masuk: statusData.get_balance || 0
  }
});
} catch (error) {
console.error('❌ Error checking deposit status:', error);
res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.post('/deposit/cancel', requireLogin, async (req, res) => {
try {
const { trxid } = req.body;
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({
success: false,
message: 'User not found'
});
}
if (!trxid) {
return res.status(400).json({
success: false,
message: 'Parameter trxid is required'
});
}


const url = `${BASE_URL}/deposit/cancel`;
const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("id", trxid);

const cancelResponse = await axios.post(url, params.toString(), {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
  }
});

const cancelData = cancelResponse.data?.data;
if (!cancelData) {
  return res.status(404).json({
    success: false,
    message: 'Data not found from external server'
  });
}

res.json({
  success: true,
  message: 'Deposit successfully canceled',
  data: {
    trxid: cancelData.id,
    status: cancelData.status,
    created_at: cancelData.created_at
  }
});
} catch (error) {
console.error('❌ Error canceling deposit:', error);
res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.post('/api/categori', requireLogin, async (req, res) => {
try {
const url = ${BASE_URL}/layanan/price_list;


const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': url,
  },
  body: params
});

const data = await response.json();

if (!data.status) {
  return res.status(500).json({
    success: false,
    message: 'Server maintenance',
    maintenance: true,
    ip_message: data.message.replace(/[^0-9.]+/g, '')
  });
}

const categories = [...new Set(data.data.map(item => item.category))];

res.json({
  success: true,
  data: categories,
  message: 'Data kategori berhasil didapatkan'
});
} catch (error) {
console.error('Error:', error);
res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.post('/api/providers', requireLogin, async (req, res) => {
try {
const url = ${BASE_URL}/layanan/price_list;


const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': url,
  },
  body: params
});

const data = await response.json();

if (!data.status) {
  return res.status(500).json({
    success: false,
    message: 'Server maintenance',
    maintenance: true,
    ip_message: data.message.replace(/[^0-9.]+/g, '')
  });
}

const providers = [...new Set(data.data.map(item => item.provider))];

res.json({
  success: true,
  data: providers,
  message: 'Data provider berhasil didapatkan'
});
} catch (error) {
console.error('Error:', error);
res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

app.post('/api/price-list', requireLogin, async (req, res) => {
try {
const { category, provider } = req.body;


const url = `${BASE_URL}/layanan/price_list`;

const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': url,
  },
  body: params
});

const result = await response.json();

if (!result.status) {
  throw new Error('Server maintenance');
}

let data = result.data || [];

data.sort(regeXcomp);

if (category) {
  data = data.filter(i => 
    i.category && i.category.toLowerCase() === category.toLowerCase()
  );
}

if (provider) {
  data = data.filter(i =>
    i.provider && i.provider.toLowerCase() === provider.toLowerCase()
  );
}

const formattedData = data.map(i => {
  const finalPrice = calculateFinalPrice(i.price);
  return {
    code: i.code,
    name: i.name,
    category: i.category,
    type: i.type,
    provider: i.provider,
    brand_status: i.brand_status,
    status: i.status,
    img_url: customImageUrl,
    final_price: finalPrice,
    price_formatted: `Rp ${toRupiah(finalPrice)}`,
    status_emoji: i.status === "available" ? "✅" : "❎"
  };
});

res.json({
  success: true,
  data: formattedData,
  message: 'Data produk berhasil didapatkan'
});
} catch (error) {
console.error('Error:', error);
res.status(500).json({
success: false,
message: error.message || 'Terjadi kesalahan pada server'
});
}
});

app.post("/order/create", requireLogin, async (req, res) => {
try {
const { code, target } = req.body; // Ubah dari query ke body
const user = await User.findById(req.session.userId); // Dapatkan user dari session


if (!code) {
  return res.status(400).json({
    success: false,
    message: "Code produk harus diisi",
  });
}

if (!target) {
  return res.status(400).json({
    success: false,
    message: "Target harus diisi",
  });
}

const priceListUrl = `${BASE_URL}/layanan/price_list`;
const priceParams = new URLSearchParams();
priceParams.append("api_key", ATLAN_API_KEY);
priceParams.append("type", "prabayar");

const priceResponse = await axios.post(priceListUrl, priceParams, {
  headers: {
    "Content-Type": "application/x-www-form-urlencoded",
    "User-Agent":
      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
  },
});

if (!priceResponse.data.status) {
  return res.status(500).json({
    success: false,
    message: "Server maintenance",
    maintenance: true,
    ip_message: priceResponse.data.message.replace(/[^0-9.]+/g, ""),
  });
}

const product = priceResponse.data.data.find((item) => item.code === code);
if (!product) {
  return res.status(404).json({
    success: false,
    message: "Produk tidak ditemukan",
  });
}

const productPrice = parseInt(product.price);
const hargaTotal = Math.round(productPrice * 1.02);

if (user.saldo < hargaTotal) {
  user.history.push({
    aktivitas: "Order",
    nominal: hargaTotal,
    status: "Gagal - Saldo tidak mencukupi",
    code: generateReffId(),
    tanggal: new Date(),
  });
  await user.save();
  return res.status(400).json({
    success: false,
    message: "Saldo tidak mencukupi",
  });
}

const reff_id = generateReffId();
const createTransactionUrl = `${BASE_URL}/transaksi/create`;
const transactionParams = new URLSearchParams();
transactionParams.append("api_key", ATLAN_API_KEY);
transactionParams.append("code", code);
transactionParams.append("reff_id", reff_id);
transactionParams.append("target", target);

const transactionResponse = await axios.post(
  createTransactionUrl,
  transactionParams,
  {
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
      "User-Agent":
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    },
  }
);

const trx = transactionResponse.data?.data;
if (!trx?.id || !trx?.price) {
  user.history.push({
    aktivitas: "Order",
    nominal: hargaTotal,
    status: "Gagal - Data transaksi tidak lengkap",
    code: reff_id,
    tanggal: new Date(),
  });
  await user.save();
  return res.status(500).json({
    success: false,
    message: "Gagal membuat transaksi: ID atau harga kosong",
  });
}

res.json({
  success: true,
  message: "Transaksi berhasil dibuat",
  data: {
    ...trx,
    price: hargaTotal,
  },
});

const checkStatus = async () => {
  const statusUrl = `${BASE_URL}/transaksi/status`;
  const statusParams = new URLSearchParams();
  statusParams.append("api_key", ATLAN_API_KEY);
  statusParams.append("id", trx.id);
  statusParams.append("type", "prabayar");

  const statusResponse = await axios.post(statusUrl, statusParams, {
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
      "User-Agent":
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    },
  });

  const status = statusResponse.data.data?.status;

  if (status === "success") {
    user.saldo -= hargaTotal;
    if (user.role === "agen") {
      user.coin += Math.floor(hargaTotal * 0.01);
    }
    user.history.push({
      aktivitas: `Order - ${code} - ${target}`,
      nominal: hargaTotal,
      status: "Sukses",
      code: trx.id,
      tanggal: new Date(),
    });
    await user.save();
    console.log(`✅ Transaksi ${trx.id} sukses dan saldo dipotong`);
    return;
  }

  if (status === "failed") {
    user.history.push({
      aktivitas: `Order - ${code} - ${target}`,
      nominal: hargaTotal,
      status: "Gagal",
      code: trx.id,
      tanggal: new Date(),
    });
    await user.save();
    console.log(`❌ Transaksi ${trx.id} gagal`);
    return;
  }

  setTimeout(checkStatus, 5000);
};

checkStatus();
} catch (error) {
console.error("❌ Error:", error.response?.data || error.message);
res.status(500).json({
success: false,
message: "Terjadi kesalahan pada server",
});
}
});

app.get("/order/check", requireLogin, async (req, res) => {
try {
const { trxid } = req.query;
const user = await User.findById(req.session.userId);


if (!trxid) {
  return res.status(400).json({
    success: false,
    message: 'Parameter "trxid" harus diisi',
  });
}

const statusUrl = `${BASE_URL}/transaksi/status`;
const statusParams = new URLSearchParams();
statusParams.append("api_key", ATLAN_API_KEY);
statusParams.append("id", trxid);
statusParams.append("type", "prabayar");

const response = await axios.post(statusUrl, statusParams, {
  headers: {
    "Content-Type": "application/x-www-form-urlencoded",
    "User-Agent":
      "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
  },
});

const status = response.data?.data?.status;

if (!status) {
  return res.status(404).json({
    success: false,
    message: "Transaksi tidak ditemukan atau gagal mendapatkan status",
  });
}

let statusMessage;
if (status === "success") {
  statusMessage = "Transaksi berhasil";
} else if (status === "pending") {
  statusMessage = "Transaksi sedang diproses";
} else if (status === "failed") {
  statusMessage = "Transaksi gagal";
} else {
  statusMessage = "Status transaksi tidak diketahui";
}

res.json({
  success: true,
  message: statusMessage,
  data: {
    trxid: trxid,
    status: status,
  },
});
} catch (error) {
console.error("❌ Error:", error.response?.data || error.message);
res.status(500).json({
success: false,
message: "Terjadi kesalahan pada server",
});
}
});

//=====[ PRODUCT ENDPOINT ]=====//

async function getServerPublicIp() {
try {
const response = await axios.get('https://api.ipify.org?format=json');
return response.data.ip;
} catch (error) {
return null;
}
}

function validateApiKey(req, res, next) {
const apiKey = req.query.apikey;

if (!apiKey) {
return res.status(400).json({
success: false,
message: 'Parameter "apikey" harus diisi',
});
}

User.findOne({ apiKey })
.then(async (user) => {
if (!user) {
return res.status(401).json({
success: false,
message: "Invalid API Key",
});
}


  // Mendapatkan IP klien dari berbagai sumber
  const requestIp =
    req.headers['x-forwarded-for']?.split(',').shift().trim() || // Proxy headers
    req.ip || // Express built-in IP detection
    req.connection.remoteAddress; // Fallback

  // Jika IP klien adalah localhost (::1 atau 127.0.0.1), gunakan IP publik server
  const detectedIp = ['::1', '127.0.0.1'].includes(requestIp)
    ? await getServerPublicIp()
    : requestIp;

  // Pisahkan whitelistIp menjadi array jika berupa string dengan koma
  const whitelistIpArray = Array.isArray(user.whitelistIp)
    ? user.whitelistIp
    : user.whitelistIp.split(',').map(ip => ip.trim());

  // Gunakan IP yang terdeteksi untuk validasi whitelist
  if (whitelistIpArray.length > 0 && !whitelistIpArray.includes('0.0.0.0')) {
    if (!detectedIp || !whitelistIpArray.includes(detectedIp)) {
      console.warn(`Unauthorized API access from IP ${detectedIp} for user ${user.username}. IP not in whitelist: [${whitelistIpArray.join(', ')}]`);
      return res.status(403).json({
        success: false,
        message: "IP Anda saat ini tidak diizinkan untuk menggunakan API Key ini.",
        your_ip: detectedIp
      });
    }
  }

  if (user.isLocked) {
    return res.status(429).json({
      success: false,
      message: "Anda harus menunggu 5 detik sebelum melakukan request lagi.",
    });
  }

  user.isLocked = true;
  await user.save();

  setTimeout(async () => {
    try {
      if (!user.isVerified) {
        return res.status(403).json({
          success: false,
          message: "Akun belum terverifikasi. Silakan verifikasi akun terlebih dahulu.",
        });
      }

      req.user = user;
      next();
    } catch (error) {
      console.error("Error during API Key validation timeout:", error);
      return res.status(500).json({
        success: false,
        message: "Internal server error during validation",
      });
    } finally {
      user.isLocked = false;
      try {
        await user.save();
      } catch (saveError) {
        console.error("Error saving user after unlocking in validateApiKey:", saveError);
      }
    }
  }, 5000);
})
.catch((error) => {
  console.error("Error validating API Key (DB query):", error);
  return res.status(500).json({
    success: false,
    message: "Internal server error",
  });
});
}

const ipRegex = /^(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?).){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/;

function isValidIp(ip) {
return ip === '0.0.0.0' || ipRegex.test(ip);
}

app.get('/profile/whitelist-ip/add', requireLogin, async (req, res) => {
const { ip } = req.body;

if (!ip) {
return res.status(400).json({ success: false, message: 'Alamat IP diperlukan' });
}

if (!isValidIp(ip)) {
return res.status(400).json({ success: false, message: 'Format alamat IP tidak valid' });
}

try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'Pengguna tidak ditemukan' });
}


if (user.whitelistIp.includes('0.0.0.0') && ip !== '0.0.0.0' && user.whitelistIp.length === 1) {
  user.whitelistIp = [];
}

if (user.whitelistIp.includes(ip)) {
  return res.status(400).json({ success: false, message: 'Alamat IP sudah ada di whitelist' });
}

user.whitelistIp.push(ip);
await user.save();

return res.status(200).json({
  success: true,
  message: 'Alamat IP berhasil ditambahkan ke whitelist',
  whitelistIp: user.whitelistIp.join(',')
});
} catch (error) {
console.error('Error adding IP to whitelist:', error);
return res.status(500).json({ success: false, message: 'Kesalahan server internal' });
}
});

app.post('/profile/whitelist-ip/remove', requireLogin, async (req, res) => {
const { ip } = req.body;

if (!ip) {
return res.status(400).json({ success: false, message: 'Alamat IP diperlukan' });
}

try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'Pengguna tidak ditemukan' });
}


const index = user.whitelistIp.indexOf(ip);
if (index === -1) {
  return res.status(400).json({ success: false, message: 'Alamat IP tidak ditemukan di whitelist' });
}

user.whitelistIp.splice(index, 1);

if (user.whitelistIp.length === 0) {
  user.whitelistIp.push('0.0.0.0');
}

await user.save();

return res.status(200).json({
  success: true,
  message: 'Alamat IP berhasil dihapus dari whitelist',
  whitelistIp: user.whitelistIp.join(',')
});
} catch (error) {
console.error('Error removing IP from whitelist:', error);
return res.status(500).json({ success: false, message: 'Kesalahan server internal' });
}
});

app.post('/profile/whitelist-ip/set', requireLogin, async (req, res) => {
const { ips } = req.body;

if (!Array.isArray(ips)) {
return res.status(400).json({ success: false, message: 'Input "ips" harus berupa array' });
}

try {
const user = await User.findById(req.session.userId);
if (!user) {
return res.status(404).json({ success: false, message: 'Pengguna tidak ditemukan' });
}


if (ips.length === 0) {
  user.whitelistIp = ['0.0.0.0'];
} else {
  for (const singleIp of ips) {
    if (typeof singleIp !== 'string' || !isValidIp(singleIp)) {
      return res.status(400).json({ success: false, message: `Format alamat IP tidak valid: ${singleIp}` });
    }
  }
  const uniqueIps = [...new Set(ips)];
  user.whitelistIp = uniqueIps;
}

await user.save();

return res.status(200).json({
  success: true,
  message: 'Whitelist IP berhasil diatur ulang',
  whitelistIp: user.whitelistIp.join(',')
});
} catch (error) {
console.error('Error setting IP whitelist:', error);
return res.status(500).json({ success: false, message: 'Kesalahan server internal' });
}
});

app.get("/h2h/profile", validateApiKey, async (req, res) => {
try {
const user = req.user;
if (!user) {
return res.status(404).json({
success: false,
message: "User not found",
});
}


const userData = {
  fullname: user.fullname,
  username: user.username,
  nomor: user.nomor,
  email: user.email,
  profileUrl: user.profileUrl,
  saldo: user.saldo,
  coin: user.coin,
  apiKey: user.apiKey,
  tanggalDaftar: user.tanggalDaftar,
  role: user.role,
  isVerified: user.isVerified,
  lastLogin: user.lastLogin,
  referralCode: user.referralCode,
  history: user.history.slice(-5).reverse(),
};

res.json({
  success: true,
  message: "Profile data retrieved successfully",
  data: userData,
});
} catch (error) {
console.error("Error fetching profile:", error);
res.status(500).json({
success: false,
message: "Internal server error",
});
}
});

app.get("/h2h/mutasi", validateApiKey, async (req, res) => {
try {
const user = req.user;


if (!user) {
  return res.status(404).json({
    success: false,
    message: "User not found",
  });
}

return res.status(200).json({
  success: true,
  history: user.history,
});
} catch (error) {
console.error("Error fetching history:", error);
return res.status(500).json({
success: false,
message: "Internal server error",
});
}
});

app.get('/h2h/categori', validateApiKey, async (req, res) => {
try {
const url = ${BASE_URL}/layanan/price_list;


const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': url,
  },
  body: params
});

const data = await response.json();

if (!data.status) {
  return res.status(500).json({
    success: false,
    message: 'Server maintenance',
    maintenance: true,
    ip_message: data.message.replace(/[^0-9.]+/g, '')
  });
}

const categories = [...new Set(data.data.map(item => item.category))];

res.json({
  success: true,
  data: categories,
  message: 'Data kategori berhasil didapatkan'
});
} catch (error) {
console.error('Error:', error);
res.status(500).json({
success: false,
message: 'Internal server error'
});
}
});

app.get('/h2h/providers', validateApiKey, async (req, res) => {
try {
const url = ${BASE_URL}/layanan/price_list;


const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': url,
  },
  body: params
});

const data = await response.json();

if (!data.status) {
  return res.status(500).json({
    success: false,
    message: 'Server maintenance',
    maintenance: true,
    ip_message: data.message.replace(/[^0-9.]+/g, '')
  });
}

const providers = [...new Set(data.data.map(item => item.provider))];

res.json({
  success: true,
  data: providers,
  message: 'Data provider berhasil didapatkan'
});
} catch (error) {
console.error('Error:', error);
res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

app.get('/h2h/price-list', validateApiKey, async (req, res) => {
try {
const { category, provider } = req.query;


const url = `${BASE_URL}/layanan/price_list`;

const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    'Accept-Language': 'en-US,en;q=0.9',
    'Referer': url,
  },
  body: params
});

const result = await response.json();

if (!result.status) {
  throw new Error('Server maintenance');
}

let data = result.data || [];

data.sort(regeXcomp);

if (category) {
  data = data.filter(i => 
    i.category && i.category.toLowerCase() === category.toLowerCase()
  );
}

if (provider) {
  data = data.filter(i =>
    i.provider && i.provider.toLowerCase() === provider.toLowerCase()
  );
}

const formattedData = data.map(i => {
  const finalPrice = calculateFinalPrice(i.price);
  return {
    code: i.code,
    name: i.name,
    category: i.category,
    type: i.type,
    provider: i.provider,
    brand_status: i.brand_status,
    status: i.status,
    img_url: customImageUrl,
    final_price: finalPrice,
    price_formatted: `Rp ${toRupiah(finalPrice)}`,
    status_emoji: i.status === "available" ? "✅" : "❎"
  };
});

res.json({
  success: true,
  data: formattedData,
  message: 'Data produk berhasil didapatkan'
});
} catch (error) {
console.error('Error:', error);
res.status(500).json({
success: false,
message: error.message || 'Terjadi kesalahan pada server'
});
}
});

const customImageUrl = "https://i.pinimg.com/236x/f2/7d/e0/f27de0e4a01ba9dfe8607ac03a4f7aae.jpg";

const regeXcomp = (a, b) => {
const aPrice = Number(a.price.replace(/[^0-9.-]+/g, ""));
const bPrice = Number(b.price.replace(/[^0-9.-]+/g, ""));
return aPrice - bPrice;
};

const calculateFinalPrice = (price) => {
const cleanPrice = Number(price.toString().replace(/[^0-9.-]+/g, ""));
const markup = cleanPrice * 0.02; // 2% dari harga
return Math.round(cleanPrice + markup); // dibulatkan biar rapi
};

const toRupiah = (value) => {
return value.toLocaleString('id-ID');
};

const getProductsByProvider = async (provider) => {
const url = ${BASE_URL}/layanan/price_list;

const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("type", "prabayar");

const response = await fetch(url, {
method: 'POST',
headers: {
'Content-Type': 'application/x-www-form-urlencoded',
'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
'Accept-Language': 'en-US,en;q=0.9',
'Referer': url,
},
body: params
});

const data = await response.json();

if (!data.status) {
throw new Error('Server maintenance');
}

data.data.sort(regeXcomp);

return data.data
.filter(i => i.provider === provider)
.map(i => {
const finalPrice = calculateFinalPrice(i.price);
return {
code: i.code,
name: i.name,
category: i.category,
type: i.type,
provider: i.provider,
brand_status: i.brand_status,
status: i.status,
img_url: customImageUrl,
final_price: finalPrice,
price_formatted: Rp ${toRupiah(finalPrice)},
status_emoji: i.status === "available" ? "✅" : "❎"
};
});
};

app.get('/h2h/order/create', validateApiKey, async (req, res) => {
try {
const { code, target } = req.query;
const user = req.user;


if (!code) {
  return res.status(400).json({
    success: false,
    message: 'Code produk harus diisi'
  });
}

if (!target) {
  return res.status(400).json({
    success: false,
    message: 'Target harus diisi'
  });
}

const priceListUrl = `${BASE_URL}/layanan/price_list`;

const priceParams = new URLSearchParams();
priceParams.append("api_key", ATLAN_API_KEY);
priceParams.append("type", "prabayar");

const priceResponse = await axios.post(priceListUrl, priceParams, {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
  }
});

if (!priceResponse.data.status) {
  return res.status(500).json({
    success: false,
    message: 'Server maintenance',
    maintenance: true,
    ip_message: priceResponse.data.message.replace(/[^0-9.]+/g, '')
  });
}

const product = priceResponse.data.data.find(item => item.code === code);
if (!product) {
  return res.status(404).json({
    success: false,
    message: 'Produk tidak ditemukan'
  });
}

const productPrice = parseInt(product.price);
const hargaTotal = Math.round(productPrice * 1.02); // naik 2%

if (user.saldo < hargaTotal) {
  user.history.push({
    aktivitas: 'Order',
    nominal: hargaTotal,
    status: 'Gagal - Saldo tidak mencukupi',
    code: generateReffId(),
    tanggal: new Date()
  });
  await user.save();
  return res.status(400).json({
    success: false,
    message: 'Saldo tidak mencukupi'
  });
}

const reff_id = generateReffId();

const createTransactionUrl = `${BASE_URL}/transaksi/create`;
const transactionParams = new URLSearchParams();
transactionParams.append("api_key", ATLAN_API_KEY);
transactionParams.append("code", code);
transactionParams.append("reff_id", reff_id);
transactionParams.append("target", target);

const transactionResponse = await axios.post(createTransactionUrl, transactionParams, {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
  }
});

const trx = transactionResponse.data?.data;
if (!trx?.id || !trx?.price) {
  user.history.push({
    aktivitas: 'Order',
    nominal: hargaTotal,
    status: 'Gagal - Data transaksi tidak lengkap',
    code: reff_id,
    tanggal: new Date()
  });
  await user.save();
  return res.status(500).json({
    success: false,
    message: 'Gagal membuat transaksi: ID atau harga kosong'
  });
}

res.json({
  success: true,
  message: "Transaksi berhasil dibuat",
  data: {
    ...trx,
    price: hargaTotal
  }
});

const checkStatus = async () => {
  const statusUrl = `${BASE_URL}/transaksi/status`;
  const statusParams = new URLSearchParams();
  statusParams.append("api_key", ATLAN_API_KEY);
  statusParams.append("id", trx.id);
  statusParams.append("type", "prabayar");

  const statusResponse = await axios.post(statusUrl, statusParams, {
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
    }
  });

  const status = statusResponse.data.data?.status;

  if (status === "success") {
    user.saldo -= hargaTotal;
    if (user.role === 'agen') {
      user.coin += Math.floor(hargaTotal * 0.01);
    }
    user.history.push({
      aktivitas: `Order - ${code} - ${target}`,
      nominal: hargaTotal,
      status: 'Sukses',
      code: trx.id,
      tanggal: new Date()
    });
    await user.save();
    console.log(`✅ Transaksi ${trx.id} sukses dan saldo dipotong`);
    return;
  }

  if (status === "failed") {
    user.history.push({
      aktivitas: `Order - ${code} - ${target}`,
      nominal: hargaTotal,
      status: 'Gagal',
      code: trx.id,
      tanggal: new Date()
    });
    await user.save();
    console.log(`❌ Transaksi ${trx.id} gagal`);
    return;
  }

  setTimeout(checkStatus, 5000);
};

checkStatus();
} catch (error) {
console.error('❌ Error:', error.response?.data || error.message);
res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

function generateReffId() {
return 'TRX-' + Math.random().toString(36).substring(2, 10).toUpperCase();
}

app.get('/h2h/order/check', async (req, res) => {
try {
const { trxid, apikey } = req.query;


if (!apikey || !trxid) {
  return res.status(400).json({
    success: false,
    message: 'Parameter "trxid" dan "apikey" harus diisi'
  });
}

const user = await User.findOne({ apiKey: apikey });
if (!user) {
  return res.status(401).json({
    success: false,
    message: 'API Key tidak valid'
  });
}

const statusUrl = `${BASE_URL}/transaksi/status`;
const statusParams = new URLSearchParams();
statusParams.append("api_key", ATLAN_API_KEY);
statusParams.append("id", trxid);
statusParams.append("type", "prabayar");

const response = await axios.post(statusUrl, statusParams, {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36',
  }
});

const status = response.data?.data?.status;

if (!status) {
  return res.status(404).json({
    success: false,
    message: 'Transaksi tidak ditemukan atau gagal mendapatkan status'
  });
}

let statusMessage;
if (status === "success") {
  statusMessage = "Transaksi berhasil";
} else if (status === "pending") {
  statusMessage = "Transaksi sedang diproses";
} else if (status === "failed") {
  statusMessage = "Transaksi gagal";
} else {
  statusMessage = "Status transaksi tidak diketahui";
}

res.json({
  success: true,
  message: statusMessage,
  data: {
    trxid: trxid,
    status: status
  }
});
} catch (error) {
console.error('❌ Error:', error.response?.data || error.message);
res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

async function uploadToCloudMini(buffer, fileName) {
try {
const form = new FormData();
form.append('file', buffer, { filename: fileName, contentType: 'image/png' });
const response = await axios.post('https://files.cloudmini.net/upload', form, {
headers: { ...form.getHeaders() }
});
const { filename, expiry_time } = response.data;
return {
url: https://files.cloudmini.net/download/${filename},
expired: expiry_time
};
} catch (err) {
console.error('🚫 Upload to CloudMini Failed:', err);
throw err;
}
}

async function generateQRAndUpload(data) {
try {
const qrBuffer = await qrcode.toBuffer(data, { type: 'png' });
const filename = v-pedia-${Math.random().toString(36).substring(2, 8)}.png;
const { url } = await uploadToCloudMini(qrBuffer, filename);
return url;
} catch (error) {
console.error('❌ Error saat membuat/upload QR:', error);
throw error;
}
}

app.get('/h2h/deposit/create', validateApiKey, async (req, res) => {
try {
const { nominal } = req.query;
const user = req.user;


  if (!nominal) {
      return res.status(400).json({
          success: false,
          message: 'Parameter nominal wajib diisi'
      });
  }

  const reff_id = generateReffId();
  const url = `${BASE_URL}/deposit/create`;

  const params = new URLSearchParams();
  params.append("api_key", ATLAN_API_KEY);
  params.append("reff_id", reff_id);
  params.append("nominal", nominal);
  params.append("type", "ewallet");
  params.append("metode", "qrisfast");

  const depositResponse = await axios.post(url, params.toString(), {
      headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
          'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
      }
  });

  const depositData = depositResponse.data?.data;
  if (!depositData?.id) {
      return res.status(400).json({
          success: false,
          message: 'Gagal membuat permintaan deposit'
      });
  }


  res.json({
      success: true,
      message: 'Permintaan deposit dibuat',
      data: {
          ...depositData,
          qr_image: depositData.qr_image
      }
  });

  const checkDeposit = async () => {
      const statusUrl = `${BASE_URL}/deposit/status`;
      const statusParams = new URLSearchParams();
      statusParams.append("api_key", ATLAN_API_KEY);
      statusParams.append("id", depositData.id);

      const statusResponse = await axios.post(statusUrl, statusParams.toString(), {
          headers: {
              'Content-Type': 'application/x-www-form-urlencoded',
              'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
          }
      });

      const statusData = statusResponse.data?.data;
      const status = statusData?.status;

      if (status === "success") {
          const saldoMasuk = parseInt(statusData.get_balance || 0);
          user.saldo += saldoMasuk;
          user.history.push({
              aktivitas: 'Deposit',
              nominal: saldoMasuk,
              status: 'Sukses',
              code: depositData.id,
              tanggal: new Date()
          });
          await user.save();
          console.log(`✅ Deposit sukses: ${saldoMasuk} untuk ${user.username}`);
          return;
      }

      if (status === "failed") {
          user.history.push({
              aktivitas: 'Deposit',
              nominal: parseInt(nominal),
              status: 'Gagal',
              code: depositData.id,
              tanggal: new Date()
          });
          await user.save();
          console.log(`❌ Deposit gagal untuk ${user.username}`);
          return;
      }

      setTimeout(checkDeposit, 5000);
  };

  checkDeposit();
} catch (error) {
console.error('❌ Error:', error);
res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

app.get('/h2h/deposit/status', validateApiKey, async (req, res) => {
const { trxid } = req.query;
const user = req.user;

if (!trxid) {
return res.status(400).json({
success: false,
message: 'Parameter trxid wajib diisi'
});
}

try {
const url = ${BASE_URL}/deposit/status;
const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("id", trxid);


const statusResponse = await axios.post(url, params.toString(), {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
  }
});

const statusData = statusResponse.data?.data;
if (!statusData) {
  return res.status(404).json({
    success: false,
    message: 'Data tidak ditemukan dari server eksternal'
  });
}

return res.json({
  success: true,
  message: 'Status deposit ditemukan',
  data: {
    trxid: statusData.reff_id,
    status: statusData.status,
    saldo_masuk: statusData.get_balance || 0
  }
});
} catch (error) {
console.error('❌ Error saat cek status deposit:', error);
res.status(500).json({
success: false,
message: 'Terjadi kesalahan pada server'
});
}
});

app.get('/h2h/deposit/cancel', validateApiKey, async (req, res) => {
const { trxid } = req.query;
const user = req.user;

if (!trxid) {
return res.status(400).json({
success: false,
message: 'Parameter trxid wajib diisi'
});
}

try {
const url = ${BASE_URL}/deposit/cancel;
const params = new URLSearchParams();
params.append("api_key", ATLAN_API_KEY);
params.append("id", trxid);


const cancelResponse = await axios.post(url, params.toString(), {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
  }
});

const cancelData = cancelResponse.data?.data;
if (!cancelData) {
  return res.status(404).json({
    success: false,
    message: 'Data tidak ditemukan dari server eksternal'
  });
}

return res.json({
  success: true,
  message: 'Deposit berhasil dibatalkan',
  data: {
    trxid: cancelData.id,
    status: cancelData.status,
    created_at: cancelData.created_at
  }
});
} catch (error) {
console.error('❌ Error saat membatalkan deposit:', error);


if (error.response && error.response.data) {
  return res.status(error.response.status || 500).json({
    success: false,
    message: error.response.data.message || 'Gagal membatalkan deposit'
  });
}

res.status(500).json({
  success: false,
  message: 'Terjadi kesalahan pada server'
});
}
});

app.post("/api/v1/buy-panel", validateApiKey, async (req, res) => {
const { username, paket } = req.body; // Ambil input dari body request
const user = req.user; // Dapatkan user dari middleware validateApiKey

// Daftar paket yang tersedia
const availablePackets = ["1gb", "2gb", "3gb", "4gb", "5gb", "6gb", "7gb", "8gb", "9gb", "20gb", "unli"];

// Validasi input
if (!username) {
return res.status(400).json({
success: false,
message: "Parameter 'username' harus diisi",
});
}
if (!paket || !availablePackets.includes(paket.toLowerCase())) {
return res.status(400).json({
success: false,
message: Paket tidak valid. Paket yang tersedia: ${availablePackets.join(", ")},
});
}

// Tentukan harga dan spesifikasi berdasarkan paket
let memo, disk, cpu, harga;
switch (paket.toLowerCase()) {
case "1gb":
memo = 1024;
disk = 1024;
cpu = 50;
harga = 1000;
break;
case "2gb":
memo = 2048;
disk = 2048;
cpu = 100;
harga = 2000;
break;
case "3gb":
memo = 3072;
disk = 3072;
cpu = 150;
harga = 3000;
break;
case "4gb":
memo = 4096;
disk = 4096;
cpu = 200;
harga = 4000;
break;
case "5gb":
memo = 5120;
disk = 5120;
cpu = 250;
harga = 5000;
break;
case "6gb":
memo = 6144;
disk = 6144;
cpu = 300;
harga = 6000;
break;
case "7gb":
memo = 7168;
disk = 7168;
cpu = 350;
harga = 7000;
break;
case "8gb":
memo = 8192;
disk = 8192;
cpu = 400;
harga = 8000;
break;
case "9gb":
memo = 9216;
disk = 9216;
cpu = 450;
harga = 9000;
break;
case "10gb":
memo = 10240;
disk = 10240;
cpu = 500;
harga = 10000;
break;
case "unli":
memo = 0;
disk = 0;
cpu = 0;
harga = 15000;
break;
default:
return res.status(400).json({
success: false,
message: "Paket tidak dikenali",
});
}

// Validasi saldo pengguna
if (user.saldo < harga) {
user.history.push({
aktivitas: "Buy Panel",
nominal: harga,
status: "Gagal - Saldo tidak mencukupi",
tanggal: new Date(),
});
await user.save();
return res.status(400).json({
success: false,
message: "Saldo tidak mencukupi",
});
}

try {
// Konfigurasi headers untuk request ke Pterodactyl API
const headers = {
Accept: "application/json",
"Content-Type": "application/json",
Authorization: Bearer ${process.env.PTERO_API_KEY},
};


// Buat user baru di Pterodactyl
const email = `${username}@gmail.com`;
const password = `${username}${disk}`; // Password sederhana berdasarkan username dan disk
const createUserResponse = await axios.post(
  `${process.env.PTERO_DOMAIN}/api/application/users`,
  {
    email,
    username,
    first_name: username,
    last_name: username,
    language: "en",
    password,
  },
  { headers }
);

const newUser = createUserResponse.data.attributes;

// Buat server baru di Pterodactyl
const createServerResponse = await axios.post(
  `${process.env.PTERO_DOMAIN}/api/application/servers`,
  {
    name: username,
    description: "Server dibuat via API Buy Panel",
    user: newUser.id,
    egg: 15, // Egg ID untuk Node.js
    docker_image: "ghcr.io/parkervcp/yolks:nodejs_18",
    startup: "npm start",
    environment: {
      INST: "npm",
      USER_UPLOAD: "0",
      AUTO_UPDATE: "0",
      CMD_RUN: "npm start",
      JS_FILE: "index.js",
    },
    limits: {
      memory: memo,
      swap: 0,
      disk,
      io: 500,
      cpu,
    },
    feature_limits: {
      databases: 5,
      backups: 5,
      allocations: 5,
    },
    deploy: {
      locations: [1], // Lokasi server (ID lokasi)
      dedicated_ip: false,
      port_range: [],
    },
  },
  { headers }
);

const newServer = createServerResponse.data.attributes;

user.saldo -= harga;
user.history.push({
  aktivitas: "Buy Panel",
  nominal: harga,
  status: "Sukses",
  tanggal: new Date(),
});
await user.save();

return res.status(201).json({
  success: true,
  message: "Server berhasil dibuat",
  data: {
    user: {
      id: newUser.id,
      username: newUser.username,
      email,
      password,
    },
    server: {
      id: newServer.id,
      name: newServer.name,
      memory: memo,
      disk,
      cpu,
    },
  },
});
} catch (error) {
console.error("Error creating server:", error.response?.data || error.message);


user.history.push({
  aktivitas: "Buy Panel",
  nominal: harga,
  status: "Gagal - Internal server error",
  tanggal: new Date(),
});
await user.save();

return res.status(500).json({
  success: false,
  message: "Terjadi kesalahan saat membuat server",
  error: error.response?.data || error.message,
});
}
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
console.log(Server running on port ${PORT});
});
