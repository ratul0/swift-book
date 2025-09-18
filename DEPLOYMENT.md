# Deployment Instructions

## ✅ Setup Checklist

Your Hugo site is configured with:
- ✅ Hugo 0.146.0 (minimum required version)
- ✅ Hugo Book theme configured as git submodule
- ✅ GitHub Actions workflow for automatic deployment
- ✅ All content properly structured in `/content/docs/`
- ✅ Removed unsupported shortcodes (button, columns)

## 🚀 Deploy to GitHub Pages

1. **Initialize Git** (if not already done):
   ```bash
   git init
   ```

2. **Add the Hugo Book theme**:
   ```bash
   git submodule add https://github.com/alex-shpak/hugo-book themes/hugo-book
   ```

3. **Commit all files**:
   ```bash
   git add .
   git commit -m "Initial commit: Swift book with Hugo"
   ```

4. **Create GitHub repository** and push:
   ```bash
   git remote add origin https://github.com/ratul0/swift-book.git
   git branch -M main
   git push -u origin main
   ```

5. **Enable GitHub Pages**:
   - Go to Settings → Pages
   - Under "Build and deployment", select "GitHub Actions"

The site will build automatically and be available at:
https://ratul0.github.io/swift-book/

## 🔧 Troubleshooting

If the build fails:
1. Check Actions tab for error logs
2. Ensure the theme submodule is properly initialized
3. Verify Hugo version in workflow matches theme requirements

## 📝 Local Testing

To test locally:
```bash
# Install Hugo 0.146+ extended version
brew install hugo

# Clone with submodules
git clone --recurse-submodules your-repo-url
cd swift-book

# Run local server
hugo server -D
```

Visit http://localhost:1313 to preview.