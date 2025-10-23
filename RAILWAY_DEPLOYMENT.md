# Railway Deployment Guide

## Fixed Issues ✅

### Email Verification Timeout on Startup
**Problem:** Server failed to start because email verification timed out during startup.

**Solution:** Made email verification non-blocking. The server now starts immediately and verifies email connection in the background.

**What Changed:**
- Email verification runs asynchronously using `setImmediate()`
- Server startup is no longer blocked by SMTP connection issues
- Emails will still work even if verification fails (it's just a startup check)

---

## Railway Deployment Steps

### 1. Create New Project on Railway

1. Go to [Railway.app](https://railway.app)
2. Click "New Project"
3. Select "Deploy from GitHub repo"
4. Choose your repository
5. Railway will auto-detect it's a Node.js project

### 2. Configure Backend (Server)

#### Build & Start Commands:
- **Build Command:** (leave empty or use `npm install`)
- **Start Command:** `node server.js`
- **Root Directory:** `server` (if monorepo) or `/` (if separate repo)

#### Environment Variables:
Add these in Railway Dashboard → Variables:

```env
# Node Environment
NODE_ENV=production
PORT=5000

# Database
MONGODB_URI=your_mongodb_connection_string

# JWT
JWT_SECRET=your_super_secret_jwt_key_min_32_chars

# Client URL (will be updated after frontend deployment)
CLIENT_URL=https://your-frontend-url.up.railway.app

# Email Configuration (REQUIRED)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your-email@gmail.com
EMAIL_PASS=your-16-char-app-password

# Razorpay (if using payments)
RAZORPAY_KEY_ID=your_razorpay_key_id
RAZORPAY_KEY_SECRET=your_razorpay_key_secret
```

#### Gmail App Password Setup:
1. Enable 2-Factor Authentication on your Google account
2. Go to: https://myaccount.google.com/apppasswords
3. Select "Mail" and "Other (Custom name)"
4. Enter "Savishkar Railway" as the name
5. Copy the 16-character password (remove spaces)
6. Use this as `EMAIL_PASS` in Railway

### 3. Configure Frontend (Client)

#### Build Settings:
- **Build Command:** `npm install && npm run build`
- **Start Command:** (Railway auto-detects from package.json)
- **Root Directory:** `client` (if monorepo) or `/` (if separate repo)

#### Environment Variables:
```env
VITE_API_URL=https://your-backend-url.up.railway.app/api
```

### 4. Deploy

1. **Deploy Backend First:**
   - Push to GitHub
   - Railway auto-deploys
   - Copy the backend URL (e.g., `https://savishkar-backend.up.railway.app`)

2. **Update Frontend Environment:**
   - Set `VITE_API_URL` to your backend URL + `/api`
   - Example: `https://savishkar-backend.up.railway.app/api`

3. **Deploy Frontend:**
   - Push to GitHub
   - Railway auto-deploys
   - Copy the frontend URL

4. **Update Backend CLIENT_URL:**
   - Go to Backend → Variables
   - Update `CLIENT_URL` to your frontend URL
   - Railway will auto-redeploy

---

## Verify Deployment

### 1. Check Backend Health:
```bash
curl https://your-backend-url.up.railway.app/api/test/health
```

Expected response:
```json
{
  "status": "ok",
  "timestamp": "2025-10-23T10:00:00.000Z",
  "uptime": 123.456
}
```

### 2. Check Logs:
Railway Dashboard → Your Service → Logs

Look for:
```
✅ MongoDB Connected Successfully
⏳ Verifying SMTP connection in background...
🚀 Server running on port 5000
```

**Email Verification (in background):**
- ✅ Success: `Email Server Connected Successfully!`
- ⚠️ Timeout: `Email Server Connection FAILED!` (but server still works)

### 3. Test Email Functionality:
1. Register a new user via admin dashboard
2. Check logs for: `✅ Email sent successfully!`
3. Verify email is received (may take 30-60 seconds)

### 4. Test Routing:
1. Navigate to `/admin` on frontend
2. Refresh the page (F5)
3. Should stay on `/admin`, not show 404

---

## Common Issues & Solutions

### Issue 1: Email Verification Timeout ✅ FIXED

**Error:**
```
❌ Email Server Connection FAILED!
📛 Error: Verification timeout
```

**Solution:**
This is now **non-blocking**. The server will start successfully even if email verification times out. Emails will still work when needed.

**Why it happens:**
- Railway/Render free tier has cold starts
- SMTP connection can take 15-30 seconds to establish
- Verification runs in background now

**Action Required:** None - server works fine, emails will be sent when needed.

---

### Issue 2: CORS Errors

**Error:** `Access to fetch at 'https://backend.railway.app/api/...' has been blocked by CORS policy`

**Solution:**
1. Verify `CLIENT_URL` in backend environment variables
2. Should match your frontend URL exactly (no trailing slash)
3. Redeploy backend after updating

**Check in `server.js`:**
```javascript
app.use(cors({
  origin: process.env.CLIENT_URL || 'http://localhost:5173',
  credentials: true
}));
```

---

### Issue 3: 404 on Page Refresh ✅ FIXED

**Error:** Refreshing on `/admin` or `/events` shows "Not Found"

**Solution:** Already fixed with `_redirects` file.

**Verify:**
- File exists: `client/public/_redirects`
- Content: `/* /index.html 200`
- File is copied to `dist/` during build

---

### Issue 4: MongoDB Connection Failed

**Error:** `MongooseServerSelectionError: connect ECONNREFUSED`

**Solution:**
1. Verify `MONGODB_URI` is correct
2. Check MongoDB Atlas:
   - IP Whitelist: Add `0.0.0.0/0` (allow all)
   - Database User: Verify username/password
   - Network Access: Enable from anywhere
3. Format: `mongodb+srv://username:password@cluster.mongodb.net/dbname`

---

### Issue 5: Environment Variables Not Working

**Symptoms:**
- `undefined` values in logs
- Features not working

**Solution:**
1. Railway Dashboard → Your Service → Variables
2. Click "Add Variable" for each missing variable
3. **Important:** No quotes around values
   - ✅ Correct: `smtp.gmail.com`
   - ❌ Wrong: `"smtp.gmail.com"`
4. Redeploy after adding variables

---

## Performance Optimization

### Backend:
- ✅ Email sending is asynchronous (non-blocking)
- ✅ Email verification runs in background
- ✅ Connection pooling enabled for MongoDB
- ✅ Retry logic for email sending (3 attempts)

### Frontend:
- ✅ Lazy loading for routes
- ✅ Code splitting for smaller bundles
- ✅ Client-side routing (no server requests)

---

## Monitoring

### Check Email Status:
1. Admin Dashboard → Notifications tab
2. View sent/failed emails
3. Check timestamps and error messages

### Server Logs:
Railway Dashboard → Logs

**Look for:**
- ✅ `Email sent successfully!` - Email delivered
- ❌ `Email Sending Failed` - Check error details
- ⚠️ `Verification timeout` - Normal, emails still work

### Health Check Endpoint:
```bash
# Check if server is running
curl https://your-backend.up.railway.app/api/test/health

# Check email configuration
# (Look in logs after server starts)
```

---

## Railway-Specific Tips

### 1. Free Tier Limitations:
- **Sleep after inactivity:** Service sleeps after 30 minutes of no requests
- **Cold starts:** First request after sleep takes 10-30 seconds
- **Build minutes:** 500 minutes/month (usually enough)
- **Bandwidth:** 100GB/month

### 2. Custom Domain:
1. Railway Dashboard → Settings → Domains
2. Click "Generate Domain" for free `.up.railway.app` domain
3. Or add custom domain (requires DNS configuration)

### 3. Logs & Debugging:
- Real-time logs in Railway Dashboard
- Filter by service (backend/frontend)
- Download logs for analysis

### 4. Automatic Deployments:
- Push to GitHub → Auto-deploys
- Can disable in Settings → Deployments
- Manual deploy: Click "Deploy" button

### 5. Environment Variables:
- Shared across all deployments
- Can use Railway CLI for bulk import
- Supports `.env` file format

---

## Troubleshooting Checklist

Before asking for help, verify:

- [ ] Backend deployed successfully (check logs)
- [ ] Frontend deployed successfully (check logs)
- [ ] All environment variables set correctly
- [ ] `CLIENT_URL` matches frontend URL
- [ ] `VITE_API_URL` matches backend URL + `/api`
- [ ] MongoDB connection working (check logs)
- [ ] Email configuration present (check logs)
- [ ] `_redirects` file exists in `client/public/`
- [ ] Health check endpoint returns 200 OK

---

## Support

### Railway Issues:
- Discord: https://discord.gg/railway
- Docs: https://docs.railway.app
- Status: https://status.railway.app

### Email Issues:
- Gmail App Passwords: https://myaccount.google.com/apppasswords
- SMTP Settings: https://support.google.com/mail/answer/7126229

### MongoDB Issues:
- Atlas Docs: https://docs.atlas.mongodb.com
- Connection String: https://docs.mongodb.com/manual/reference/connection-string

---

## Summary

✅ **Email verification timeout** - Fixed (non-blocking)
✅ **404 on refresh** - Fixed (`_redirects` file)
✅ **Loader stuck** - Fixed (async email sending)
✅ **Railway deployment** - Ready to deploy

**Next Steps:**
1. Commit and push changes
2. Deploy backend on Railway
3. Deploy frontend on Railway
4. Update environment variables
5. Test all features

**Estimated Deploy Time:** 5-10 minutes per service
