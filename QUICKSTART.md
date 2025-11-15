# 🚀 QUICK START GUIDE

## Get Up and Running in 2 Minutes!

### Step 1: Download the Files ✅

You already have them! The gorbagio-viewer folder contains:
- `index.html` - The viewer
- `data.json` - Sample data (REPLACE THIS)
- `README.md` - Full documentation

### Step 2: Add Your Real Data 📦

1. Find your downloaded JSON file: `gorbagio_gorboy_master_XXXXX.json`
2. **REPLACE** the `data.json` file with your real one
3. **RENAME** it to exactly `data.json`

### Step 3: Test Locally 🖥️

**Quick Test (Any OS):**

Just double-click `index.html` - it should open in your browser!

**Better Test (Recommended):**

Open terminal/command prompt in the folder and run:

```bash
python -m http.server 8000
```

Then open: http://localhost:8000

### Step 4: Verify It Works ✅

You should see:
- All 227 gor-boy NFTs
- Filters populated with real traits
- Stats showing: Total: 227, Listed: ~17, Floor Price
- Click any NFT → Opens on Magic Eden

---

## Next Steps 🎯

### Push to GitHub

```bash
cd gorbagio-viewer
git init
git add .
git commit -m "GORBAGIO gor-boy viewer"

# Create repo on GitHub first, then:
git remote add origin https://github.com/YOUR-USERNAME/gorbagio-viewer.git
git push -u origin main
```

### Enable GitHub Pages

1. Go to repo Settings → Pages
2. Source: main branch
3. Save
4. Site live at: `https://YOUR-USERNAME.github.io/gorbagio-viewer/`

### Add to Webflow

Embed code:
```html
<iframe 
  src="https://YOUR-USERNAME.github.io/gorbagio-viewer/" 
  width="100%" 
  height="1200px" 
  frameborder="0">
</iframe>
```

---

## 🐛 Troubleshooting

**Blank page?**
- Make sure `data.json` is in the same folder as `index.html`
- Check browser console (F12) for errors

**Only seeing 2 sample NFTs?**
- You haven't replaced `data.json` with your real file yet!

**Need help?**
- Read the full README.md
- Check the browser console (F12)

---

**That's it! You're done!** 🎉

The viewer is ready to use locally, push to GitHub, and integrate with Webflow.
