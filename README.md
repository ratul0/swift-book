# Swift & iOS Development Book

A comprehensive guide to Swift programming and iOS development, from fundamentals to production patterns.

## 🚀 Quick Start

### Setting up GitHub Pages

1. **Create a new GitHub repository**
   ```bash
   # Initialize git repo
   git init
   git add .
   git commit -m "Initial commit: Swift book with Hugo"
   ```

2. **Push to GitHub**
   ```bash
   # Replace YOUR-USERNAME and YOUR-REPO-NAME
   git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
   git branch -M main
   git push -u origin main
   ```

3. **Initialize the theme submodule** (IMPORTANT!)
   ```bash
   git submodule add https://github.com/alex-shpak/hugo-book themes/hugo-book
   git add .
   git commit -m "Add Hugo Book theme"
   git push
   ```

4. **Enable GitHub Pages**
   - Go to your repository on GitHub
   - Navigate to Settings > Pages
   - Under "Build and deployment", select "GitHub Actions" as the source
   - The workflow will automatically run on your next push

5. **Update the configuration**
   - Edit `hugo.toml`:
     - Replace `YOUR-USERNAME` with your GitHub username
     - Replace `swift-book` with your repository name if different

## 📝 Local Development

To test the site locally:

1. **Install Hugo** (extended version)
   ```bash
   # macOS
   brew install hugo
   
   # or download from https://gohugo.io/installation/
   ```

2. **Clone with submodules**
   ```bash
   git clone --recurse-submodules https://github.com/YOUR-USERNAME/YOUR-REPO-NAME.git
   cd YOUR-REPO-NAME
   ```

3. **Run local server**
   ```bash
   hugo server -D
   ```
   Visit http://localhost:1313

## 📁 Project Structure

```
.
├── content/          # Book content
│   ├── _index.md    # Homepage
│   └── docs/        # Chapters
├── static/          # Static assets
├── themes/          # Hugo Book theme (submodule)
├── hugo.toml        # Hugo configuration
└── .github/         # GitHub Actions workflow
```

## ✨ Features

- **Hugo Book Theme**: Clean, modern book layout with sidebar navigation
- **GitHub Actions**: Automatic deployment on push
- **Search**: Built-in search functionality
- **Dark Mode**: Auto/Light/Dark theme switcher
- **Mobile Responsive**: Works great on all devices
- **Table of Contents**: Auto-generated for each chapter
- **Edit Links**: Direct links to edit pages on GitHub

## 🔧 Customization

### Adding New Chapters

1. Create a new markdown file in `content/docs/`
2. Add front matter:
   ```yaml
   ---
   title: "Chapter Title"
   weight: 250  # Controls order in sidebar
   ---
   ```

### Updating Theme

```bash
git submodule update --remote --merge
```

## 📄 License

This book content is provided for educational purposes.