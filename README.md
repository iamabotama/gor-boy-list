# GORBAGIO Gor-Boy NFT Viewer

A beautiful, responsive viewer for all 227 GORBAGIO NFTs with the "gor-boy" hands trait.

## 📁 Project Structure

```
gorbagio-viewer/
├── index.html          # Main viewer (this file)
├── data.json          # NFT data (place your generated JSON here)
└── README.md          # This file
```

## 🚀 Local Setup

### Step 1: Prepare Your Files

1. **Get your JSON file:**
   - Take the `gorbagio_gorboy_master_XXXXX.json` file you just downloaded
   - Rename it to `data.json`

2. **Place files together:**
   ```
   your-folder/
   ├── index.html    (download from outputs)
   ├── data.json     (your renamed JSON file)
   ```

### Step 2: Test Locally

**Option A: Using Python (Recommended)**
```bash
# In your project folder, run:
python -m http.server 8000

# Then open: http://localhost:8000
```

**Option B: Using Node.js**
```bash
# Install http-server globally
npm install -g http-server

# Run in your project folder
http-server

# Then open: http://localhost:8080
```

**Option C: Using VS Code**
- Install "Live Server" extension
- Right-click `index.html` → "Open with Live Server"

### Step 3: Verify It Works

You should see:
- ✅ All 227 gor-boy NFTs in a grid
- ✅ Filter dropdowns populated
- ✅ Stats showing correct counts
- ✅ Click any NFT → opens Magic Eden page

---

## 🌐 Deploy to GitHub (for Webflow Integration)

### Step 1: Create GitHub Repository

```bash
# In your project folder
git init
git add .
git commit -m "Initial commit - GORBAGIO Gor-Boy viewer"

# Create a new repo on GitHub, then:
git remote add origin https://github.com/YOUR-USERNAME/gorbagio-viewer.git
git branch -M main
git push -u origin main
```

### Step 2: Enable GitHub Pages

1. Go to your repo on GitHub
2. Settings → Pages
3. Source: Deploy from branch
4. Branch: `main` → `/root`
5. Save

Your site will be live at:
`https://YOUR-USERNAME.github.io/gorbagio-viewer/`

---

## 🔗 Webflow Integration

### Option 1: Embed via iframe (Simplest)

In Webflow, add an **Embed element**:

```html
<iframe 
  src="https://YOUR-USERNAME.github.io/gorbagio-viewer/" 
  width="100%" 
  height="1200px" 
  frameborder="0">
</iframe>
```

### Option 2: Direct Integration (Advanced)

In Webflow, add an **Embed element** with:

```html
<div id="gorbagio-viewer"></div>

<script>
  // Fetch the HTML from GitHub
  fetch('https://YOUR-USERNAME.github.io/gorbagio-viewer/')
    .then(response => response.text())
    .then(html => {
      document.getElementById('gorbagio-viewer').innerHTML = html;
    });
</script>
```

### Option 3: Custom Code (Most Control)

1. Copy the CSS from `<style>` section → Webflow Custom Code (Head)
2. Copy the HTML structure → Webflow page
3. Copy the JavaScript from `<script>` section → Webflow Custom Code (Footer)
4. Update the `fetch('data.json')` line to:
   ```javascript
   fetch('https://raw.githubusercontent.com/YOUR-USERNAME/gorbagio-viewer/main/data.json')
   ```

---

## 🎨 Features

- ✅ **Responsive Grid** - Works on all devices
- ✅ **Smart Filters** - Search by name, filter by traits
- ✅ **Listing Status** - See which are for sale
- ✅ **Live Stats** - Total, listed, floor price
- ✅ **Click to Buy** - Opens Magic Eden directly
- ✅ **Sort Options** - By number or price

---

## 🔄 Updating Data

To refresh NFT data (owners, prices):

1. Run the builder script again
2. Download new JSON
3. Replace `data.json`
4. If on GitHub: commit and push
   ```bash
   git add data.json
   git commit -m "Update NFT data"
   git push
   ```

The viewer will automatically load the new data!

---

## 💡 Customization

### Change Colors

Edit the CSS in `index.html`:
```css
/* Change primary color (currently green) */
#00ff00  →  #YOUR_COLOR

/* Change background */
background: #0a0a0a  →  background: #YOUR_COLOR
```

### Add More Filters

Add new filter dropdowns in the HTML and corresponding logic in JavaScript.

---

## 🐛 Troubleshooting

**"Failed to load NFT data"**
- Make sure `data.json` is in the same folder as `index.html`
- Check browser console for errors
- Verify JSON file is valid (use jsonlint.com)

**Images not loading**
- Check internet connection
- Some images may be slow from IPFS

**GitHub Pages not updating**
- Wait 2-3 minutes after pushing
- Hard refresh browser (Ctrl+Shift+R)
- Check GitHub Actions for build status

---

## 📝 License

This is a viewer for the GORBAGIO NFT collection. All NFT images and data belong to their respective owners.

---

**Built with ❤️ for the GORBAGIO community** 🗑️
