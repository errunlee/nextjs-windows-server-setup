
# Deploying a Next.js App on IIS with PM2 and URL Rewrite

Follow these steps to deploy your Next.js application using IIS as a reverse proxy, PM2 for process management, and URL Rewrite.

---

## 1. Build Your Next.js App

On your development machine or build server, run:

```bash
npm run build
```

This will generate the `.next` folder and prepare your app for production.

---

## 2. Copy Build Files

Copy the following files and folders into your IIS vhost directory (e.g., `C:\inetpub\wwwroot\your-site`):

- `.next/`
- `public/`
- `package.json`
- `next.config.ts` or `next.config.js`
- `server.js` (see below)

---

## 3. Install Required Software on Server

- Install **[Node.js](https://nodejs.org/)**.
- Install **[PM2](https://pm2.keymetrics.io/)** globally:

```bash
npm install -g pm2
```

- Install **[URL Rewrite Module](https://www.iis.net/downloads/microsoft/url-rewrite)** for IIS.
- Install **[Application Request Routing (ARR)](https://www.iis.net/downloads/microsoft/application-request-routing)** to enable proxying.
- (Optional) Install **[iisnode](https://github.com/Azure/iisnode)** (not needed if using PM2 only).

---

## 4. Install App Dependencies

Navigate to the copied project folder and run:

```bash
npm install --omit=dev
```

This installs only production dependencies.

---

## 5. Create `server.js` File

Place this `server.js` file in your project root:

```js
const { createServer } = require("http");
const next = require("next");

const app = next({ dev: false });
const handle = app.getRequestHandler();

app.prepare().then(() => {
  createServer((req, res) => {
    handle(req, res);
  }).listen(5173, (err) => {
    if (err) throw err;
    console.log("Server running on http://localhost:5173");
  });
});
```

---

## 6. Start App Using PM2

```bash
pm2 start server.js --name "nextjs-app"
pm2 save
pm2 startup
```

This ensures your app starts automatically on server reboot.

---

## 7. Create `web.config` for IIS Reverse Proxy

Add this `web.config` file in the root of the vhost directory:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="ReverseProxyInboundRule1" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://localhost:5173/{R:1}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

---

## 8. Configure URL Rewrite in IIS Manager (If not using `web.config` directly)

1. Open **IIS Manager**.
2. Select your **website** from the Connections panel.
3. Double-click **URL Rewrite**.
4. Click **Add Rules...** on the right Actions panel.
5. Select **Blank Rule** under **Inbound Rules**, then click **OK**.
6. Configure the rule:
   - **Name:** ReverseProxyInboundRule1
   - **Match URL:**
     - Requested URL: Matches the Pattern
     - Using: Regular Expressions
     - Pattern: `(.*)`
   - **Action:**
     - Action Type: Rewrite
     - Rewrite URL: `http://localhost:5173/{R:1}`
     - Check **Append query string**
     - Leave other options as default.
7. Click **Apply** on the right Actions panel to save.

---

## 9. Setup IIS Site and ARR

- Create a new IIS website pointing to your app folder.
- Setup site bindings (hostnames, ports).
- Ensure **Application Request Routing (ARR)** is installed and enabled.
- In ARR settings, enable **proxy**:
  - Go to **Server Proxy Settings** and check **Enable proxy**.

---

✅ Now, your IIS site will proxy requests to your Next.js app running on port 5173 managed by PM2.
